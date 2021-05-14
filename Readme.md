## base GCP setup
- create new project within GCP console, name: strimzi-and-jaeger-eval
- create new configuration for gcloud cli
  ```gcloud config configuration create strimzi-jaeger```
- set project
  ```gcloud config set project strimzi-and-jaeger-eval
- authenticate with your Google account
  ```gcloud auth login```

## create GKE cluster

For this playground let's create a single-zone cluster, consisting of 3 nodes

```bash
gcloud container clusters create strimzi-jaeger-eval \
  --num-nodes=3 \
  --zone=europe-central2-a \
  --labels=requestor=gerd-koenig,customer=scigility,project=evaluation
```

## deploy Strimzi operator

We'll use plain Strimzi yaml files, to avoid installing Helm additionally.
Alternative would be to install Strimzi operator via Helm.
The operator will be installed into namespace _kafka-op_ and the Kafka cluster itself into namespace _kafka-cluster_.

### preparations
```bash
kubectl create ns kafka-op
kubectl create ns kafka-cluster
wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.22.1/strimzi-0.22.1.tar.gz
tar -xvzf strimzi-0.22.1.tar.gz
cd strimzi-0.22.1
```

### Set namespace for the operator

```bash
sed -i 's/namespace: .*/namespace: kafka-op/' install/cluster-operator/*RoleBinding*.yaml
```

### Set namespace & additional rolebindings for Kafka cluster

Open file 060-Deployment-strimzi-cluster-operator.yaml (under install/cluster-operator folder) and locate the STRIMZI_NAMESPACE environment variable to specify our namespace, as shown below:

```bash
          env:
            - name: STRIMZI_NAMESPACE
              value: kafka-cluster
```

Create rolebindings for our desired namespace:

```bash
kubectl apply -f install/cluster-operator/020-RoleBinding-strimzi-cluster-operator.yaml -n kafka-cluster
kubectl apply -f install/cluster-operator/031-RoleBinding-strimzi-cluster-operator-entity-operator-delegation.yaml -n kafka-cluster
kubectl apply -f install/cluster-operator/032-RoleBinding-strimzi-cluster-operator-topic-operator-delegation.yaml -n kafka-cluster
```

### finally deploy the Strimzi operator

```bash
kubectl apply -f install/cluster-operator -n kafka-op
kubectl create -f install/cluster-operator/040-Crd-kafka.yaml # because this file is too large for kubectl apply ...
```

...and verify it: ```kubectl get deployments -n kafka-op```

## Deploy Kafka cluster

You'll find sample .yaml files in examples/kafka folder. Based on file kafka-persistent.yaml we are creating a specification for our test cluster, as shown below:

```
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: strimzi-cluster
spec:
  kafka:
    version: 2.7.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
      - name: external
        port: 9094
        type: nodeport
        tls: false
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      log.message.format.version: "2.7"
      inter.broker.protocol.version: "2.7"
    storage:
      type: persistent-claim
      size: 5Gi
      deleteClaim: false
    logging:
      type: inline
      loggers:
        kafka.root.logger.level: "INFO"    
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 5Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}

```
This will setup a 3-node Kafka cluster with a single-node Zookeeper. Kafka will be reachable via port 9092 without authentication and encryption, as well as on port 9093 tls encrypted. Additionally we have an external facing listener on port 9094, which is created via a NodePort service. 

Save above file as kafka-deployment.yaml and create the cluster in our desired namespace:  
```bash
kubectl apply -f kafka-deployment.yaml -n kafka-cluster
```

Check: ```kubectl get all -n kafka-cluster```

## Creating a topic

Now that Kafka is up and running, let's create a topic by using the TopicOperator

```bash
cat <<EOF | kubectl apply -n kafka-cluster -f -
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: my-first-topic
  labels:
    strimzi.io/cluster: strimzi-cluster
spec:
  partitions: 3
  replicas: 2
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824
EOF
```

List the available topics:

```bash
kubectl run kafka-producer -ti \
    --image=strimzi/kafka:0.20.0-rc1-kafka-2.6.0 \
    --rm=true \
    --restart=Never \
    -- bin/kafka-topics.sh --bootstrap-server strimzi-cluster-kafka-bootstrap.kafka-cluster:9092 --list
```

## Producing/Consuming messages

Start a Producer:

```bash
kubectl run kafka-producer -ti \
  --image=strimzi/kafka:latest-kafka-2.4.0 \
  --rm=true --restart=Never \
  -- bin/kafka-console-producer.sh --broker-list strimzi-cluster-kafka-bootstrap.kafka-cluster:9092 --topic my-topic
```

and in another terminal, start a Consumer:

```bash
kubectl run kafka-consumer -ti \
  --image=strimzi/kafka:latest-kafka-2.4.0 \
  --rm=true --restart=Never \
  -- bin/kafka-console-consumer.sh --bootstrap-server strimzi-cluster-kafka-bootstrap.kafka-cluster:9092 --topic my-first-topic --from-beginning
```

## Jager

### Deploying Jager core components

Install Operator Lifecycle Manager (OLM), a tool to help manage the Operators running on your cluster.
```
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.18.1/install.sh | bash -s v0.18.1
```

Install the operator by running the following command:

```
kubectl create -f https://operatorhub.io/install/jaeger.yaml
```

This Operator will be installed in the "operators" namespace and will be usable from all namespaces in the cluster.
After install, watch your operator come up using next command: ```kubectl get csv -n operators```
To use it, checkout the custom resource definitions (CRDs) introduced by this operator to start using it.

Now that the Jaeger Operator is in place, we can create our Jaeger CRD. In this demo we are using the _All-in-one_ bundle of Jaeger, which includes all components within one binary. It will be installed in the _default_ namespace:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: my-jaeger
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: debug
  storage:
    type: memory
    options:
      memory:
        max-traces: 100000
  ingress:
    enabled: false
  agent:
    strategy: DaemonSet
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
EOF
```

### Accessing the Jaeger UI

Port forwarding from your local box to Jaeger-UI:```kubectl port-forward service/my-jaeger-query 8080:16686```
Now open a browser and navigate to ```http://localhost:8080```

## Tracing in Action

Let's deploy a sample application, consisting of a producer, consumer and a Kafka-streams application in between.

- create two topics: ```kubectl apply -f ./jaeger-example/1-topics.yaml```
- create Producer: ```kubectl apply -f ./jaeger-example/2-producer.yaml```
- create Streams app: ```kubectl apply -f ./jaeger-example/3-kafka-streams.yaml```
- create Consumer: ```kubectl apply -f ./jaeger-example/4-consumer.yaml```

