## Deploy Kafka _manually_

Below you'll find the steps to deploy our Kafka cluster (and creation of a topic) manually, means no automated mechanism as e.g. GitOps paradigm.

Simply apply the manifest [kafka-deployment.yaml](kafka-setup/kafka-deployment.yaml) to our desired namespace:  

```bash
kubectl apply -f ./kafka-setup/kafka-deployment.yaml -n kafka-cluster
```

Check: ```kubectl get all -n kafka-cluster```

**Creating a topic**

Now that Kafka is up and running, let's create a topic by using the TopicOperator

```bash
kubectl apply -n kafka-cluster -f ./kafka-setup/topics.yaml
```