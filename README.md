# Command Line

```bash

$ cd /vagrant/

$ ./hack/setup-vms.sh

```


# Kubeadm Ansible Playbook

Build a Kubernetes cluster using Ansible with kubeadm. The goal is easily install a Kubernetes cluster on machines running:

  - Ubuntu 16.04
  - CentOS 7
  - Debian 9

System requirements:

  - Deployment environment must have Ansible `2.4.0+`
  - Master and nodes must have passwordless SSH access

# Usage

Add the system information gathered above into a file called `hosts.ini`. For example:
```
[master]
192.16.35.12

[node]
192.16.35.[10:11]

[kube-cluster:children]
master
node
```

If you're working with ubuntu, add the following properties to each host `ansible_python_interpreter='python3'`:
```
[master]
192.16.35.12 ansible_python_interpreter='python3'

[node]
192.16.35.[10:11] ansible_python_interpreter='python3'

[kube-cluster:children]
master
node

```

Before continuing, edit `group_vars/all.yml` to your specified configuration.

For example, I choose to run `flannel` instead of calico, and thus:

```yaml
# Network implementation('flannel', 'calico')
network: flannel
```

**Note:** Depending on your setup, you may need to modify `cni_opts` to an available network interface. By default, `kubeadm-ansible` uses `eth1`. Your default interface may be `eth0`.

After going through the setup, run the `site.yaml` playbook:

```sh
$  ansible-playbook site.yaml -i hosts.ini
...
==> master1: TASK [addon : Create Kubernetes dashboard deployment] **************************
==> master1: changed: [192.16.35.12 -> 192.16.35.12]
==> master1:
==> master1: PLAY RECAP *********************************************************************
==> master1: 192.16.35.10               : ok=18   changed=14   unreachable=0    failed=0
==> master1: 192.16.35.11               : ok=18   changed=14   unreachable=0    failed=0
==> master1: 192.16.35.12               : ok=34   changed=29   unreachable=0    failed=0
```

Download the `admin.conf` from the master node:

```sh
$ scp k8s@k8s-master:/etc/kubernetes/admin.conf .
```

Verify cluster is fully running using kubectl:

```sh

$ export KUBECONFIG=~/admin.conf
$ kubectl get node
NAME      STATUS    AGE       VERSION
master1   Ready     22m       v1.6.3
node1     Ready     20m       v1.6.3
node2     Ready     20m       v1.6.3

$ kubectl get po -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
etcd-master1                            1/1       Running   0          23m
...
```

# Resetting the environment

Finally, reset all kubeadm installed state using `reset-site.yaml` playbook:

```sh
$ ansible-playbook reset-site.yaml
```
**IMPORTANT:** HTTPS endpoints are only available if you used [Recommended Setup](https://github.com/kubernetes/dashboard/wiki/Installation#recommended-setup), followed [Getting Started](https://github.com/kubernetes/dashboard/blob/master/README.md#getting-started) guide to deploy Dashboard or manually provided `--tls-key-file` and `--tls-cert-file` flags. In case you did not and you access Dashboard over HTTP, then Dashboard can be accessed the same way as [older versions](https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.6.X-and-below).

**NOTE**: Dashboard should not be exposed publicly over HTTP. For domains accessed over HTTP it will not be possible to sign in. Nothing will happen after clicking Sign in button on login page.

## `kubectl proxy`

`kubectl proxy` creates proxy server between your machine and Kubernetes API server. By default it is only accessible locally (from the machine that started it).

First let's check if `kubectl` is properly configured and has access to the cluster. In case of error follow [this guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to install and set up `kubectl`.
```sh
$ kubectl cluster-info
# Example output
Kubernetes master is running at https://192.168.30.148:6443
Heapster is running at https://192.168.30.148:6443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://192.168.30.148:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Start local proxy server.
```sh
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

Once proxy server is started you should be able to access Dashboard from your browser.

To access HTTPS endpoint of dashboard go to: `http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

**NOTE:** Dashboard should not be exposed publicly using `kubectl proxy` command as it only allows HTTP connection. For domains other than `localhost` and `127.0.0.1` it will not be possible to sign in. Nothing will happen after clicking `Sign in` button on login page.


### Configure kubectl

Run the following command as `user` and `root` to configure `kubectl` command line CLI tool to communicate with the Kubernetes environment.

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check Kubernetes version

```
$ kubectl version --short

Client Version: v1.15.6
Server Version: v1.15.6
```

Untaint the node - this is required since we have only one VM to install objects.

```
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

Check node status and note that it is not yet ready since we have not yet installed a pod network

```
$ kubectl get nodes
NAME    STATUS     ROLES    AGE   VERSION
osc01   NotReady   master   95s   v1.15.6
```

Check pod status in `kube-system` and you will notice that `coredns` pods are in pending state. This is due to the fact that we have not yet installed the pod network. 

Make sure that `etcd`, `kube-apiserver`  , `kube-controller-manager`, `kube-proxy` and `kube-scheduler` pods are showing READY state `1/1` and Status `Runnning`.

```
$ kubectl get pods -A
NAME                            READY   STATUS    RESTARTS   AGE
coredns-bb49df795-lcjvx         0/1     Pending   0          119s
coredns-bb49df795-wqmzb         0/1     Pending   0          119s
etcd-osc01                      1/1     Running   0          80s
kube-apiserver-osc01            1/1     Running   0          60s
kube-controller-manager-osc01   1/1     Running   0          58s
kube-proxy-vprqc                1/1     Running   0          119s
kube-scheduler-osc01            1/1     Running   0          81s
```

## Install Calico network for pods

Choose proper version of Calico [Link](https://docs.projectcalico.org/v3.10/getting-started/kubernetes/requirements)

Calico 3.10 is tested with Kubernetes versions 1.14, 1.15 and 1.16

```
$ export POD_CIDR=10.142.0.0/16
$ curl https://docs.projectcalico.org/v3.14/manifests/calico.yaml -O
$ sed -i -e "s?192.168.0.0/16?$POD_CIDR?g" calico.yaml
$ kubectl apply -f calico.yaml
```

It may take a while to pull Calico images over the slow network.

Check docker images being pulled.
```
$ sudo docker images calico/*
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
calico/node                 v3.10.1             4a88ba569c29        11 days ago         192MB
calico/cni                  v3.10.1             4f761b4ba7f5        11 days ago         163MB
calico/kube-controllers     v3.10.1             8f87d09ab811        11 days ago         50.6MB
calico/pod2daemon-flexvol   v3.10.1             5b249c03bee8        11 days ago         9.78MB
```

Check the status of the cluster and wait for all pods to be in `Running` and `Ready 1/1` state.

```
$ kubectl get pods -A
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-866db6d5f7-w9mfq   1/1     Running   0          33s
calico-node-mwgzx                          1/1     Running   0          33s
coredns-bb49df795-lcjvx                    1/1     Running   0          4m
coredns-bb49df795-wqmzb                    1/1     Running   0          4m
etcd-osc01                                 1/1     Running   0          3m21s
kube-apiserver-osc01                       1/1     Running   0          3m1s
kube-controller-manager-osc01              1/1     Running   0          2m59s
kube-proxy-vprqc                           1/1     Running   0          4m
kube-scheduler-osc01                       1/1     Running   0          3m22s
```

Our single node basic Kubernetes cluster is now up and running.

```
$ kubectl get nodes -o wide
NAME    STATUS   ROLES    AGE     VERSION    INTERNAL-IP       ---
osc01   Ready    master   5m28s   v1.15.0    192.168.142.101   ---


--- EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
--- <none>        CentOS Linux 7 (Core)   3.10.0-957.21.3.el7.x86_64   docker://18.9.8
```

## Create an admin account

```
$ kubectl --namespace kube-system create serviceaccount admin
```

Grant Cluster Role Binding to the `admin` account

```
$ kubectl create clusterrolebinding admin --serviceaccount=kube-system:admin --clusterrole=cluster-admin
```

## Install kubectl on client machines

We will use the existing VM - which already has `kubectl` and the GUI to run a browser. 

However, you can use `kubectl` from a client machine to manage the Kubernetes environment. Follow the [link](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for installing `kubectl` on your chice of client machine (Windows, MacBook or Linux). 

## Install busybox to check

```
$ kubectl create -f https://k8s.io/examples/admin/dns/busybox.yaml
```

## Install hostname deployment for sanity checks

Create a deployment
```
kubectl run hostnames --image=k8s.gcr.io/serve_hostname \
                        --labels=app=hostnames \
                        --port=9376 \
                        --replicas=3
```

Create a service

```
$ kubectl expose deployment hostnames --port=80 --target-port=9376
```



## Sanity check for the cluster

[Check this link for testing the cluster](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)

Check pod
```
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          13s
```

## Install helm and tiller 

Starting with Helm 3, the tiller will not be required. We will be using Helm 2.x related charts, so we will not be installing Helm 3.x until charts are migrated to Helm 3.x. 

Since Helm charts that we are going to use still However, we will be installing Helm v2.16.7

In principle tiller can be installed using `helm init`.

```
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.16.7-linux-amd64.tar.gz | tar xz

$ cd linux-amd64
$ sudo mv helm /bin
```

Create `tiller` service accoun and grant cluster admin to the `tiller` service account.

```
$ kubectl -n kube-system create serviceaccount tiller

$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
```

Helm can be installed with and without security. If no security is required (like demo/test environment), follow Option - 1 or follow option - 2 to install helm with security.

### Option - 1 : No security, ideal for running in a sandbox environment.

Initialize the `helm` and it will install `tiller` server in Kubernetes.

```
$ helm init --service-account tiller
```

Wait for tiller to get deployed. (check `kubectl get pods -A`)

Check helm version

```
$ helm version --short

Client: v2.16.1+gbbdfe5e
Server: v2.16.1+gbbdfe5e
```

If you installed helm without secruity, skip to the [next](#Install-Kubernetes-dashboard) section.

### Option - 2 : With TLS security, ideal for running in production.

Install step

```
$ curl -LOs  https://github.com/smallstep/cli/releases/download/v0.14.4/step_linux_0.14.4_amd64.tar.gz

$ tar xvfz step_linux_0.14.4_amd64.tar.gz

$ sudo mv  step_0.14.4/bin/step /bin

$ mkdir -p ~/helm
$ cd ~/helm
$ step certificate create --profile root-ca "My iHelm Root CA" root-ca.crt root-ca.key
$ step certificate create intermediate.io inter.crt inter.key --profile intermediate-ca --ca ./root-ca.crt --ca-key ./root-ca.key
$ step certificate create helm.io helm.crt helm.key --profile leaf --ca inter.crt --ca-key inter.key --no-password --insecure --not-after 17520h
$ step certificate bundle root-ca.crt inter.crt ca-chain.crt

$ helm init \
--override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}' \
--tiller-tls --tiller-tls-verify \
--tiller-tls-cert=./helm.crt \
--tiller-tls-key=./helm.key \
--tls-ca-cert=./ca-chain.crt \
--service-account=tiller

$ cd ~/.helm
$ cp ~/helm/helm.crt cert.pem
$ cp ~/helm/helm.key key.pem
$ rm -fr ~/helm ## Copy dir somewhere and protect it.
```

## Update helm repository

Update Helm repo

```
$ helm repo update
```

If secure helm is used, use --tls at the end of helm commands to use TLS between helm and server.

List Helm repo

```
$ helm repo list
NAME  	URL                                             
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts   
```

## Install Kubernetes dashboard

Install kubernetes dashboard helm chart

```
$ helm install stable/kubernetes-dashboard --name k8web --namespace kube-system --set fullnameOverride="dashboard"

Note: add --tls above if using secure helm
```

```
$ kubectl get pods -n kube-system
NAME                                          READY   STATUS    RESTARTS   AGE
calico-kube-controllers-866db6d5f7-w9mfq      1/1     Running   0          170m
calico-node-mwgzx                             1/1     Running   0          170m
coredns-bb49df795-lcjvx                       1/1     Running   0          173m
coredns-bb49df795-wqmzb                       1/1     Running   0          173m
etcd-osc01                                    1/1     Running   0          173m
k8web-kubernetes-dashboard-574d4b5798-hszh5   1/1     Running   0          44s
kube-apiserver-osc01                          1/1     Running   0          172m
kube-controller-manager-osc01                 1/1     Running   0          172m
kube-proxy-vprqc                              1/1     Running   0          173m
kube-scheduler-osc01                          1/1     Running   0          173m
tiller-deploy-66478cb847-79hmq                1/1     Running   0          2m24s
```

Check helm charts that we deployed

```
$ helm list
NAME    REVISION        UPDATED                         ---
k8web   1               Mon Sep 30 22:21:01 2019        ---

--- STATUS          CHART                           APP VERSION     NAMESPACE
--- DEPLOYED        kubernetes-dashboard-1.10.0     1.10.1          kube-system
```

Check service names for the dashboard

```
$ kubectl get svc -n kube-system
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   ---
dashboard     ClusterIP   1.4.40.19   <none>        ---
kube-dns      ClusterIP   10.96.0.10     <none>        ---
tiller-deploy ClusterIP   10.98.111.98   <none>        ---

--- PORT(S)         AGE
--- 443/TCP         2m56s
--- 53/UDP,53/TCP   176m
--- 44134/TCP       31m
```

We will patch the dashboard service from CluserIP to NodePort so that we could run the dashboard using the node IP address.

```ssh
kubectl -n kube-system patch svc dashboard --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
```
Traditionally you can edit it .  

```ssh
$ kubectl -n kube-system edit  svc  dashboard
```
You should see `yaml` representation of the service. Change `type: ClusterIP` to `type: NodePort` and save file. If it's already changed go to next step.	Check Kubernetes version
```yaml	
# Please edit the object below. Lines beginning with a '#' will be ignored,	
# and an empty file will abort the edit. If an error occurs while saving this file will be	
# reopened with the relevant failures.	
#	
apiVersion: v1	
...	
  name: kubernetes-dashboard	
  namespace: kube-system	
  resourceVersion: "343478"	
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard-head	
  uid: 8e48f478-993d-11e7-87e0-901b0e532516	
spec:	
  clusterIP: 10.100.124.90	
  externalTrafficPolicy: Cluster	
  ports:	
  - port: 443	
    protocol: TCP	
    targetPort: 8443	
  selector:	
    k8s-app: kubernetes-dashboard	
  sessionAffinity: None	
  type: ClusterIP	
status:	
  loadBalancer: {}	
```	

Next we need to check port on which Dashboard was exposed.	
```sh	
$ kubectl -n kube-system get service dashboard	
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE	
dashboard   1.4.40.19   <nodes>       443:31707/TCP   21h
```

### Run Kubernetes dashboard

Check the internal DNS server

```
$ kubectl exec -it busybox -- cat /etc/resolv.conf

nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local servicemesh.local
options ndots:5
```

Internal service name resolution.

```
$ kubectl exec -it busybox -- nslookup kube-dns.kube-system.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system.svc.cluster.local
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

$ kubectl exec -it busybox -- nslookup hostnames.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hostnames.default.svc.cluster.local
Address 1: 10.98.229.90 hostnames.default.svc.cluster.local
```

Edit VM's `/etc/resolv.conf` to add Kubernetes DNS server

```
sudo vi /etc/resolv.conf
```

Add the following two lines for name resolution of Kubernetes services and save file.

```
search cluster.local
nameserver 10.96.0.10
```

### Get authentication token

If you need to access Kubernetes environment remotely, create a `~/.kube` directory on your client machine and then scp the `~/.kube/config` file from the Kubernetes master to your `~/.kube` directory.

Run this on the Kubernetes master node

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin | awk '{print $1}')
```

Output:

```
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin | awk '{print $1}')
Name:         admin-token-2f4z8
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: 81b744c4-ab0b-11e9-9823-00505632f6a0

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi0yZjR6OCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjgxYjc0NGM0LWFiMGItMTFlOS05ODIzLTAwNTA1NjMyZjZhMCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbiJ9.iaWllI4XHQ9UQQHwXQRaafW7pSD6EpNJ_rEaFqkd5qwedxgJodD9MJ90ujlZx4UtvUt2rTURHsJR-qdbFoUEVbE3CcrfwGkngYFrnU6xjwO3KydndyhLb6v6DKdUH3uQdMnu4V1RVYBCq2Q1bOsejsgNUIxJw1R8N7eUpIte64qUfGYtrFT_NBTnA9nEZPfPAiSlBBXbC0ZSBKXzqOD4veCXsqlc0yy5oXHOoMjROm-Uhv4Oh0gTwdpb-at8Y0p9mPjIy9IQuzSo3Pg5hDKMex4Pwm8WLus4wAaS4mZKu2PI3O2-hhep3GlyvuVH8pOiXQ4p1TI5c0qdDs2rQRs4ow
```

Highlight the authentication token from your screen, right click to copy to the clipboard.

Find out the node port for the `dashboard` service.

```
$ kubectl get svc -n kube-system

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard       NodePort    10.102.12.203   <none>        443:31869/TCP   2m7s
kube-dns        ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   7m34s
tiller-deploy   ClusterIP   10.109.36.64    <none>        44134/TCP       3m13s
```

Doubleclick Google Chrome from the desktop of the VM and run https://localhost:31869 and change the port number as per your output.

Click `Token` and paste the token from the clipboard (Right click and paste).

You have Kubernetes 1.15.6 single node environment ready for you now. 

The following are optional and are not recommended. Skip to [this](#power-down-vm).

## Check if kube-proxy is OK

There must be two entries for the hostnames

```
sudo iptables-save | grep hostnames

-A KUBE-SERVICES ! -s 10.142.0.0/16 -d 10.98.229.90/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.98.229.90/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
[vikram@istio04 ~]$ sudo iptables-save | grep hostnames
-A KUBE-SERVICES ! -s 10.142.0.0/16 -d 10.98.229.90/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.98.229.90/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
```


## Install Metrics server (Optional)

Metrics server is required if we need to run `kubectl top` commands to show the metrics.

```
helm install stable/metrics-server --name metrics --namespace kube-system --set fullnameOverride="metrics" --set args="{--logtostderr,--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP\,ExternalIP\,Hostname}"
```

Make sure that the `v1beta1.metrics.k8s.io` service is available

```
$ kubectl get apiservice v1beta1.metrics.k8s.io
NAME                     SERVICE               AVAILABLE   AGE
v1beta1.metrics.k8s.io   kube-system/metrics   True        13m
```

If the service shows `FailedDiscoveryCheck` or `MissingEndpoints`, it might be the firewall issue. Make sure that `https` is enabled through the firewall.

If the AVAILLABLE shows `False (MissingEndpoints)`, wait for the end points to become available. Try above command again and make sure that the AVAILABLE shows `True` for the api service `v1beta1.metrics.k8s.io`.

Run the following.
```
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
```
Wait for a few minutes, `kubectl top nodes` and `kubectl top pods -A` should show output.



## Install VMware Octant (Optional)

VMware provides [Octant](https://github.com/vmware/octant) an alternative to Kubernetes dashboard.

You can install `Octant` on your Windows, MacBook, Linux and it is a simple to use an alternative to using Kubernetes dashboard. Refer to [https://github.com/vmware/octant](https://github.com/vmware/octant) for details to install Octant.

## Install Prometheus and Grafana (Optional)

This is optional if we do not have enough resources in the VM to deploy additional charts. 

```
helm install stable/prometheus-operator --namespace monitoring --name mon

Note: add --tls above if using secure helm
```

Check monitoring pods

```
$ kubectl -n monitoring get pods
NAME                                     READY   STATUS    RESTARTS   AGE
alertmanager-mon-alertmanager-0          2/2     Running   0          28s
mon-grafana-75954bf666-jgnkd             2/2     Running   0          33s
mon-kube-state-metrics-ff5d6c45b-s68np   1/1     Running   0          33s
mon-operator-6b95cf776f-tqdp8            1/1     Running   0          33s
mon-prometheus-node-exporter-9mdhr       1/1     Running   0          33s
prometheus-mon-prometheus-0              3/3     Running   1          18s
```

Check Services

```ssh
$ kubectl -n monitoring get svc
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   --- PORT(S)             AGE
alertmanager-operated                  ClusterIP   None             <none>        --- 9093/TCP,6783/TCP   19s
mon-grafana                            ClusterIP   10.98.241.51     <none>        --- 80/TCP              23s
mon-kube-state-metrics                 ClusterIP   10.111.186.181   <none>        --- 8080/TCP            23s
mon-prometheus-node-exporter           ClusterIP   10.108.189.227   <none>        --- 9100/TCP            23s
mon-prometheus-operator-alertmanager   ClusterIP   10.106.154.135   <none>        --- 9093/TCP            23s
mon-prometheus-operator-operator       ClusterIP   10.110.132.10    <none>        --- 8080/TCP            23s
mon-prometheus-operator-prometheus     ClusterIP   10.106.118.107   <none>        --- 9090/TCP            23s
prometheus-operated                    ClusterIP   None             <none>        --- 9090/TCP            9s
```

The grafana UI can be opened using: `http://10.98.241.51` for service `mon-grafana`. The IP address will be different in your case.

A node port can also be configured for the `mon-grafana` to use the local IP address of the VM instead of using cluster IP address.

```
# kubectl get svc -n monitoring mon-grafana
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
mon-grafana   ClusterIP   10.105.49.113   <none>        80/TCP    95s
```

Edit the service by running `kubectl edit svc -n monitoring mon-grafana` and change `type` from `ClusterIP` to `NodePort`.

Find out the `NodePort` for the `mon-grafana` service.

```
# kubectl get svc -n monitoring mon-grafana
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
mon-grafana   NodePort   10.105.49.113   <none>        80:32620/TCP   3m15s
```

The grafana UI can be opened through http://localhost:32620 and this node port will be different in your case.

The default user id is `admin` and the password is `prom-operator`. This can be seen through `kubectl -n monitoring get secret mon-grafana -o yaml` and then run `base64 -d` againgst the encoded value for `admin-user` and `admin-password` secret.

You can also open Prometheus UI either by NodePort method as descibed above or by using `kubectl port-forward`

Open a command line window to proxy the Prometheus pod's port to the localhost

First terminal
```
kubectl port-forward -n monitoring prometheus-mon-prometheus-operator-prometheus-0 9090
```

Open `http://localhost:9090` to open the Prometheus UI and `http://localhost:9090/alerts` for alerts.

### Delete prometheus 

If you need to free-up resources from the VM, delete prometheus using the following clean-up procedure.

```
helm delete mon --purge
helm delete ns monitoring
kubectl -n kube-system delete crd \
           alertmanagers.monitoring.coreos.com \
           podmonitors.monitoring.coreos.com \
           prometheuses.monitoring.coreos.com \
           prometheusrules.monitoring.coreos.com \
           servicemonitors.monitoring.coreos.com

Note: add --tls above if using secure helm
```

## Uninstall Kubernetes and Docker

In case Kuberenetes needs to be uninstalled.

Find out node name using `kubectl get nodes`

```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
```

Remove kubeadm

```
sudo systemctl stop kubelet
sudo kubeadm reset
sudo iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
sudo yum -y remove kubeadm kubectl kubelet kubernetes-cni kube*

rm -fr ~/.kube
```

Remove docker and images
```
sudo su -
docker rm -f $(docker ps -qa)
docker volume rm $(docker volume ls -q)
docker rmi $(docker images -q)
systemctl stop docker
rm -fr /var/lib/docker/* 
yum -y remove docker-ce docker-ce-cli
cleanupdirs="/var/lib/etcd /etc/kubernetes /etc/cni /opt/cni /var/lib/cni /var/run/calico /var/lib/kubelet"
for dir in $cleanupdirs; do
  echo "Removing $dir"
  rm -rf $dir
done
```

## Power down VM

Click `Player` > `Power` > `Shutdown Guest`.

 It is highly recommended that you take a backup of the directory after installing Kubernetes environment. You can restore the VM from the backup to start again, should you need it.

The files in the directory may show as:

The output shown using `git bash` running in Windows.

```
$ ls -lh
total 7.3G
-rw-r--r-- 1 vikram 197609 2.1G Jul 21 09:44 dockerbackend.vmdk
-rw-r--r-- 1 vikram 197609 8.5K Jul 21 09:44 kube01.nvram
-rw-r--r-- 1 vikram 197609    0 Jul 20 16:34 kube01.vmsd
-rw-r--r-- 1 vikram 197609 3.5K Jul 21 09:44 kube01.vmx
-rw-r--r-- 1 vikram 197609  261 Jul 21 08:58 kube01.vmxf
-rw-r--r-- 1 vikram 197609 5.2G Jul 21 09:44 osdisk.vmdk
-rw-r--r-- 1 vikram 197609 277K Jul 21 09:44 vmware.log
```

Copy above directory to your backup drive for use it later.

## Power up VM

Locate `kube01.vmx` and right click to open it either using `VMware Player` or `VMware WorkStation`.

Open `Terminal` and run `kubectl get pods -A` and wait for all pods to be ready and in `Running` status.

## Conclusion

This is a pretty basic Kubernetes cluster just by using a single VM - which is good for learning purposes. In reality, we should use a Kubernetes distribution built by a provider such as RedHat OpenShift or IBM Cloud Private or use public cloud provider such as AWS, GKE, Azure or many others.

## Ascinema Cast

Click the link below to open the [Asciinema](https://ascinema.org) player to playback the commands that were captured while building the Kubernetes environment.

[![asciicast](https://asciinema.org/a/271731.svg)](https://asciinema.org/a/271731)
