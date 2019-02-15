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

## NodePort

This way of accessing Dashboard is only recommended for development environments in a single node setup. 

Edit `kubernetes-dashboard` service.
```sh
$ kubectl -n kube-system edit service kubernetes-dashboard
```

You should see `yaml` representation of the service. Change `type: ClusterIP` to `type: NodePort` and save file. If it's already changed go to next step.
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
$ kubectl -n kube-system get service kubernetes-dashboard
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   10.100.124.90   <nodes>       443:31707/TCP   21h
```

Dashboard has been exposed on port `31707 (HTTPS)`. Now you can access it from your browser at: `https://<master-ip>:31707`. `master-ip` can be found by executing `kubectl cluster-info`. Usually it is either `127.0.0.1` or IP of your machine, assuming that your cluster is running directly on the machine, on which these commands are executed.

In case you are trying to expose Dashboard using NodePort on a multi-node cluster, then you have to find out IP of the node on which Dashboard is running to access it. Instead of accessing `https://<master-ip>:<nodePort>` you should access `https://<node-ip>:<nodePort>`.

## API Server

In case Kubernetes API server is exposed and accessible from outside you can directly access dashboard at:
`https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

**Note:** This way of accessing Dashboard is only possible if you choose to install your user certificates in the browser. In example certificates used by kubeconfig file to contact API Server can be used.

## Ingress

Dashboard can be also exposed using Ingress resource. For more information check: https://kubernetes.io/docs/concepts/services-networking/ingress.

# Dashboard usage

Using parameter --address  , --accept-hosts ,let everybody access.


```bash

kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'

```

## NodePort

we can use NodePort to explode , according documents not recommentd in production.

```bash

kubectl -n kube-system edit service kubernetes-dashboard

```

```yaml

apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kube-system"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
  creationTimestamp: 2018-05-01T07:23:41Z
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "1750"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard
  uid: 9329577a-4d10-11e8-a548-00155d000529
spec:
  clusterIP: 10.103.5.139
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

We need to replace *type: ClusterIP*   by *ype: NodePort* .

```bash

kubectl -n kube-system get service kubernetes-dashboard

NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.103.5.139   <none>        443:31795/TCP   4h
```

如上所示，Dashboard已經在31795端口上公開，現在可以在外部使用https://<cluster-ip>:31795進行訪問。需要注意的是，在多節點的集群中，必須找到運行Dashboard節點的IP來訪問，而不是Master節點的IP，在本文的示例，我部署了兩台服務器，MatserIP為192.168.0.8，ClusterIP為192.168 .0.10

Dashboard Certificate management

https://github.com/kubernetes/dashboard/wiki/Certificate-management




1. Create account named "admin-user" under namspece "kube-system".

```yaml

# admin-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

```

Executing command " kubectl create " :

```bash

kubectl create -f admin-user.yaml

```

2. Binding Role

Under default situaltion, while  kubeadm creating cluster ,and creating admin role. we can immedeately use it .

```yaml
 
# admin-user-role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

```

Executing command " kubectl create " :

```bash

kubectl create -f  admin-user-role-binding.yaml

```

3. Getting Token

```bash

kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')


ame:         admin-user-token-qrj82
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=6cd60673-4d13-11e8-a548-00155d000529

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXFyajgyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2Y2Q2MDY3My00ZDEzLTExZTgtYTU0OC0wMDE1NWQwMDA1MjkiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.C5mjsa2uqJwjscWQ9x4mEsWALUTJu3OSfLYecqpS1niYXxp328mgx0t-QY8A7GQvAr5fWoIhhC_NOHkSkn2ubn0U22VGh2msU6zAbz9sZZ7BMXG4DLMq3AaXTXY8LzS3PQyEOCaLieyEDe-tuTZz4pbqoZQJ6V6zaKJtE9u6-zMBC2_iFujBwhBViaAP9KBbE5WfREEc0SQR9siN8W8gLSc8ZL4snndv527Pe9SxojpDGw6qP_8R-i51bP2nZGlpPadEPXj-lQqz4g5pgGziQqnsInSMpctJmHbfAh7s9lIMoBFW7GVE8AQNSoLHuuevbLArJ7sHriQtDB76_j4fmA
ca.crt:     1025 bytes
namespace:  11 bytes

```


