## Deploy Kafka , the GitOps way

### Setup Flux

Below you'll find the steps to deploy our Kafka cluster (and creation of a topic) by simply pushing changes/yaml to Git. The GitOps tool, in this case [Flux v2](https://fluxcd.io/) observes the Git repo and applies any detected changes to the Kubernetes cluster.
Since Flux creates a Git repo on bootstrapping, you need a _personal access token_ for GitHub (because I am using GitHub, but GitLab is also possible), so that Flux is able to authentiacate and create a repository (check all permissions under _repo_) ==> please find [instructions to create access token here](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token).  
Within the [Flux Docu](https://fluxcd.io/docs) you'll find very good explanations for starting as well as enhanced aspects of using Flux !!

**Goal**
Below steps are demoing a scenario where you can deploy Kafka cluster in two different stages, "testing" and "production". Both stages are separated by putting them into different namespaces. Via _kustomize_ you can configure both stages independant via their corresponding directories, and Flux is observing those directories.  
The base configuration for the Kafka cluster is in folder _strimzi-jaeger-eval/kafka-setup/base_ and as an example for how to set properties per stage, you'll find *-patch.yml files in the production subdir for setting a higher number of brokers and zookeeper servers in production environment.

|cluster stage|namespace|kustomize dir|
|---|---|---|
|testing|testing|strimzi-jaeger-eval/kafka-setup/testing|
|production|kafka-cluster|strimzi-jaeger-eval/kafka-setup/production|

1. export your GitHub data

   ```bash
   export GITHUB_USER=<your Github username>
   export GITHUB_TOKEN=<your GitHub token>
   ```

2. install Flux (on Linux): ```curl -s https://fluxcd.io/install.sh | sudo bash``` (at time of writing this doc, I got v 0.14.1 installed)  
  or grab a binary release from [here](https://github.com/fluxcd/flux2/releases)

3. verify that the K8s cluster and tools are covering Flux prerequisites
  
   ```bash
   flux check --pre 
   ```

   produced the output:

   ```bash
    ??? checking prerequisites
    ??? kubectl 1.20.5 >=1.18.0-0
    ??? Kubernetes 1.19.9-gke.1400 >=1.16.0-0
    ??? prerequisites checks passed
    ```

4. bootstrapping Flux
  
   ```bash
   flux bootstrap github \
    --owner=$GITHUB_USER \
    --repository=flux-kafka-demo \
    --branch=main \
    --path=./my-flux \
    --personal 
   ```

   this creates several lines of output, at the end it should show success, like:

   ```bash
    ...
    ??? waiting for Kustomization "flux-system/flux-system" to be reconciled
    ??? Kustomization reconciled successfully
    ??? confirming components are healthy
    ??? source-controller: deployment ready
    ??? kustomize-controller: deployment ready
    ??? helm-controller: deployment ready
    ??? notification-controller: deployment ready
    ??? all components are healthy
    ```

   From now on the repo _flux-kafka-demo_ is the source of truth for your Flux installation !


### Configure repo observation

Now that we have the base setup Flux done, we can start configuring a repo to be observed by our Flux setup.  
To be able to make config changes on your own, I recommend to fork/clone my repo, then configure it for Flux and apply some config changes to see Flux in action.  
The goal is, to have the yaml manifests within [kafka-setup](./kafka-setup) being observed.


1. fork/clone my repo **gkoenig/strimzi-jaeger-eval** , e.g. ```git clone https://github.com/gkoenig/strimzi-jaeger-eval.git``` into your own GitHub account
2. clone **your** Flux Github repo: ```git clone https://github.com/$GITHUB_USER/flux-kafka-demo.git```
3. create a manifest, pointing to **your** strimzi-jaeger-eval repository (since this is the one we want to be observed ;)
  
    > ensure the export of env variable GITHUB_USER with your GitHub username is in place !

    ```bash
    export GITHUB_USER=\<your-github-username\>
    cd flux-kafka-demo \
    ```

    ```bash
    flux create source git strimzi-jaeger-eval-main \
    --url=https://github.com/$GITHUB_USER/strimzi-jaeger-eval \
    --branch=main \
    --interval=30s \
    --export > ./my-flux/strimzi-jaeger-eval-main-branch-source.yaml
    ```

4. commit and push the -source.yaml

    ```bash
    git add -A && git commit -m "created source link to Git repo"
    git push
    ```

5. now that we have only the info about the source git repo configure, let's actually deploy an "application", which is basically a Strimzi Kafka cluster here. Flux offers different template mechanisms for that (helm, kustomize, plain yaml manifests,..). We'll use _kustomize_. The corresponding kustomize yaml is located in folder _kafka-setup_ under the referenced Git repository (the strimzi-jaeger-eval one). Let's create the flux-kustomization for both our environments, _production_ and _testing_ .  
We will do it one by one, starting with creating the Kafka cluster in namespace _testing_ , waiting until all resources are up, then do the same for namespace _kafka-cluster_. Reason for that is, that I ran into weird errors (maybe race conditions) by creating both clusters in parallel,  means by pushing both kustomizations together to let kafka-setup/production and kafka-setup/testing being creatd in parallel.
  
    ```bash
    flux create kustomization strimzi-jaeger-eval-testing-kustomization \
    --source=strimzi-jaeger-eval-main \
    --path="./kafka-setup/testing" \
    --prune=true \
    --validation=client \
    --interval=5m \
    --export > ./my-flux/strimzi-jaeger-eval-testing-kustomization.yaml
    ```

    Your directory layout should like the following:

    ```bash
    .
    ????????? my-flux
        ????????? flux-system
        ??????? ????????? gotk-components.yaml
        ??????? ????????? gotk-sync.yaml
        ??????? ????????? kustomization.yaml
        ????????? strimzi-jaeger-eval-testing-kustomization.yaml
        ????????? strimzi-jaeger-eval-main-branch-source.yaml
    ```

    ```bash
    git add -A && git commit -m "created Kustomization for kafka-cluster in testing namespace"
    git push
    ```

    - Flux reconsiliation (==checking that target state is equal to Git)

    ```bash
    watch flux get kustomizations
    ```
  
    until you see something like:
  
    ```bash
    NAME                                       READY   MESSAGE                                                         REVISION
    flux-system                                True    Applied revision: main/80655111cfe968f1bf37c1e9a8e639af7c1fb2eb main/80655111cfe968f1bf37c1e9a8e639af7c1fb2eb
    strimzi-jaeger-eval-testing-kustomization  True    Applied revision: main/7b83b08a58ec359accd9001ea66d28f112f52a5c main/7b83b08a58ec359accd9001ea66d28f112f52a5c
    ```

    ! of course, the commit hash will be different !

   - Kafka and Zookeeper resources

    wait a couple of minutes....and amazingly, you'll see that they got created, e.g. our "testing" cluster in namespace "testing":

    ```bash
    kubectl get po,service -n testing
    ```

    ```bash
    NAME                                                           READY   STATUS    RESTARTS   AGE
    pod/testing-strimzi-cluster-entity-operator-5db67f65fc-8pmjl   3/3     Running   0          14m
    pod/testing-strimzi-cluster-kafka-0                            1/1     Running   0          15m
    pod/testing-strimzi-cluster-kafka-1                            1/1     Running   0          15m
    pod/testing-strimzi-cluster-zookeeper-0                        1/1     Running   0          15m

    NAME                                                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    service/testing-strimzi-cluster-kafka-0                    NodePort    10.3.253.25    <none>        9094:31681/TCP               15m
    service/testing-strimzi-cluster-kafka-1                    NodePort    10.3.250.98    <none>        9094:31002/TCP               15m
    service/testing-strimzi-cluster-kafka-bootstrap            ClusterIP   10.3.252.94    <none>        9091/TCP,9092/TCP,9093/TCP   15m
    service/testing-strimzi-cluster-kafka-brokers              ClusterIP   None           <none>        9091/TCP,9092/TCP,9093/TCP   15m
    service/testing-strimzi-cluster-kafka-external-bootstrap   NodePort    10.3.243.213   <none>        9094:30321/TCP               15m
    service/testing-strimzi-cluster-zookeeper-client           ClusterIP   10.3.249.78    <none>        2181/TCP                     15m
    service/testing-strimzi-cluster-zookeeper-nodes            ClusterIP   None           <none>        2181/TCP,2888/TCP,3888/TCP   15m
    ```

6. now let the production kafka cluster be created

    ```bash
    # create flux kustomization
    flux create kustomization strimzi-jaeger-eval-prod-kustomization \
    --source=strimzi-jaeger-eval-main \
    --path="./kafka-setup/production" \
    --prune=true \
    --validation=client \
    --interval=5m \
    --export > ./my-flux/strimzi-jaeger-eval-prod-kustomization.yaml

    # push it to flux repo
    git add -A && git commit -m "created Kustomization for kafka-cluster in kafka-cluster namespace"
    git push
    
    # wait some minutes until all pods are up and running
    kubectl get po,svc -n kafka-cluster

    NAME                                                       READY   STATUS    RESTARTS   AGE
    pod/prod-strimzi-cluster-entity-operator-cbf979547-28gqx   3/3     Running   0          13m
    pod/prod-strimzi-cluster-kafka-0                           1/1     Running   0          14m
    pod/prod-strimzi-cluster-kafka-1                           1/1     Running   0          14m
    pod/prod-strimzi-cluster-kafka-2                           1/1     Running   0          14m
    pod/prod-strimzi-cluster-zookeeper-0                       1/1     Running   0          14m

    NAME                                                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    service/prod-strimzi-cluster-kafka-0                    NodePort    10.3.243.78    <none>        9094:31145/TCP               14m
    service/prod-strimzi-cluster-kafka-1                    NodePort    10.3.248.151   <none>        9094:30991/TCP               14m
    service/prod-strimzi-cluster-kafka-2                    NodePort    10.3.254.35    <none>        9094:31685/TCP               14m
    service/prod-strimzi-cluster-kafka-bootstrap            ClusterIP   10.3.252.128   <none>        9091/TCP,9092/TCP,9093/TCP   14m
    service/prod-strimzi-cluster-kafka-brokers              ClusterIP   None           <none>        9091/TCP,9092/TCP,9093/TCP   14m
    service/prod-strimzi-cluster-kafka-external-bootstrap   NodePort    10.3.243.118   <none>        9094:32405/TCP               14m
    service/prod-strimzi-cluster-zookeeper-client           ClusterIP   10.3.249.122   <none>        2181/TCP                     14m
    service/prod-strimzi-cluster-zookeeper-nodes            ClusterIP   None           <none>        2181/TCP,2888/TCP,3888/TCP   14m
    ```

7.  Let's create a topic

    The same mechanism as before applies also to the topic creation. In the Git repo there are dedicated directories for _production_ and _testing_ , because topic configurations will differ between environments and you'll find the according .yaml files in the corresponding subdirectories per environment. Let's also create flux-kustomizations to observe changes to those topic configuration files.

    
    ```bash
    flux create kustomization kafkatopic-testing-kustomization \
    --source=strimzi-jaeger-eval-main \
    --path="./kafka-topics/testing" \
    --prune=true \
    --validation=client \
    --depends-on 'strimzi-jaeger-eval-testing-kustomization' \
    --interval=5m \
    --export > ./my-flux/kafkatopic-testing-kustomization.yaml
    
    git add -A && git commit -m "created Kustomization for kafka-topics in namespace testing"
    git push
    ```
    
    **now let's list the topics, and grep for our custom topic starting with 'my-'**

    ```bash
    #first by listing the Strimzi resource 'kafkatopic'
    kubectl get kafkatopics -n testing | grep 'my-'
    
    #output
    my-first-topic                   testing-strimzi-cluster   2            1                    True
    ```

    ```bash
    #second, by calling the Kafka API directly, means by executing kafka cli tool
    kubectl run kafka-producer -ti \
    --image=strimzi/kafka:0.20.0-rc1-kafka-2.6.0 \
    --rm=true \
    --restart=Never \
    -- bin/kafka-topics.sh --bootstrap-server testing-strimzi-cluster-kafka-bootstrap.testing:9092 --list | grep "my-"
    
    #provides the (expected) output of exactly one topic, "my-first-topic":

    If you don't see a command prompt, try pressing enter.
    
    my-first-topic
    ```

    **Now let's do the same for namespace _kafka-cluster_ , our "production"**

    ```bash
    flux create kustomization kafkatopic-prod-kustomization \
    --source=strimzi-jaeger-eval-main \
    --path="./kafka-topics/production" \
    --prune=true \
    --validation=client \
    --interval=5m \
    --depends-on 'strimzi-jaeger-eval-prod-kustomization' \
    --export > ./my-flux/kafkatopic-prod-kustomization.yaml

    git add -A && git commit -m "created Kustomization for kafka-topics in namespace kafka-cluster"
    git push
    
    ```
  

