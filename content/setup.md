# Setup a Kubernetes Cluster
This tutorial refers to a cluster of nodes (virtual, physical or a mix of both) running CentOS 7.3 Operating System. We'll set Kubernetes components as system processes managed by systemd.

   * [Requirements](#requirements)
   * [Configure Master](#configure-masters)
   * [Configure Workers](#configure-workers)
   * [Test the cluster](#test-the-cluster)
   * [Configure DNS service](#configure-dns-service)
   * [Configure GUI dashboard](#configure-gui-dashboard)
   
## Requirements
Our initial cluster will be made of 1 Master node and 3 Workers nodes. All machines can be virtual or physical or a mix of both. Minimum hardware requirements are: 2 vCPUs, 2GB of RAM, 16GB HDD for OS. All machines will be installed with Linux CentOS 7.3. Firewall and Selinux will be disabled. An NTP server is installed and running on all machines. On worker nodes, Docker is installed with a Device Mapper on a separate 10GB HDD. Internet access.

Here the hostnames:

   * kubem00 10.10.10.80 (master)
   * kubew03 10.10.10.83 (worker)
   * kubew04 10.10.10.84 (worker)
   * kubew05 10.10.10.85 (worker)

Make sure to enable DNS resolution for the above hostnames or set the ``/etc/hosts`` file on all the machines.

In this tutorial we are not going to provision any overlay networks and instead rely on Layer 3 networking between machines. That means we need to add routes to our hosts. Make sure IP forwarding kernel option is enabled on all hosts

    cat /etc/sysctl.conf
      net.ipv4.ip_forward = 1
    sysctl -p /etc/sysctl.conf

The IP address space for containers will be allocated from the ``10.38.0.0/16`` cluster range assigned to each Kubernetes worker through the node registration process. Based on the above configuration each node will receive a 24-bit subnet

    10.38.0.0/24
    10.38.1.0/24
    10.38.2.0/24

In addition, Kubernetes allocates IP addresses for internal services. These addresses are not required to be routable outside the cluster itself. However, each node needs to know where to find the cluster's subnets.

Here the releases we'll use during this tutorial

   * Kubernetes 1.7.0
   * Docker 1.12.6
   * Etcd 3.2.6

## Configure Masters
On the Master, first install etcd and kubernetes

    wget https://github.com/coreos/etcd/releases/download/v3.2.6/etcd-v3.2.6-linux-amd64.tar.gz
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kube-apiserver
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kube-controller-manager
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kube-scheduler
    wget https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kubectl

Estract and install the binaries

    tar -xvf etcd-v3.2.6-linux-amd64.tar.gz && mv etcd-v3.2.6-linux-amd64/etcd* /usr/bin/
    chmod +x kube-apiserver && mv kube-apiserver /usr/bin/
    chmod +x kube-controller-manager && mv kube-controller-manager /usr/bin/
    chmod +x kube-scheduler && mv kube-scheduler /usr/bin/
    chmod +x kubectl && mv kubectl /usr/bin/

### Configure etcd
Create the etcd data directory

    mkdir -p /var/lib/etcd

Before launching and enabling the etcd service, set options in the ``/etc/systemd/system/etcd.service`` startup file

    [Unit]
    Description=etcd
    Documentation=https://github.com/coreos
    After=network.target
    After=network-online.target
    Wants=network-online.target

    [Service]
    Type=notify
    ExecStart=/usr/bin/etcd \
      --name kubem00 \
      --initial-advertise-peer-urls http://10.10.10.80:2380 \
      --listen-peer-urls http://10.10.10.80:2380 \
      --listen-client-urls http://10.10.10.80:2379,http://127.0.0.1:2379 \
      --advertise-client-urls http://10.10.10.80:2379 \
      --initial-cluster-token etcd-cluster-token \
      --initial-cluster kubem00=http://10.10.10.80:2380 \
      --initial-cluster-state new \
      --data-dir=/var/lib/etcd \
      --debug=false

    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

Start and enable the service

    systemctl daemon-reload
    systemctl start etcd
    systemctl enable etcd
    systemctl status etcd

### Configure API server
To configure the API server, configure the ``/etc/systemd/system/kube-apiserver.service`` startup file

    [Unit]
    Description=Kubernetes API Server
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target
    After=etcd.service

    [Service]
    Type=notify
    ExecStart=/usr/bin/kube-apiserver \
      --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
      --anonymous-auth=false \
      --etcd-servers=http://10.10.10.80:2379 \
      --advertise-address=10.10.10.80 \
      --allow-privileged=true \
      --audit-log-maxage=30 \
      --audit-log-maxbackup=3 \
      --audit-log-maxsize=100 \
      --audit-log-path=/var/lib/audit.log \
      --enable-swagger-ui=true \
      --event-ttl=1h \
      --insecure-bind-address=0.0.0.0 \
      --service-cluster-ip-range=10.32.0.0/16 \
      --service-node-port-range=30000-32767 \
      --v=2
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target



Start and enable the service

    systemctl daemon-reload
    systemctl start kube-apiserver
    systemctl enable kube-apiserver
    systemctl status kube-apiserver

### Configure controller manager
To configure the kubernetes controller manager, create the ``/etc/systemd/system/kube-controller-manager.service`` startup file

    [Unit]
    Description=Kubernetes Controller Manager
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target

    [Service]
    ExecStart=/usr/bin/kube-controller-manager \
      --address=0.0.0.0 \
      --allocate-node-cidrs=true \
      --cluster-cidr=10.38.0.0/16 \
      --cluster-name=kubernetes \
      --master=http://10.10.10.80:8080 \
      --service-cluster-ip-range=10.32.0.0/16 \
      --v=2

    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

Start and enable the service

    systemctl daemon-reload
    systemctl start kube-controller-manager
    systemctl enable kube-controller-manager
    systemctl status kube-controller-manager
    
### Configure scheduler
To configure the kubernetes scheduler, create the ``/etc/systemd/system/kube-scheduler.service`` startup file

    [Unit]
    Description=Kubernetes Scheduler
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target

    [Service]
    ExecStart=/usr/bin/kube-scheduler \
      --master=http://10.10.10.80:8080 \
      --v=2

    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

Start and enable the service

    systemctl daemon-reload
    systemctl start kube-scheduler
    systemctl enable kube-scheduler
    systemctl status kube-scheduler

### Configure CLI
On the master node, configure the ``kubectl`` command line. Please not it is not required to enable kubectl on the master node. It can be installed and configured on any machine that is able to reach the API Server.

The contest defines the namespace as well as the cluster and the user accessing the resources.

Create a context for default admin access to the cluster

    kubectl config set-credentials root
    User "root" set.

    kubectl config set-cluster unsecure-kubernetes --server=http://10.10.10.80:8080
    Cluster "unsecure-kubernetes" set.

    kubectl config set-context default/unsecure-kubernetes/root --cluster=unsecure-kubernetes --user=root --namespace=default
    Context "default/unsecure-kubernetes/root" set.

then enable the context

    kubectl config use-context default/unsecure-kubernetes/root
    Switched to context "default/unsecure-kubernetes/root"
    
and check it

    kubectl config current-context
    default/unsecure-kubernetes/root

Now it is possible to query and operate with the cluster

    kubectl get componentstatuses
    NAME                 STATUS    MESSAGE              ERROR
    scheduler            Healthy   ok
    controller-manager   Healthy   ok
    etcd-0               Healthy   {"health": "true"}

It is possible to define more contexts and switch between different contexts with different scopes and priviledges. The contexts are stored in the hidden file ``.kube/conf`` under the user home directory

    apiVersion: v1
    clusters:
    - cluster:
        server: http://10.10.10.90:8080
      name: unsecure-kubernetes
    contexts:
    - context:
        cluster: unsecure-kubernetes
        namespace: default
        user: root
      name: default/unsecure-kubernetes/admin
    current-context: default/unsecure-kubernetes/root
    kind: Config
    preferences: {}
    users:
    - name: root
      user: {}

## Configure Workers
On all the worker nodes, install kubernetes and docker

    yum -y install kubernetes docker

### Configure Docker
There are a number of ways to customize the Docker daemon flags and environment variables. The recommended way from Docker web site is to use the platform-independent ``/etc/docker/daemon.json`` file instead of the systemd unit file. This file is available on all the Linux distributions

```json
{
 "debug": true,
 "storage-driver": "devicemapper",
 "iptables": false,
 "ip-masq": false
}
```

On CentOS systems, the suggested storage mapper is the Device Mapper.

Also, since Kubernetes uses a different network model than Docker, we need to prevent Docker to use NAT/IP Table rewriting. For this reason, we disable the IP Table and NAT options in Docker daemon.

Start and enable the docker service

    systemctl start docker
    systemctl enable docker
    systemctl status docker

As usual, Docker will create the default ``docker0`` bridge network interface

    ifconfig docker0
    docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
            inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
            ether 02:42:c3:64:b4:7f  txqueuelen 0  (Ethernet)

However, we're not going to use it since Kubernetes networking is based on the **CNI** Container Network Interface.

### Setup the CNI network plugins
Download the CNI network plugins from [here](https://github.com/kalise/Kubernetes-Lab-Tutorial/raw/master/cni-plugins/cni-amd64.tar.gz) and place them in the expected directory

    mkdir -p /opt/cni
    tar -xvf cni-amd64.tar.gz -C /opt/cni

### Configure kubelet
To configure the kubelet component, edit the ``/usr/lib/systemd/system/kubelet.service`` startup file

    [Unit]
    Description=Kubernetes Kubelet Server
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=docker.service
    Requires=docker.service

    [Service]
    WorkingDirectory=/var/lib/kubelet
    ExecStart=/usr/bin/kubelet \
      --network-plugin=kubenet \
      --allow-privileged=true \
      --api-servers=http://10.10.10.80:8080 \
      --cluster-dns=10.32.0.10 \
      --cluster-domain=cluster.local \
      --container-runtime=docker \
      --register-node=true

    Restart=on-failure

    [Install]
    WantedBy=multi-user.target

The cluster DNS parameter above refers to the IP of the DNS internal service used by Kubernetes for service discovery. It should be in the IP addressese space used for internal services. We're going to configure it soon.

Start and enable the kubelet service

    systemctl daemon-reload
    systemctl start kubelet
    systemctl enable kubelet
    systemctl status kubelet

### Configure proxy
To configure the proxy component, edit the ``/usr/lib/systemd/system/kube-proxy.service`` startup file

    [Unit]
    Description=Kubernetes Kube-Proxy Server
    Documentation=https://github.com/GoogleCloudPlatform/kubernetes
    After=network.target

    [Service]
    ExecStart=/usr/bin/kube-proxy \
      --master=http://10.10.10.80:8080 \
      --cluster-cidr=10.38.0.0/16 \
      --proxy-mode=iptables \
      --masquerade-all=true

    Restart=on-failure
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

Start and enable the service

    systemctl daemon-reload
    systemctl start kube-proxy
    systemctl enable kube-proxy
    systemctl status kube-proxy


## Define the Container Network Routes
Now that each worker node is online we need to add routes to make sure that containers running on different machines can talk to each other. The first thing we need to do is gather the information required to populate the nodes routing table.

On the master node, use kubectl to gather the IP addresses

    kubectl get nodes \
     -o jsonpath='{range .items[*]} {.spec.externalID} {.status.addresses[?(@.type=="InternalIP")].address} {.spec.podCIDR} {"\n"}{end}'
    
    kube01 10.10.10.81 10.38.0.0/24
    kube02 10.10.10.82 10.38.1.0/24
    kube03 10.10.10.83 10.38.2.0/24

On the master node, given ``ens160`` the cluster network interface, create the script file ``/etc/sysconfig/network-scripts/route-ens160`` for adding permanent static routes containing the following

    10.38.0.0/24 via 10.10.10.81
    10.38.1.0/24 via 10.10.10.82
    10.38.2.0/24 via 10.10.10.83

Restart the network service and check the master routing table

    systemctl restart network
    
    netstat -nr
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    0.0.0.0         10.10.10.1      0.0.0.0         UG        0 0          0 ens160
    10.10.10.0      0.0.0.0         255.255.255.0   U         0 0          0 ens160
    10.38.0.0       10.10.10.81     255.255.255.0   UG        0 0          0 ens160
    10.38.1.0       10.10.10.82     255.255.255.0   UG        0 0          0 ens160
    10.38.2.0       10.10.10.83     255.255.255.0   UG        0 0          0 ens160

On all the worker nodes, create a similar script file.

For example, on the worker ``kube01``, create the file ``/etc/sysconfig/network-scripts/route-ens160`` containing the following

    10.38.1.0/24 via 10.10.10.82
    10.38.2.0/24 via 10.10.10.83

Restart the network service and check the master routing table

    systemctl restart network
    
    netstat -nr
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
    0.0.0.0         10.10.10.1      0.0.0.0         UG        0 0          0 ens160
    10.10.10.0      0.0.0.0         255.255.255.0   U         0 0          0 ens160
    10.38.0.0       0.0.0.0         255.255.255.0   U         0 0          0 cbr0
    10.38.1.0       10.10.10.84     255.255.255.0   UG        0 0          0 ens160
    10.38.2.0       10.10.10.85     255.255.255.0   UG        0 0          0 ens160
    172.17.0.0      0.0.0.0         255.255.0.0     U         0 0          0 docker0

Repeat the steps for all workers paying attention to set the routes correctly. Finally, check the nodes can reach each other on these IP addresses.

## Test the cluster
The cluster should be now running. Check to make sure the cluster can see the node, by logging to the master

    kubectl get nodes
    NAME      STATUS    AGE
    kube01   Ready     2d
    kube02   Ready     2d
    kube03   Ready     2d

Kubernetes cluster stores all of its internal state in etcd. The idea is, that you should interact with Kubernetes only via its API provided by API service. API service abstracts away all the Kubernetes cluster state manipulating by reading from and writing into the etcd cluster. Let’s explore what’s stored in the etcd cluster after fresh installation:

    [root@kube00 ~]# etcdctl ls
    /registry

The etcd has the /registry area where stores all info related to the cluster. The /kube-centos area contains the flannel network configuration.

Let's see the cluster info stored in etcd

    [root@kube00 ~]# etcdctl ls /registry --recursive
    /registry/deployments
    /registry/deployments/default
    /registry/deployments/kube-system
    /registry/ranges
    /registry/ranges/serviceips
    /registry/ranges/servicenodeports
    /registry/secrets
    /registry/secrets/default
    /registry/secrets/default/default-token-92z22
    /registry/secrets/kube-system
    /registry/secrets/kube-system/default-token-sj7t3
    /registry/secrets/demo
    ...

For example, get detailed info about a worker node

    [root@kubem00 ~]# etcdctl ls /registry/minions
    /registry/minions/kube01
    /registry/minions/kube02
    /registry/minions/kube03
    
    [root@kubem00 ~]# etcdctl get /registry/minions/kube01 | jq .


## Configure DNS service
To enable service name discovery in our Kubernetes cluster, we need to configure an embedded DNS service. To do so, we need to deploy DNS pod and service having configured kubelet to resolve all DNS queries from this local DNS service.

Login to the master node and download the DNS template ``kubedns-template.yaml`` from [here](https://github.com/kalise/Kubernetes-Lab-Tutorial/blob/master/examples/kubedns-template.yaml)

This template defines a Replica Controller and a DNS service. The controller defines three containers running on the same pod: a DNS server, a dnsmaq for caching and healthz for liveness probe:
```yaml
...
    spec:
      containers:
      - name: kubedns
        image: gcr.io/google_containers/kubedns-amd64:1.8
...
      - name: dnsmasq
        image: gcr.io/google_containers/kube-dnsmasq-amd64:1.4
...
      - name: healthz
        image: gcr.io/google_containers/exechealthz-amd64:1.2
```

Here the DNS service definition
```yaml
...
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.254.3.100
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

Note: make sure you have updated in the above file the correct cluster IP ``10.254.3.100`` as we specified in kubelet configuration file for DNS service ``--cluster-dns=10.254.3.100`` option.


Create the DNS for service discovery

    [root@kube00 ~]# kubectl create -f kubedns-template.yaml
    replicationcontroller "kube-dns-v20" created
    service "kube-dns" created

and check if it works in the dedicated namespace

    [root@kube00 ~]# kubectl get all -n kube-system
    NAME              DESIRED   CURRENT   READY     AGE
    rc/kube-dns-v20   1         1         1         22m
    NAME           CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
    svc/kube-dns   10.254.3.100   <none>        53/UDP,53/TCP   22m
    NAME                    READY     STATUS    RESTARTS   AGE
    po/kube-dns-v20-3xk4v   3/3       Running   0          22m

To test if it works, create a file named ``busybox.yaml`` with the following contents:
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

Then create a pod using this file

    kubectl create -f busybox.yaml
    
wait for pod is running and validate that DNS is working by resolving the kubernetes service

    kubectl exec -ti busybox -- nslookup kubernetes    
    Server:    10.254.3.100
    Address 1: 10.254.3.100 kube-dns.kube-system.svc.cluster.local
    Name:      kubernetes
    Address 1: 10.254.0.1 kubernetes.default.svc.cluster.local

Take a look inside the ``resolv.conf file`` of the busybox container
    
    kubectl exec busybox cat /etc/resolv.conf
    search default.svc.cluster.local svc.cluster.local cluster.local
    nameserver 10.254.3.100
    nameserver 8.8.8.8
    options ndots:5

Each time a new service starts on the cluster, it will register with the DNS letting all the pods to reach the new service.

## Configure GUI dashboard
Kubernetes dashboard provides a GUI through which we can manage Kubernetes work units. We can create, delete or edit all work unit from this dashboard. Kubernetes dashboard is deployed as a pod in a dedicated namespace.

Download the dashboard deploy from [here](https://github.com/kalise/Kubernetes-Lab-Tutorial/blob/master/examples/kubegui-deploy.yaml) and deploy it

    kubectl create -f kubegui-deploy.yaml
    deployment "kubernetes-dashboard" created

Download the dashboard service from [here](https://github.com/kalise/Kubernetes-Lab-Tutorial/blob/master/examples/kubegui-svc.yaml) and expose to the external port 8080

    kubectl create -f kubegui-svc.yaml
    service "kubernetes-dashboard" created

When objects are ready

    kubectl get all -n kube-system
    
    NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    deploy/kubernetes-dashboard   1         1         1            1           23s
    NAME              DESIRED   CURRENT   READY     AGE
    rc/kube-dns-v20   1         1         1         1d
    NAME                       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
    svc/kube-dns               10.254.3.100     <none>        53/UDP,53/TCP   1d
    svc/kubernetes-dashboard   10.254.180.188   <none>        80/TCP          18s
    NAME                                 DESIRED   CURRENT   READY     AGE
    rs/kubernetes-dashboard-3543765157   1         1         1         23s
    NAME                                       READY     STATUS    RESTARTS   AGE
    po/kube-dns-v20-3xk4v                      3/3       Running   3          1d
    po/kubernetes-dashboard-3543765157-tbc49   1/1       Running   0          23s


point the browser to the public master IP address ``http://10.10.10.80:8080/ui``. Please, note, the dashboard requires the embedded DNS server for service discovery.
