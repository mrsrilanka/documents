A kubernetes cluster is consists of bunch of nodes/servers managed thru etcd. A server crash could happen at anytime. So what would you do when the cluster nodes reboot and shows up as "not ready" status to re-join the kubernetes cluster? I searched google and other resources to find a good document to use but I could not and thought of writing this down for the k8s fambam to use. 

```
[rainer@k8s-master ~]$ sudo kubectl get nodes -o wide
NAME         STATUS     ROLES                  AGE     VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
k8s-master   Ready      control-plane,master   7d22h   v1.21.0   192.168.100.212   <none>        CentOS Linux 7 (Core)   3.10.0-1160.21.1.el7.x86_64   docker://1.13.1
worker1      NotReady   <none>                 7d22h   v1.21.0   192.168.100.143   <none>        CentOS Linux 7 (Core)   3.10.0-1160.21.1.el7.x86_64   docker://1.13.1
worker2      NotReady   <none>                 7d22h   v1.21.0   192.168.100.166   <none>        CentOS Linux 7 (Core)   3.10.0-1160.21.1.el7.x86_64   docker://1.13.1
[rainer@k8s-master ~]$
```

After the reboot first login to your master node and check for the  kubectl and see if it can see the other nodes. 

[rainer@k8s-master ~]$ sudo kubectl get nodes
The connection to the server 192.168.100.212:6443 was refused - did you specify the right host or port?
[rainer@k8s-master ~]$ 

This could be several reasons
1. Turn off swap

```
sudo swapoff -a
```

2. Check for the docker process 

```
[rainer@k8s-master ~]$ sudo docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
[rainer@k8s-master ~]$ 
```

Restart docker process

```
systemctl restart docker
```

and verify the docker process

```
docker ps
```
3. export the kubeconfig 

```
[rainer@k8s-master ~]$ export KUBECONFIG=/etc/kubernetes/kubelet.conf 
[rainer@k8s-master ~]$ sudo kubectl get nodes
[sudo] password for rainer: 
NAME         STATUS     ROLES                  AGE     VERSION
k8s-master   Ready      control-plane,master   7d22h   v1.21.0
worker1      NotReady   <none>                 7d22h   v1.21.0
worker2      NotReady   <none>                 7d22h   v1.21.0
[rainer@k8s-master ~]$
```
Now you can see the nodes are visible to the master in the ocntrol plain. At this point you can restart docker and swapoff the nodes and check if the status changes from the Master but if it doesnt then execute the following. 

recreate the token and certs  

```
[rainer@k8s-master ~]$ sudo kubeadm token create
ub0shw.x3q5psfieb1g0z4t
[rainer@k8s-master ~]$ sudo kubeadm token list
TOKEN                     TTL         EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
ub0shw.x3q5psfieb1g0z4t   23h         2021-04-17T21:02:19-04:00   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
[rainer@k8s-master ~]$ openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -pubkey | openssl rsa -pubin -outform DER 2>/dev/null | sha256sum | cut -d' ' -f1
0d4f4e1aeb369cf036e1bc868ef7b499a45e443400e78385138ff19b2035c3f5
[rainer@k8s-master ~]$ 
```

Login to the worker/slave/minion nodes and make sure the swapoff -a executed and docker process is running. Execute the follwoing according to your own token and cert you created in your cluster. 

```
[rainer@worker2 ~]$ sudo kubeadm join 192.168.100.212:6443 --token ub0shw.x3q5psfieb1g0z4t --discovery-token-ca-cert-hash 0d4f4e1aeb369cf036e1bc868ef7b499a45e443400e78385138ff19b2035c3f5
[preflight] Running pre-flight checks
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
	[ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
[rainer@worker2 ~]$ 
```

Now you can check the master and the node status 

```
[rainer@k8s-master ~]$ sudo kubectl get nodes -o wide
NAME         STATUS     ROLES                  AGE     VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
k8s-master   Ready      control-plane,master   7d22h   v1.21.0   192.168.100.212   <none>        CentOS Linux 7 (Core)   3.10.0-1160.21.1.el7.x86_64   docker://1.13.1
worker1      NotReady   <none>                 7d22h   v1.21.0   192.168.100.143   <none>        CentOS Linux 7 (Core)   3.10.0-1160.21.1.el7.x86_64   docker://1.13.1
worker2      Ready      <none>                 7d22h   v1.21.0   192.168.100.166   <none>        CentOS Linux 7 (Core)   3.10.0-1160.21.1.el7.x86_64   docker://1.13.1
[rainer@k8s-master ~]$ 
```

And you are set and continue the same to add the rest of the nodes as needed. 
