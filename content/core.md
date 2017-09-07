# Getting started with core concepts
In this section we're going through core concepts of Kubernetes: 

   * [Pods](#core)
   * [Labels](#labels)
   * [Controllers](#controllers)
   * [Deployments](#deployments)
   * [Services](#services)
   * [Volumes](#volumes)
      
## Pods    
In Kubernetes, a group of one or more containers is called a pod. Containers in a pod are deployed together, and are started, stopped, and replicated as a group. The simplest pod definition describes the deployment of a single container. For example, an nginx web server pod might be defined as such

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mynginx
  namespace: default
  labels:
    run: nginx
spec:
  containers:
  - name: mynginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

A pod definition is a declaration of a desired state. Desired state is a very important concept in the Kubernetes model. Many things present a desired state to the system, and it is Kubernetes’ responsibility to make sure that the current state matches the desired state. For example, when you create a Pod, you declare that you want the containers in it to be running. If the containers happen to not be running (e.g. program failure, …), Kubernetes will continue to (re-)create them for you in order to drive them to the desired state. This process continues until the Pod is deleted.

Create a pod containing an nginx server from the pod-nginx.yaml file

    [root@kubem00 ~]# kubectl create -f pod-nginx.yaml

List all pods:

    [root@kubem00 ~]# kubectl get pods -o wide
    NAME      READY     STATUS    RESTARTS   AGE       IP            NODE
    mynginx   1/1       Running   0          8m        172.30.21.2   kuben03

Describe the pod:

    [root@kubem00 ~]# kubectl describe pod mynginx
    Name:           mynginx
    Namespace:      default
    Node:           kuben03/10.10.10.83
    Start Time:     Wed, 05 Apr 2017 11:17:28 +0200
    Labels:         run=nginx
    Status:         Running
    IP:             172.30.21.2
    Controllers:    <none>
    Containers:
      mynginx:
        Container ID:       docker://a35dafd66ac03f28ce4213373eaea56a547288389ea5c901e27df73593aa5949
        Image:              nginx:latest
        Image ID:           docker-pullable://docker.io/nginx
        Port:               80/TCP
        State:              Running
          Started:          Wed, 05 Apr 2017 11:17:37 +0200
        Ready:              True
        Restart Count:      0
        Volume Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-92z22 (ro)
        Environment Variables:      <none>
    Conditions:
      Type          Status
      Initialized   True
      Ready         True
      PodScheduled  True
    Volumes:
      default-token-92z22:
        Type:       Secret (a volume populated by a Secret)
        SecretName: default-token-92z22
    QoS Class:      BestEffort
    Tolerations:    <none>
    Events:
    ...

Delete the pod:

    [root@kubem00 ~]# kubectl delete pod mynginx
    pod "mynginx" deleted

In the example above, we had a pod with a single container nginx running inside.

Kubernetes let's user to have multiple containers running in a pod. All containers inside the same pod share the same resources, e.g. network and volumes and are always scheduled togheter on the same node. The primary reason that Pods can have multiple containers is to support helper applications that assist a primary application. Typical examples of helper applications are data pullers, data pushers, and proxies. Helper and primary applications often need to communicate with each other, typically through a shared filesystem or loopback network interface.

## Labels
In Kubernetes, labels are a system to organize objects into groups. Labels are key-value pairs that are attached to each object. Label selectors can be passed along with a request to the apiserver to retrieve a list of objects which match that label selector.

To add a label to a pod, add a labels section under metadata in the pod definition:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
...
```

To label a running pod

      [root@kubem00 ~]# kubectl label pod mynginx type=webserver
      pod "mynginx" labeled

To list pods based on labels

      [root@kubem00 ~]# kubectl get pods -l type=webserver
      NAME      READY     STATUS    RESTARTS   AGE
      mynginx   1/1       Running   0          21m

Labels can be applied not only to pods but also to other Kuberntes objects like nodes. For example, we want to label or worker nodes based on the their position in the datacenter

      [root@kubem00 ~]# kubectl label node kuben01 rack=rack01
      node "kuben01" labeled
      
      [root@kubem00 ~]# kubectl get nodes -l rack=rack01
      NAME      STATUS    AGE
      kuben01   Ready     2d
      
      [root@kubem00 ~]# kubectl label node kuben02 rack=rack01
      node "kuben02" labeled
      
      [root@kubem00 ~]# kubectl get nodes -l rack=rack01
      NAME      STATUS    AGE
      kuben01   Ready     2d
      kuben02   Ready     2d

Labels are also used as selector for services and deployments.

## Controllers
A Replication Controller ensures that a specified number of pod replicas are running at any one time. In other words, a Replication Controller makes sure that a pod or homogeneous set of pods are always up and available. If there are too many pods, it will kill some. If there are too few, it will start more. Unlike manually created pods, the pods maintained by a Replication Controller are automatically replaced if they fail, get deleted, or are terminated.

A Replica Controller configuration consists of:

 * The number of replicas desired
 * The pod definition
 * The selector to bind the managed pod

A selector is a label assigned to the pods that are managed by the replica controller. Labels are included in the pod definition that the replica controller instantiates. The replica controller uses the selector to determine how many instances of the pod are already running in order to adjust as needed.

In the ``nginx-rc.yaml`` file, define a replica controller with replica 1 for our nginx pod.
```yaml
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 3
  selector:
    run: nginx
  template:
    metadata:
      name: nginx
      labels:
        run: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Create a replica controller

    [root@kubem00 ~]# kubectl create -f nginx-rc.yaml
    replicationcontroller "nginx" created

List and describe a replica controller

    [root@kubem00 ~]# kubectl get rc
    NAME      DESIRED   CURRENT   READY     AGE
    nginx     3         3         3         1m

    [root@kubem00 ~]# kubectl describe rc nginx
    Name:           nginx
    Namespace:      default
    Image(s):       nginx:latest
    Selector:       run=nginx
    Labels:         run=nginx
    Replicas:       3 current / 3 desired
    Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
    No volumes.

The Replication Controller makes it easy to scale the number of replicas up or down, either manually or by an auto-scaling control agent, by simply updating the replicas field. For example, scale dow to zero replicas in order to delete all pods controlled by a given replica controller

    [root@kubem00 ~]# kubectl scale rc nginx --replicas=0
    replicationcontroller "nginx" scaled

    [root@kubem00 ~]# kubectl get rc nginx
    NAME      DESIRED   CURRENT   READY     AGE
    nginx     0         0         0         7m

Scale out the replica controller to create new pods

    [root@kubem00 ~]# kubectl scale rc nginx --replicas=9
    replicationcontroller "nginx" scaled

    [root@kubem00 ~]# kubectl get rc nginx
    NAME      DESIRED   CURRENT   READY     AGE
    nginx     9         9         0         9m

Also in case of failure of a node, the replica controller takes care of keep the same number of pods by scheduling the containers running on the failed node to the remaining nodes in the cluster.

To delete a replica controller

    [root@kubem00 ~]# kubectl delete rc/nginx
    replicationcontroller "nginx" deleted

## Deployments
A Deployment provides declarative updates for pods and replicas. You only need to describe the desired state in a Deployment object, and it will change the actual state to the desired state. The Deployment object defines the following details:

  * The elements of a Replication Controller definition
  * The strategy for transitioning between deployments

To create a deployment for our nginx webserver, edit the ``nginx-deploy.yaml`` file as
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  generation: 1
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

and create the deployment

    [root@kubem00 ~]# kubectl create -f nginx-deploy.yaml
    deployment "nginx" created

The deployment creates the following objects

    [root@kubem00 ~]# kubectl get all -l run=nginx

    NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    deploy/nginx   3         3         3            3           4m

    NAME                 DESIRED   CURRENT   READY     AGE
    rs/nginx-664452237   3         3         3         4m

    NAME                       READY     STATUS    RESTARTS   AGE
    po/nginx-664452237-h8dh0   1/1       Running   0          4m
    po/nginx-664452237-ncsh1   1/1       Running   0          4m
    po/nginx-664452237-vts63   1/1       Running   0          4m

According to the definitions set in the file, above, there are three pods and a replica set. The replica set object is very similar to the replica controller object we saw before. You may notice that the name of the replica set is always ``<name-of-deployment>-<hash value of the pod template>``

A deployment - as a replica controller, can be scaled up and down

    [root@kubem00 ~]# kubectl scale deploy nginx --replicas=6
    deployment "nginx" scaled

    [root@kubem00 ~]# kubectl get deploy nginx
    NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    nginx     6         6         6            3           11m

In addition to replicas management, a deployment also defines the strategy for updates pods

```yaml
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
```

In the snippet above, we set the update strategy as rolling update. During the lifetime of an application, some services need to be update, for example because the image changed. To update a service without an outage, Kubernetes updates one or more pod at a time, rather than taking down the entire application.

For example, to update the pods with a different version of nginx image

    [root@kubem00 ~]# kubectl set image deploy nginx nginx=nginx:1.9.1
    deployment "nginx" image updated

    [root@kubem00 ~]# kubectl rollout status deploy nginx
    Waiting for rollout to finish: 4 out of 6 new replicas have been updated...
    Waiting for rollout to finish: 4 out of 6 new replicas have been updated...
    Waiting for rollout to finish: 4 out of 6 new replicas have been updated...
    ...
    deployment "nginx" successfully rolled out

    [root@kubem00 ~]# kubectl get deploy
    NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    nginx     6         6         6            6           40m
    
    [root@kubem00 ~]# kubectl get rs
    NAME               DESIRED   CURRENT   READY     AGE
    nginx-2148973303   6         6         6         1m
    nginx-664452237    0         0         0         40m
    
    [root@kubem00 ~]# kubectl get pods
    NAME                     READY     STATUS    RESTARTS   AGE
    nginx-2148973303-29hff   1/1       Running   0          2m
    nginx-2148973303-91xt9   1/1       Running   0          1m
    nginx-2148973303-cnnt0   1/1       Running   0          2m
    nginx-2148973303-f2hr1   1/1       Running   0          1m
    nginx-2148973303-g5h4f   1/1       Running   0          1m
    nginx-2148973303-xcdlw   1/1       Running   0          1m

Deployment can ensure that only a certain number of pods may be down while they are being updated. By default, it ensures that at least 25% less than the desired number of pods are up. Deployment can also ensure that only a certain number of pods may be created above the desired number of pods. By default, it ensures that at most 25% more than the desired number of pods are up.

## Services
Kubernetes pods, as containers, are ephemeral. Replication Controllers create and destroy pods dynamically, e.g. when scaling up or down or when doing rolling updates. While each pod gets its own IP address, even those IP addresses cannot be relied upon to be stable over time. This leads to a problem: if some set of pods provides functionality to other pods inside the Kubernetes cluster, how do those pods find out and keep track of which other?

A Kubernetes Service is an abstraction which defines a logical set of pods and a policy by which to access them. The set of pods targeted by a Service is usually determined by a label selector. Kubernetes offers a simple Endpoints API that is updated whenever the set of pods in a service changes.

To create a service for our nginx webserver, edit the ``nginx-service.yaml`` file

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    run: nginx
spec:
  selector:
    run: nginx
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 80
  type: ClusterIP
```

Create the service

    [root@kubem00 ~]# kubectl create -f nginx-service.yaml
    service "nginx" created
    
    [root@kubem00 ~]# kubectl get service -l run=nginx
    NAME      CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    nginx     10.254.60.24   <none>        8000/TCP    38s

Describe the service

    [root@kubem00 ~]# kubectl describe service nginx
    Name:                   nginx
    Namespace:              default
    Labels:                 run=nginx
    Selector:               run=nginx
    Type:                   ClusterIP
    IP:                     10.254.60.24
    Port:                   <unset> 8000/TCP
    Endpoints:              172.30.21.3:80,172.30.4.4:80,172.30.53.4:80
    Session Affinity:       None
    No events.

The above service is associated to our previous nginx pods. Pay attention to the service selector ``run=nginx`` field. It tells Kubernetes that all pods with the label ``run=nginx`` are associated to this service, and should have traffic distributed amongst them. In other words, the service provides an abstraction layer, and it is the input point to reach all of the associated pods.

Pods can be added to the service arbitrarily. Make sure that the label ``run=nginx`` is associated to any pod we would to bind to the service. Define a new pod from the following file without (intentionally) any label
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mynginx
  namespace: default
  labels:
spec:
  containers:
  - name: mynginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Create the new pod

    [root@kubem00 ~]# kubectl create -f nginx-pod.yaml
    pod "mynginx" created

    [root@kubem00 ~]# kubectl get pods
    NAME                    READY     STATUS    RESTARTS   AGE
    mynginx                 1/1       Running   0          11s
    nginx-664452237-6h8zw   1/1       Running   0          16m
    nginx-664452237-kmmqk   1/1       Running   0          15m
    nginx-664452237-xhnjt   1/1       Running   0          16m

The just created new pod is not still associated to the nginx service

    [root@kubem00 ~]# kubectl get endpoints | grep nginx
    NAME         ENDPOINTS                                    AGE
    nginx        172.30.21.2:80,172.30.4.2:80,172.30.4.3:80   40m


Now, let's to lable the new pod with ``run=nginx`` label

    [root@kubem00 ~]# kubectl label pod mynginx run=nginx
    pod "mynginx" labeled

We can see a new endpoint is added to the service

    [root@kubem00 ~]# kubectl get endpoints | grep nginx
    NAME         ENDPOINTS                                                AGE
    nginx        172.30.21.2:80,172.30.4.2:80,172.30.4.3:80 + 1 more...   46m

Any pod in the cluster need for the nginx service will be able to talk with this service by the service address no matter which IP address will be assigned to the nginx pod. Also, in case of multiple nginx pods, the service abstraction acts as load balancer between the nginx pods.

The picture below shows the relationship between services and pods

![](../img/service-pod.png?raw=true)

Create a pod from the below yaml file
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```

We'll use this pod to address the nginx service

    [root@kubem00 ~]# kubectl create -f busybox.yaml
    pod "busybox" created

    [root@kubem00 ~]# kubectl exec -it busybox sh
    / # wget -O - 10.254.105.187:8000
    Connecting to 10.254.105.187:8000 (10.254.105.187:8000)
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>
    ...
    </body>
    </html>
    / # exit
    [root@kubem00 ~]#

In kubernetes, the service abstraction acts as stable entrypoint for any application, no matter wich is the IP address of the pod(s) running that application. We can destroy all nginx pods and recreate but service will be always the same IP address and port.

## Volumes
Containers are ephemeral, meaning the container file system only lives as long as the container does. If the application state needs to survive relocation, reboot, and crash of the hosting pod, we need to use some persistent storage mechanism. Kubernetes - as docker does, uses the concept of data volume.  A kubernetes volume has an explicit lifetime, the same as the pod that encloses it. Data stored in a kubernetes volume survive across relocation, reboot, restart and crash of the hosting pod.

In this section, we are going to create two different volume types:

  1. **emptyDir**
  2. **hostPath**

However, kubernetes supports more other volume types. Please, refer to official documentation for details.

Here a simple example file ``nginx-volume.yaml`` of nginx pod containing a data volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: content-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: content-data
    emptyDir: {}
```

The file defines an nginx pod where the served html content directory ``/usr/share/nginx/html`` is mounted as volume. The volume type is ``emptyDir``. This type of volume is created when the pod is assigned to a node, and it exists as long as that pod is running on that node. As the name says, it is initially empty. When the pod is removed from the node for any reason, the data in the emptyDir volume is deleted too. However, a container crash does not remove a pod from a node, so the data in an emptyDir volume is safe across a container crash.

To test this, create the nginx pod above

    kubectl create -f nginx-volume.yaml
    pod "nginx" created
    
    kubectl get po/nginx
    NAME      READY     STATUS    RESTARTS   AGE
    nginx     1/1       Running   0          22s

Check for pod IP address and for node IP address hosting the pod

    kubectl get po/nginx -o yaml | grep IP
    hostIP: 10.10.10.86
    podIP: 172.30.5.3

Trying to access the nginx application

     curl 172.30.5.3:80
     403 Forbidden

we get *forbidden* since the html content dir (mounted as volume) is initially empty.

Login to the nginx pod and populate the volume dir

    kubectl exec -it nginx /bin/bash
    date > /usr/share/nginx/html/index.html
    root@nginx:/# exit
    exit

Now we should be able to get an answer

    curl 172.30.5.3:80
    Wed Apr 26 08:35:53 UTC 2017

Now we are going to test the volume persistance across container crash or restart.

In another window terminal, login to the master and watch for nginx pod changes

    kubectl get --watch pod nginx

    NAME      READY     STATUS    RESTARTS   AGE
    nginx     1/1       Running   0          8m

In another terminal window, login to the worker node where pod is running and kill the nginx process running inside the container

    ps -ef | grep nginx
    root      1976 12685  0 10:29 pts/0    00:00:00 grep --color=auto nginx
    root     25607 25592  0 10:18 ?        00:00:00 nginx: master process nginx -g daemon off;
    101      25620 25607  0 10:18 ?        00:00:00 nginx: worker process

    kill -9 25607

    ps -ef | grep nginx
    root      2321  2306  0 10:29 ?        00:00:00 nginx: master process nginx -g daemon off;
    101       2334  2321  0 10:29 ?        00:00:00 nginx: worker process
    root      2383 12685  0 10:30 pts/0    00:00:00 grep --color=auto nginx

Since the container nginx process runs inside a pod, the container process is restarted by kubernetes as we can see from the watching window

    kubectl get --watch pod nginx
    NAME      READY     STATUS    RESTARTS   AGE
    nginx     1/1       Running   0          8m
    nginx     0/1       Error     0         11m
    nginx     1/1       Running   1         11m

Since the html content directory is mounted as volume inside the pod, data inside the dir is not impacted by the crash and following restart of nginx container. Trying to access the content, we'll get the same content before the container restart

    curl 172.30.5.3:80
    Wed Apr 26 08:35:53 UTC 2017

Attention: with the ``emptyDir`` volume type, data in the volume is removed when the pod is deleted from the node where it was running. To achieve data persistence across pod deletion or relocation, we need for a persistent shared storage alternative. 

The other volume type we're gooing to use is ``hostPath``. With this volume type, the volume is mount from an existing directory on the file system of the node hosting the pod. Data inside the host directory are safe to container crashes and restarts as well as to pod deletion. However, if the pod is moved from a node to another one, data on the initial node are no more accessible from the new instance of the pod.

Based on the previous example, define a nginx pod using the host path ``/data/nginx/html`` as data volume for html content
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: content-data
      mountPath: /usr/share/nginx/html
    nodeSelector:
    kubernetes.io/hostname: kuben05
  volumes:
  - name: content-data
    hostPath:
      path: /data/nginx/html
```

Please, note that host dir must be present on the node hosting the pod before the pod is created by kubernetes. For this reason, we force kubernetes to schedule the pod on a specific node having hostname ``kuben05`` where the volume directory has been first created.

Login to the worker node and create the host directory

    [root@kuben05 ~]# mkdir -p /data/nginx/html

Back to the master node and schedule the pod

    kubectl create -f nginx-host-volume.yaml
    pod "nginx" created
    
    kubectl get pod nginx
    NAME      READY     STATUS    RESTARTS   AGE
    nginx     1/1       Running   0          3m

Check for pod IP address and try to access the nginx application

    kubectl get po/nginx -o yaml | grep podIP
    podIP: 172.30.41.7
    
    curl 172.30.41.7:80
    403 Forbidden

we get *forbidden* since the html content dir (mounted as volume on the host node) is initially empty.

Login to the host node pod populate the volume dir

    [root@kuben05 ~]# echo "Hello from $(hostname)"  > /data/nginx/html/index.html

Back to the master node and access the service

    curl 172.30.41.7:80
    Hello from kuben05

Data in host dir volume will survive to any crash and restart of both container and pod. To test this, delete the pod and create it again

    kubectl delete pod nginx
    pod "nginx" deleted

    kubectl create -f nginx-host-volume.yaml
    pod "nginx" created

    kubectl get po/nginx -o yaml | grep podIP
      podIP: 172.30.41.7

    curl 172.30.41.7:80
    Hello from kuben05

This works because we forced kubernetes to schedule the nginx pod always on the same host node.
