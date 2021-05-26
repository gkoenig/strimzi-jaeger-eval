# Strimzi Kafka & Jaeger tracing first steps

Below you'll find instructions to start playing around with a Kafka cluster deployed on Kubernets, by using _Strimzi_ operator.  
On top of that we'll use Jaeger to create&display tracing info. Those tracing information is created with producers, consumers and streams applications.  
As underlying Kubernetes cluster, we'll use GKE, but you can use whichever K8s flavour you prefer.  

I will also show you how to apply **GitOps** approach of bringing applications/configuration to your K8s cluster by using [Flux v2](https://fluxcd.io/) to setup Kafka and Topics. You will find the step-by-step instructions in the section about [deploying Kafka cluster](#deploy-kafka-cluster)

- [Strimzi Kafka & Jaeger tracing first steps](#strimzi-kafka--jaeger-tracing-first-steps)
  - [base GCP setup](#base-gcp-setup)
  - [create GKE cluster](#create-gke-cluster)
  - [deploy Strimzi operator](#deploy-strimzi-operator)
    - [preparations](#preparations)
    - [Set namespace for the operator](#set-namespace-for-the-operator)
    - [Set namespace & additional rolebindings for Kafka cluster](#set-namespace--additional-rolebindings-for-kafka-cluster)
    - [finally deploy the Strimzi operator](#finally-deploy-the-strimzi-operator)
  - [Deploy Kafka cluster](#deploy-kafka-cluster)
  - [Producing/Consuming messages](#producingconsuming-messages)
  - [Jager](#jager)
    - [Deploying Jager core components](#deploying-jager-core-components)
    - [Accessing the Jaeger UI](#accessing-the-jaeger-ui)
  - [First tracing in action](#first-tracing-in-action)
  - [Kafka Connect](#kafka-connect)
    - [Deploy&prepare Postgresql](#deployprepare-postgresql)
      - [Postgresql deployment](#postgresql-deployment)
      - [Create db and table](#create-db-and-table)
    - [Deploy KafkaConnect](#deploy-kafkaconnect)
    - [Deploy Connector](#deploy-connector)

## base GCP setup

- create new project within GCP console, name: strimzi-and-jaeger-eval
- create new configuration for gcloud cli: ```gcloud config configuration create strimzi-jaeger```
- set project: ```gcloud config set project strimzi-and-jaeger-eval```
- authenticate with your Google account: ```gcloud auth login```

## create GKE cluster

For this playground let's create a single-zone cluster, consisting of 2 nodes. Please adjust the _machine-type_ according to your needs, the "e2-standard-2" ships with 2CPUs and 8GB Ram.

```bash
gcloud container clusters create strimzi-jaeger-eval \
  --num-nodes=2 \
  --zone=europe-central2-a \
  --machine-type=e2-standard-2
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
until you see the deployment up and running, like e.g.:

```bash
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
strimzi-cluster-operator   1/1     1            1           51s
```

## Deploy Kafka cluster

Next, let's setup a 3-node Kafka cluster with a single-node Zookeeper. Kafka will be reachable via port 9092 without authentication and encryption, as well as on port 9093 tls encrypted. Additionally we have an external facing listener on port 9094, which is created via a NodePort service. 

To actually deploying the Kafka cluster I'll provide two different approaches. One is applying the plain yaml manifests by executing _kubectl_ commands, and the other approach is to use GitOps-tool Flux, so that Kafka specs are being applied to the K8s cluster as soon as you push them to Git.

- [manual kafka setup](./Kafka-setup-manual.md) by running _kubectl_ commands
- [kafka setup, the GitOps way](./Kafka-setup-GitOps.md) by using Flux

If you are done with setting up Kafka, you can list the available topics:

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

## First tracing in action

Let's deploy a sample application, consisting of a producer, consumer and a Kafka-streams application in between.

- create two topics: ```kubectl apply -f ./jaeger-example/1-topics.yaml```
- create Producer: ```kubectl apply -f ./jaeger-example/2-producer.yaml```
- create Streams app: ```kubectl apply -f ./jaeger-example/3-kafka-streams.yaml```
- create Consumer: ```kubectl apply -f ./jaeger-example/4-consumer.yaml```

## Kafka Connect

Now that the base Kafka cluster and Jaeger is setup, let's enrich the platform by adding Kafka Connect....and a Connector to read data from Postgresql to Kafka.

### Deploy&prepare Postgresql

First we deploy a Postgresql db, which serves as source for our Connector. We'll only run one Postgresql instance, no HA , since the focus of this setup is on _Tracing_ and not on deploying HA RDBMS on K8s.
Since the Kubernetes cluster is setup in GKE, I will use the storageclass _standard_ which is available by default. Please adjust this if you are running Kubernetes on a different provider or locally on your workstation via microk8s , k3s , etc.

#### Postgresql deployment

Inspect the yaml manifest [here](./kafka-connect/postgresql.yaml)  
and apply it via ```kubectl apply -f kafka-connect/postgresql.yaml```

Afterwards check the status of the Postgresql pod via: ```kubectl get pod -l app=postgres```

#### Create db and table

use the following command to enter a prompt within the Postgresql database. The db user + password are hardcoded (check the spec in [postgresql.yaml](kafka-connect/postgresql.yaml), they need to match.). Of course this is not according security best practices, but good enough for a quick demo. A more secure approach would be to e.g. put the credentials into a Kubernetes secret object.

```bash
kubectl run postgres-postgresql-client --rm --tty -i --restart='Never' --image docker.io/bitnami/postgresql:11.9.0-debian-10-r48 --env="PGPASSWORD=dbpasswd" --command -- psql --host postgres -U dbuser -d postgres -p 5432
```

Then create a database, a table and insert some data:

```
postgres=# CREATE DATABASE inventory;
postgres=# \c inventory;
inventory=# CREATE TABLE products (product_no integer, name text, price numeric);
inventory=# INSERT INTO products VALUES (1, 'Cheese', 9.99);
inventory=# INSERT INTO products VALUES (2, 'Beer', 5.99);
inventory=# INSERT INTO products VALUES (3, 'Salmon', 29.99);

```


### Deploy KafkaConnect

We first need to build a container, which includes the Postgresql Debezium driver, so that we actually can point our KafkaConnect connector to the Postgresql database. This step is **optional**, you can also simply go ahead and apply the yaml manifest (which is referring to the container image within my DockerHub account).

- grab the Postgresql driver archive 
```
curl https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/1.5.0.Final/debezium-connector-postgres-1.5.0.Final-plugin.tar.gz | tar xvz
```
- create Dockerfile to build the container
```
cat <<EOF >Dockerfile
FROM docker.io/strimzi/kafka:0.16.1-kafka-2.4.0
USER root:root
RUN mkdir -p /opt/kafka/plugins/debezium
COPY ./debezium-connector-postgres/ /opt/kafka/plugins/debezium/
USER 1001
EOF
```
- build the image and upload it to e.g. DockerHub (you can also use any other container repository, of course).
```
# if not logged in into dockerhub, execute: ```docker login docker.io``` first
export DOCKER_ORG=gkoenig
docker build . -t ${DOCKER_ORG}/connect-debezium
docker push ${DOCKER_ORG}/connect-debezium
```

Inspect the yaml manifest [here](./kafka-connect/kafka-connect.yaml)  
and apply it via ```kubectl apply -f kafka-connect/kafka-connect.yaml```

### Deploy Connector