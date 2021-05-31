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

2. install Flux (on Linux): ```curl -s https://fluxcd.io/install.sh | sudo bash``` (at time of writing this doc, I got v 0.13.4 installed)  
  or grab a binary release from [here](https://github.com/fluxcd/flux2/releases)

3. verify that the K8s cluster and tools are covering Flux prerequisites
  
   ```bash
   flux check --pre 
   ```

   produced the output:

   ```bash
    ► checking prerequisites
    ✔ kubectl 1.20.5 >=1.18.0-0
    ✔ Kubernetes 1.19.9-gke.1400 >=1.16.0-0
    ✔ prerequisites checks passed
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
    ◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
    ✔ Kustomization reconciled successfully
    ► confirming components are healthy
    ✔ source-controller: deployment ready
    ✔ kustomize-controller: deployment ready
    ✔ helm-controller: deployment ready
    ✔ notification-controller: deployment ready
    ✔ all components are healthy
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
    flux create source git strimzi-jaeger-eval-testing \
    --url=https://github.com/$GITHUB_USER/strimzi-jaeger-eval \
    --branch=testing \
    --interval=30s \
    --export > ./my-flux/strimzi-jaeger-eval-testing-branch-source.yaml
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

5. now that we have only the info about the source git repo configure, let's actually deploy an "application". Flux offers different template mechanisms for that (helm, kustomize, plain yaml manifests,..). We'll use _kustomize_. The corresponding kustomize yaml is located in folder _kafka-setup_ under the referenced Git repository (the strimzi-jaeger-eval one)
  
    ```bash
    flux create kustomization strimzi-jaeger-eval-prod-kustomization \
    --source=strimzi-jaeger-eval-testing \
    --path="./kafka-setup/production" \
    --prune=true \
    --validation=client \
    --interval=5m \
    --export > ./my-flux/strimzi-jaeger-eval-prod-kustomization.yaml
    ```
  
    ```bash
    flux create kustomization strimzi-jaeger-eval-testing-kustomization \
    --source=strimzi-jaeger-eval-testing \
    --path="./kafka-setup/testing" \
    --prune=true \
    --validation=client \
    --interval=5m \
    --export > ./my-flux/strimzi-jaeger-eval-testing-kustomization.yaml
    ```

    Your directory layout should like the following:

    ```bash
    .
    └── my-flux
        ├── flux-system
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        ├── strimzi-jaeger-eval-testing-kustomization.yaml
        ├── strimzi-jaeger-eval-prod-kustomization.yaml
        ├── strimzi-jaeger-eval-testing-branch-source.yaml
        └── strimzi-jaeger-eval-source.yaml

    2 directories, 5 files
    ```

6. commit and push the -source.yaml

  ```bash
  git add -A && git commit -m "created Kustomization for kafka-setup"
  git push
  ```

7. now check your cluster for ongoing progress: zookeeper and kafka deployments/service creation

   - Flux reconsiliation (==checking that target state is equal to Git)

    ```bash
    watch flux get kustomizations
    ```
  
    until you see something like:
  
    ```bash
    NAME                                       READY   MESSAGE                                                         REVISION
    flux-system                                True    Applied revision: main/80655111cfe968f1bf37c1e9a8e639af7c1fb2eb main/80655111cfe968f1bf37c1e9a8e639af7c1fb2eb
    strimzi-jaeger-eval-prod-kustomization     True    Applied revision: main/7b83b08a58ec359accd9001ea66d28f112f52a5c main/7b83b08a58ec359accd9001ea66d28f112f52a5c
    strimzi-jaeger-eval-testing-kustomization  True    Applied revision: main/7b83b08a58ec359accd9001ea66d28f112f52a5c main/7b83b08a58ec359accd9001ea66d28f112f52a5c
    ```

    ! of course, the commit hash will be different !

   - Kafka and Zookeeper resources

    ```bash
    kubectl get deployments,service -n kafka-cluster
    ```

    and amazingly, you'll see that they got created

    ```bash
    NAME                                                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    service/prod-strimzi-cluster-kafka-0                    NodePort    10.3.241.135   <none>        9094:31824/TCP               77s
    service/prod-strimzi-cluster-kafka-1                    NodePort    10.3.255.134   <none>        9094:31819/TCP               78s
    service/prod-strimzi-cluster-kafka-2                    NodePort    10.3.240.71    <none>        9094:31594/TCP               77s
    service/prod-strimzi-cluster-kafka-bootstrap            ClusterIP   10.3.250.224   <none>        9091/TCP,9092/TCP,9093/TCP   78s
    service/prod-strimzi-cluster-kafka-brokers              ClusterIP   None           <none>        9091/TCP,9092/TCP,9093/TCP   78s
    service/prod-strimzi-cluster-kafka-external-bootstrap   NodePort    10.3.251.170   <none>        9094:31433/TCP               77s
    service/prod-strimzi-cluster-zookeeper-client           ClusterIP   10.3.243.23    <none>        2181/TCP                     2m19s
    service/prod-strimzi-cluster-zookeeper-nodes            ClusterIP   None           <none>        2181/TCP,2888/TCP,3888/TCP   2m19s
    ```

   - check topics, since we also have a yaml spec defining our topic(s)

    Within file kafka-setup/topics.yaml we specified that we want to have a topic called "my-first-topic". Let's see if it is there:

    ```bash
    kubectl run kafka-producer -ti \
    --image=strimzi/kafka:0.20.0-rc1-kafka-2.6.0 \
    --rm=true \
    --restart=Never \
    -- bin/kafka-topics.sh --bootstrap-server prod-strimzi-cluster-kafka-bootstrap.kafka-cluster:9092 --list | grep "my-"
    ```

    provides the (expected) output of exactly one topic, "my-first-topic":

    ```bash
    If you don't see a command prompt, try pressing enter.
    my-first-topic
    ```

