# 7shiftsexercise

This repo was created to solve the take home exercise from 7shifts. The first section answers the questions, and the second give more details on how to create an environment with the manifests files provided.

# Exercise

1. **We want to deploy two containers that scale independently from one another**
* Container 1: This container runs code that runs a small API that returns users
from a database
* Container 2: This container runs code that runs a small API that returns shifts
from a database.

Answer: the pods are provided for testing, following this manual.

2. **For the best user experience auto scale this service when average CPU reaches 70%**

Answer: The autoscaling is configured by [hpa.yaml](manifests/hpa.yaml) file. It was not tested since it is a simple pod with nginx frontpage. You can set in this file the minimum and maximum quantity of replicas, and other parameters can be set up as described in [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). The [Autoscaling](#autoscaling) section shows an example.

3. Ensure the deployment can handle rolling deployments and rollbacks

Answer: You can update a version using rolling update, and rollback the deployment. More details on how to do this is given in [Updating apps](#updating-apps) section. This could be done with the commands provided, or configured in a script or jenkins job, so that others could simply run the script.

4. **Your development team should not be able to run certain commands on your k8s cluster, but you want them to be able to deploy and roll back. What types of IAM controls do you put in place?**

Answer: It is possible to set up a script or jenkins jobs with specific permission for each logged user. If the cluster is accessible, it is possible to create a user with access to an specific namespace, that could be an specific environment, and specify commands that could be runned. For example, to update a version, a role with update permissions to deployments should be created and associated to a AWS user. The [Cluster Authentication](https://docs.aws.amazon.com/eks/latest/userguide/managing-auth.html) manual in AWS provides more details.

5. Bonus

**How would you apply the configs to multiple environments (staging vs production)?**

Answer: It could be done something to create the yamls based on a template. I can think about 3 possible solutions:
* Using kustomize: with kustomize, you can specify which fields from the yamls templates would be replaced in a specific environment. This is a native kubernetes solution. More on [kustomize.io](https://kustomize.io/)
* Using shell/python scripting, with yq module installed, used for editing yaml files. It could be created an .env file with variables, and after applying the script, all yaml files would be created based on the templates.
* Helm Charts: a more complex solution. Need a values.yaml file to specify variables, and a chart repo. More on [heml.sh](https://helm.sh/docs/topics/charts/)

**How would you auto scale the deployment based on network latency instead of CPU?**
 Answer: It could be possible configuring a custom metric. This was not made during this exercise, but [this page](https://sysdig.com/blog/kubernetes-autoscaler/) gives an example on how to create a custom type in autoscale, using a custom metric. In this example:

 ```
 metrics:
 - type: Object
   object:
     target:
       kind: Service
       name: deployment;kuard
     metricName: net.http.request.count
     targetValue: 100
 ```


# Environment

This deployments were tested locally using [minikube](https://minikube.sigs.k8s.io/docs/start/) and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/). It is necessary to install both to follow this manual. If you already have access to a kubernetes cluster, whereas in cloud or on premise, you can create the environment there the same way.

## Containers

The 2 containers were build using the dockerfiles provided in the docker directory. Example:

```sh
cd docker
docker build -t 7shifts/container1:v1.0 container1/
docker build -t 7shifts/container2:v1.0 container2/
docker tag 7shifts/container1:v1.0 leafarlins/7shifts:c1-v1.0
docker tag 7shifts/container2:v1.0 leafarlins/7shifts:c2-v1.0
docker push leafarlins/7shifts:c1-v1.0
docker push leafarlins/7shifts:c2-v1.0
```

The images were pushed to the leafarlins/7shifts project on github. To keep it simple, the same repository received the 2 containers. They were marked with tag c1-version to container1, and c2-version to container2. For each one, versions v1.0, v1.1 and v1.2 were generated.

To run this environments, it is not necessary to build the images, once they're already on dockerhub.

There is a problem in the v1.2, what will make it be an unsuccessful update.

## Minikube

If you want to test locally, configure minikube with your favorite driver, memory size of the virtual machine and ingress enabled.

```
minikube config set driver virtualbox
minikube config set memory 4096
minikube start
minikube addons enable ingress
```

The next steps will create the environment for the 7shifts pods.

## Creating environment

Create a namespace and the pods applying the yaml config files in manifests.

```
cd manifests/
kubectl apply -f namespace.yaml
kubectl apply -f container1.yaml
```

Check if the pods are already running

```
kubectl -n 7shifts get pods
```

Check the IP address gave by ingress. Example:

```console
$ kubectl -n 7shifts get ingress
NAME                 CLASS   HOSTS                ADDRESS          PORTS   AGE
container1-ingress   nginx   container1.7shifts   192.168.99.100   80      9m40s
container2-ingress   nginx   container2.7shifts   192.168.99.100   80      42s
```

Make the address respond to the IP shown by the last command. It can be easily made in linux editing /etc/hosts file:

```
192.168.99.100 container1.7shifts
192.168.99.100 container2.7shifts
```

The two containers can be accessed in http://container1.7shifts/ and http://container2.7shifts/

The version of each image in deployment:

```console
$ kubectl -n 7shifts get deployments -o wide
NAME                READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                       SELECTOR
container1-deploy   1/1     1            1           33m   container1   leafarlins/7shifts:c1-v1.0   k8s-app=container1
container2-deploy   1/1     1            1           13m   container2   leafarlins/7shifts:c2-v1.0   k8s-app=container2
```

## Autoscaling

The autoscaling is configured by hpa.yaml file. You need a metrics-server installed in the cluster (or addon in minikube). After applying the manifest, check each pod utilization with the command.

```console
$ kubectl -n 7shifts get hpa
NAME                REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
container1-scaler   Deployment/container1-deploy   5%/70%    1         6         1          6m21s
container2-scaler   Deployment/container2-deploy   5%/70%    1         6         1          20s
```

## Updating apps

An update can be made using rolling update with the command set image. After that, we can make a rollout status to check if everything was successful.

```console
$ kubectl -n 7shifts set image deployment/container1-deploy container1=leafarlins/7shifts:c1-v1.1
deployment.apps/container1-deploy image updated
$ kubectl -n 7shifts set image deployment/container2-deploy container2=leafarlins/7shifts:c2-v1.1
deployment.apps/container2-deploy image updated
$ kubectl -n 7shifts rollout status deployment/container1-deploy
deployment "container1-deploy" successfully rolled out
$ [ "$?" -ne 0 ] && echo "ERROR" || echo "OK"
OK
$ kubectl -n 7shifts rollout status deployment/container2-deploy
deployment "container2-deploy" successfully rolled out
$ [ "$?" -ne 0 ] && echo "ERROR" || echo "OK"
OK
```

Note that with version v1.2 the pod was not successfully updated. With an ERROR exit, we could make a rollout undo to revert the version.

```console
$ kubectl -n 7shifts set image deployment/container1-deploy container1=leafarlins/7shifts:c1-v1.2
deployment.apps/container1-deploy image updated
$ kubectl -n 7shifts rollout status deployment/container1-deploy --timeout=30s
Waiting for deployment "container1-deploy" rollout to finish: 1 old replicas are pending termination...
error: timed out waiting for the condition
$ [ "$?" -ne 0 ] && echo "ERROR" || echo "OK"
ERROR
$ kubectl -n 7shifts rollout undo deployment/container1-deploy
deployment.apps/container1-deploy rolled back
```

Note that for all of this work properly, it is important to define startupProbes for every pod.
