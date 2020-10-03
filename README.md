
# Objectives

* Use cheap Home Lab based on Raspberry in order to have access any time.
* Get more hindsight on Kubernetees with hands-on.
* Not be lazy to Destroy/Create this lab because I want to try something new on K8s ==> Full Automation through ansible 
* Test a few CNIs and get some opinion on Use Case applicability

# References

Followed mostly this tutorial https://linuxconfig.org/how-to-install-kubernetes-on-ubuntu-20-04-focal-fossa-linux

# Lab Details:

So far, I have two Raspberry on my home network:

    Raspberry pi4b / 4gbps memory ==> kubernetees master
    Raspberry pi2 / 1 Gbps memory ==> kubernetees worker

The plan is to build a kubernetees cluster. Static IP addresses are allocated on my Home DHCP server to both devices: 192.168.1.150 and .151

``` 
          raspberry pi 4b                                                  raspberry pi 2          
+--------------------------------+                               +--------------------------------+
|                                |                               |                                |
|                                |                               |                                |
|          kube-master           |                               |          kubernetees           |
|                                |                               |             worker             |
|                                |                               |                                |
|                                |                               |                                |
+--------------------------------+                               +--------------------------------+
                 | .150                                                              | .150           
                 |                         192.168.1.0/24                            |                
                 |                                                                   |                
        ---------|--------------------------------|----------------------------------|------------    
                                                  |                                                   
                                                  |                                                   
                                                  |  .1                                               
                                      +----------------------------------------------+                           
                                      |                 Home Router                  |                           
                                      |                                              |                           
                                      |                                              |                           
                                      |        DHCP server with static entries       |                           
                                      |        for .150 and .151 as per drawing      |                           
                                      |                                              |                           
                                      +----------------------------------------------+                           

```
# Installation preparation

* Configure static DHCP entries on hosts (here I have .150 and .151)
* Download and flash appropriate images for (64 preferred when applicable depending on the model): https://ubuntu.com/download/raspberry-pi. I am using rufus to write the images on SD.
* I am using an offline ansible installer on my laptop so I can control the installation: https://github.com/robric/dockerwindows this container image is ready to use with sshd/ansible/git preinstalled.

# Installation steps

* STEP 1: Clone this repo
```
[root@dd1ae25265fa drive]# git clone https://github.com/robric/rpi
```
* STEP 2: Edit the inventory.ini file with the appropriate IP addresses and identifiers
```
[root@dd1ae25265fa drive]# cd rpi
[root@dd1ae25265fa rpi]# ls
cheat.yaml  inventory.ini  playbook.yml  tester.yaml
[root@dd1ae25265fa rpi]# vi inventory.ini 
[masters]
192.168.1.150  ansible_user="ubuntu" ansible_ssh_pass="ubuntu123!" id=1  <============= Change the IP of the master

[workers]
192.168.1.151  ansible_user="ubuntu" ansible_ssh_pass="ubuntu123!" id=1 <============= Change the IP of the worker - If you have several workers, make sure that id is unique (this is used for auto-naming)

[k8s_hosts:children]
masters
workers

[local]
localhost ansible_ssh_user=root ansible_ssh_pass=root123 ansible_ssh_common_args='-o StrictHostKeyChecking=no'
### Type [ESC]wq
```
* STEP 3: Clean ssh credentials if needed from previous installations for the hosts 
```
[root@dd1ae25265fa rpi]# ssh-keygen -R 192.168.1.150
# Host 192.168.1.150 found: line 3
/root/.ssh/known_hosts updated.
Original contents retained as /root/.ssh/known_hosts.old
[root@dd1ae25265fa rpi]#
```
* STEP 3: Launch the playbook
```
[root@dd1ae25265fa rpi]# ansible-playbook -i inventory.ini playbook.yml -vv
ansible-playbook 2.9.13
  config file = /etc/ansible/ansible.cfg
  [...]
```
* STEP 4: Check the status of the cluster
```
ubuntu@master-1:~$ kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-f9fd979d6-ndppl                   1/1     Running   0          18h
kube-system   coredns-f9fd979d6-q5qq2                   1/1     Running   0          18h
kube-system   etcd-master-1                             1/1     Running   0          18h
kube-system   kube-apiserver-master-1                   1/1     Running   0          18h
kube-system   kube-controller-manager-master-1          1/1     Running   1          18h
kube-system   kube-proxy-zq9ql                          1/1     Running   0          18h
kube-system   kube-scheduler-master-1                   1/1     Running   2          18h
ubuntu@master-1:~$ 

```
coredns should fail due to the lack of CNI

# CNI Installation

At this stage the cluster is not fully functionaly because it misses a CNI - so you need to install one.
Contrail ARM binaries and CRD/operator install not yet available at time of writing... but that is just a matter of time :-) 

## Calico installation 

The basic installation of calico is super simple

```
ubuntu@master-1:~$ curl https://docs.projectcalico.org/manifests/calico.yaml -O
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  182k  100  182k    0     0   769k      0 --:--:-- --:--:-- --:--:--  772k
ubuntu@master-1:~$ kubectl apply -f calico.yaml
configmap/calico-config created
customresourcedefinition.apiextensio
[...]
```

After a few minutes pods are running
```
ubuntu@master-1:~$ kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-c9784d67d-wfsjv   1/1     Running   0          4m10s
kube-system   calico-node-zjh6c                         1/1     Running   0          4m10s
kube-system   coredns-f9fd979d6-ndppl                   1/1     Running   0          7h11m
kube-system   coredns-f9fd979d6-q5qq2                   1/1     Running   0          7h11m
kube-system   etcd-master-1                             1/1     Running   0          7h11m
kube-system   kube-apiserver-master-1                   1/1     Running   0          7h11m
kube-system   kube-controller-manager-master-1          1/1     Running   0          7h11m
kube-system   kube-proxy-zq9ql                          1/1     Running   0          7h11m
kube-system   kube-scheduler-master-1                   1/1     Running   1          7h11m
ubuntu@master-1:~$
```

