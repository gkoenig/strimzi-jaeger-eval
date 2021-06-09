## Deploy Kafka _manually_

Below you'll find the steps to deploy our Kafka cluster (and creation of a topic) manually, means no automated mechanism as e.g. GitOps paradigm.
The base layour of the Git repo directory structure under folder _kafka-setup_ is mainly targeting the GitOps approach, and the scenario to configure two different clusters in two different namespaces and to configure them individually via _kustomize_.  
For the manual creation of those kafka clusters, you can apply the manifest [kafka-deployment.yaml](kafka-setup/base/kafka-deployment.yaml) to your desired namespace:  

**Testing**

```bash
kubectl apply -f ./kafka-setup/base/kafka-deployment.yaml -n testing
```

**Production**  
Potentially you want to edit the kafka-deployment.yaml first and increase e.g. the number of Kafka brokers and/or Zookeeper nodes (check for **spec/replicas**)
```bash
kubectl apply -f ./kafka-setup/base/kafka-deployment.yaml -n kafka-cluster
```


Check: ```kubectl get all -n kafka-cluster```

**Creating a topic**

Now that Kafka is up and running, let's create a topic within production cluster in namespace _kafka-cluster_ , by using the TopicOperator.  
The base topic configuration is specified in ```./kafka-topics/base/topics.yaml```. You can adjust base properties there, or overwrite settings for your desired environment in the corresponding folder ```./kafka-topics/testing``` / ```./kafka-topics/production``` in file ```kafkatopic-config-patch.yaml```.

```bash
kubectl apply -k ./kafka-topics/production
```

....and , of course, we can do the same for the _testing_ cluster:

```bash
kubectl apply -k ./kafka-topics/testing
```
