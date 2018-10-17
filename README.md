# Kubernetes Note
## 1. 架設說明
### Environment
Machine：HPE 380<br/>
OS：VMWARE ESXi<br/>
Virtual OS：Ubuntu 16.04.5 LTS<br/>
Kubernetes Version: v1.11.2

### Installation Step by Step
1. Install `kubelet`, `kubeadm`, `kubectl`
   ```
   sudo -i
   ```
    ```
   sudo apt-get update && apt-get upgrade
    ```
    ```
    apt-get install -y apt-transport-https curl docker.io
    ```
    ```
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    ```
    ```
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    ```
    ```
    deb http://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    ```
    ```
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl
    sysctl net.bridge.bridge-nf-call-iptables=1
    ```
2. ```
   swapoff -a
   docker
   kubeadm init --pod-network-cidr=10.244.0.0/16
   exit
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf  $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   export KUBECONFIG=/etc/kubernetes/admin.conf
   ```
Ref:<br/>
[install kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)<br/>
[create cluster with kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)<br/>
[安裝kubernetes](https://tachingchen.com/tw/blog/kubernetes-installation-with-kubeadm/#%E7%A7%BB%E9%99%A4-kubernetes)
## 2. 操作說明
### Attach to a Pod
```
kubectl exec -it [pod's NAME] -- /bin/bash
```
Ref:<br/>
[Get a Shell to a Running Container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/)
### Logging 
>  *On machines with systemd, the kubelet and container runtime write to journald.*

View journald:
```
journalctl | grep kube
```
Logging file location:

    /var/log/containers

Ref:<br/>
[Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)<br/>
[kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)<br/>

### Join a Node
  1. Generate a token on the master
      ```
      kubeadm token create
      ```
  2. Get the `token-ca-cert-hash` on the master:
     ```
     openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
  3. (Optional) Print join cmd from master
      ```
      kubeadm token create --print-join-command
      ```
  4. On the slave machine, change to root:
      ```
      kubeadm join --token [token] [master-ip]:[master-port] --discovery-token-ca-cert-hash sha256:[token-ca-cert-hash]
      ```
  #### Warning: Please make sure that the version of kubeadm of master and worker are matched; otherwise you will fail to join a worker
Ref:<br/>
[Join Nodes](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#join-nodes)<br/>
[kubeadm join](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)<br/>
[kubeadm init does not configure RBAC for configmaps correctly](https://github.com/kubernetes/kubeadm/issues/907)
### Stop
* Stop a node (empty a node and make the node unschedulable)
    ```
    kubectl drain [node's NAME]--delete-local-data --force --ignore-daemonsets
    ```
* Restart a node (make the node schedulable)
    ```
    kubectl uncordon [node's NAME]
    ```
Ref:<br/>
[Maintenance on a Node](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/#maintenance-on-a-node)<br/>
[Drain svg](https://kubernetes.io/images/docs/kubectl_drain.svg)
### Delete

* Delete a service
    ```
    kubectl delete service [pod's or deployment's NAME]
    ```
* Delete a deployment
    ```
    kubectl delete deployment [deployment's NAME]
    ```
* Drain a node (empty a node and make the node unschedulable)
    ```
    kubectl drain [node's NAME]--delete-local-data --force --ignore-daemonsets
    ```
* Delete a pod
    ```
    kubectl delete pod [pod' NAME]
    ```
* Delete a node
    ```
    kubectl delete node [node's NAME]
    ```
Once you have done the things above, you can init the whole cluster by:
    `kubeadm reset` or `kubeadm init`
    
Ref:<br/>
[Tear down the cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#tear-down)<br/>
[Drain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain)<br/>
[Safely Drain node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)<br/>
[kubernetes 节点维护 cordon, drain, uncordon](https://blog.csdn.net/stonexmx/article/details/73543185)<br/>
[Does restarting kubelet stop all nodes?](https://stackoverflow.com/questions/50912537/does-restarting-kubelet-stop-all-nodes)<br/>
[Kubernetes集群之清除集群](https://o-my-chenjian.com/2017/05/11/Clear-The-Cluster-Of-K8s/)



## 3. 使用說明
### Easy Creating of a Resource Step by Step
1. Build your docker file
2. Create a deployment

    > *A Kubernetes Deployment checks on the health of your Pod and restarts the Pod’s Container if it terminates.* 

    ```
    kubectl run [deployment's NAME] --image=[docker image] --port=8080 --image-pull-policy=Never
    ```
    `--port=8080` for exposing 8080 port of the container
    `--image-pull-policy=Never` for only pull the container from local registery
    
    View the deployments:
    ```
    kubectl get deployments
    ```
    Output:
    ```
    NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    [deployment's NAME] int       int       int          int         int(m/h/d)
    ```
    View the Pod:
    ```
    kubectl get pods
    ```
    Output:
    ```
    NAME                             READY    STATUS    RESTARTS   AGE
    [deployment's NAME]+hashString   int/int  string    int        int(m/h/d)
    ```
3. Create a service
    > *To make the your container accessible from outside the Kubernetes virtual network, you have to expose the Pod as a Kubernetes Service.*

    Expose a pod to the public internet:
    ```
    kubectl expose deployment [pod's or deployment's NAME] --type=NodePort
    ```
    The `--type=NodePort` flag indicates that you want to expose your Service outside of the cluster.(For more info, please visit [kubernetes service](https://kubernetes.io/docs/concepts/services-networking/service/))
    
    View the service:
    ```
    kubectl get services
    ```
    Output:
    ```
    NAME                          TYPE             CLUSTER-IP          EXTERNAL-IP   PORT(S)                                      AGE
    [pod's or deployment's NAME]  NodePort         [IP inside cluster] <pending>     8080:[port for access outside cluster]/TCP   int(m/h/d)
    kubernetes                    ClusterIP        [IP inside cluster] <none>        443/TCP                                      int(m/h/d)
    ```
4. Update container(optional)
    * Rebuild your docker image
    * 
        ```
        kubectl set image deployment/[deployment's NAME] [pod' NAME]=[new docker image]
        ```
Ref:<br/>
[Hello Minikube](https://kubernetes.io/docs/tutorials/hello-minikube/)<br/>
[kubectl overview](https://kubernetes.io/docs/user-guide/kubectl-overview/)<br/>
[kubernetes service](https://kubernetes.io/docs/concepts/services-networking/service/)

### Create a Resource
* `kubectl run [deployment's NAME] --image=[docker image]`
    `kubectl create deployment [deployment's NAME] --image=[docker image]`
Create a deployment object of the docker image
(If your docker image has a version tag, you must specify it; otherwise kubectl will fail to create a kubernetes resouce with the docker image. e.g. hello-node:**v1**)
    * useful arguments
    * `--port=[port]`
    * `--image-pull-policy=[boolean]`
* `kubectl create -f [config.yaml or config.json]`
Create resources according to the config

Ref:<br/>
[Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/)<br/>
[run vs apply and create](https://stackoverflow.com/questions/48015637/kubernetes-kubectl-run-vs-create-and-apply)<br/>
[pod vs deployment](https://stackoverflow.com/questions/41325087/in-kubernetes-what-is-the-difference-between-a-pod-and-a-deployment)

### Create a Service
* `kubectl expose deployment [pod's or deployment's NAME] --type=NodePort`
`--type=NodePort` maps a port of your master node to the serivce. Once you desire to run a public server inside the cluster, you should add this flag to enable outside connection.


Ref:<br/>
[Service](https://kubernetes.io/docs/concepts/services-networking/service/)
[Expose](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#expose)
### Commands for Printing Resources In the Cluster
#### Get
get the brief info of some resources
* `kubectl get nodes`
Brief info of all nodes in the cluster
* `kubectl get pods`
Brief info of all pods in the cluster
* `kubectl get pods -o wide`
Breif info of all pods in the cluster with more info (including the node's name to which each pod belongs)
(`-o` stands for `--output`, ouput format)
* `kubectl get pod [pod's NAME]`
Brief info of the pod
* `kubectl get services`
Brief info of all services in the cluster
* `kubectl get deployments`
Brief info of all deployments in the cluster
* `kubectl get deployment [deployment's NAME]`
Brief info of the deployment in the cluster
#### Describe
Get the detailed info of some resources
Usage: simply replace `get` in the above commands by `describe`.
e.g. `kubectl describe pods`, `kubectl describe pod [pod's name]`

Ref:
[kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

## 4. Advanced Reading
[Understanding Kubernetes Pods](https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727)<br/>
[A few things I've learned about Kubernetes](https://jvns.ca/blog/2017/06/04/learning-about-kubernetes/)<br/>
[Kubernetes mirror pod & static pod](http://www.rhca.me/?p=35)<br/>
[how does kube-apiserver restart after editing /etc/kubernetes/manifests/kube-apiserver.yaml](https://stackoverflow.com/questions/50007654/how-does-kube-apiserver-restart-after-editing-etc-kubernetes-manifests-kube-api)<br/>
[认证Kubernetes管理员（CKA](https://jimmysong.io/kubernetes-handbook/appendix/about-cka-candidate.html)<br/>
[Kubernetes 部署失败的 10 个最普遍原因](http://dockone.io/article/2247)<br/>
[kubernetes 简介： kubelet 组件功能](https://blog.csdn.net/liukuan73/article/details/54971616)<br/>
[Kubernetes核心原理（四）之Kubelet](https://blog.csdn.net/huwh_/article/details/77922293)<br/>
[Kubernetes 30天學習筆記](https://ithelp.ithome.com.tw/users/20103753/ironman/1590)
