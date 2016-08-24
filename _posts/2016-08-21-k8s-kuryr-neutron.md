---
layout: post
author: vikasc
title: Kubernetes multi-node installation (Baremetal) with Kuryr
date: 2016-08-20 03:25:06 -0700
categories:
- SDN
- OpenStack
- Kubernetes
- Kuryr
- Neutron
tags:
- OVS
- OpenStack
- Kuryr
- Kubernetes
- Neutron
---

In this post we will see how to launch a multi-node Kubernetes cluster with
OpenStack Neutron networking via project Kuryr.
If this is the first time that you are hearing words *OpenStack* and *Kubernetes* this
post is not meant for you. If you are unfamiliar with project Kuryr then checkout
[OpenStack Tokyo Kuryr Introduction Talk](https://www.openstack.org/videos/video/tokyo-2474) before continuing. 

This is going to be a *baremetal installation* where OpenStack nodes and k8s cluster will share the same
underlying hardware/machines. 

I used three machines:

| **Machine**        | **OS**           | **hostname**  |
|:-------------:|:-------------:|:-----:|
| My Laptop      | Fedora24 | fed-node |
| VirtualBox-VM      | Fedora24      |   fed-master |
| Virtualbox-VM | ubuntu16      |    ubuntu-node |

NOTE: There is no specific reason to use ubuntu. One can use fedora as well. Its just that i already had ubuntu vm so just
reused it.

All three machines are connected via a VirtualBox *host-only* network. For the sake of convenience, lets refer to the machines with their hostnames.

## High Level Plan:

1. OpenStack Installation:
  * Controller on *fed-master*
  * Just neutron on *fed-node* and *ubuntu-node*
2. Kubernetes Installation:
  * master on *fed-master*
  * Kubernetes nodes on *fed-node* and *ubuntu-node*
3. Kuryr Installation:
  * *raven* service installation on *fed-master*
  * Kuryr *cni driver* installation on *fed-node* and *fed-master*

## OpenStack Installation

I used [devstack](https://fedoraproject.org/wiki/OpenStack_devstack).

*fed-master* localrc:

~~~
[vikas@fed-master devstack]$ cat localrc 
OFFLINE=No
RECLONE=No
DEST=/opt/src/openstack
DATA_DIR=$DEST/data
LOGFILE=$DATA_DIR/logs/stack.log                                                                                                   
SCREEN_LOGDIR=$DATA_DIR/logs                                                                                                       
VERBOSE=True
HOST_IP_IFACE=enp0s8
PUBLIC_INTERFACE=enp0s8
MYSQL_PASSWORD=pass
SERVICE_TOKEN=pass
SERVICE_PASSWORD=pass
ADMIN_PASSWORD=pass
MULTI_HOST=1
HOST_IP=192.168.56.101
RABBIT_PASSWORD=pass
RABBIT_HOST=fed-master

[vikas@fed-master devstack]$ ifconfig enp0s8
enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.101  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::c6a5:3f7f:7ea0:f402  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:bf:2d:4e  txqueuelen 1000  (Ethernet)
        RX packets 540952  bytes 136898934 (130.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 693326  bytes 881289112 (840.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
~~~

*fed-node* localrc:

~~~
[vikas@fed-node devstack]$ cat localrc
OFFLINE=No
RECLONE=No
HOST_IP=192.168.56.1
DEST=/opt/src/openstack
DATA_DIR=$DEST/data
LOGFILE=$DATA_DIR/logs/stack.log
SCREEN_LOGDIR=$DATA_DIR/logs
VERBOSE=True
HOST_IP_IFACE=enp0s8
PUBLIC_INTERFACE=enp0s8
MYSQL_PASSWORD=pass
SERVICE_TOKEN=pass
SERVICE_PASSWORD=pass
ADMIN_PASSWORD=pass
RABBIT_PASSWORD=pass
SERVICE_HOST=fed-master
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
Q_HOST=$SERVICE_HOST
MULTI_HOST=1

ENABLED_SERVICES=rabbit,neutron,q-agt

[vikas@fed-node devstack]$ ifconfig vboxnet0
vboxnet0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.1  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::800:27ff:fe00:0  prefixlen 64  scopeid 0x20<link>
        ether 0a:00:27:00:00:00  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 107352  bytes 10559283 (10.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
~~~

devstack localrc on *ubuntu-node* is also same as the above one on *fed-node* except HOST_IP set to *ubuntu-node*'s host-only network IP, 192.168.56.102.

You can use any OpenStack installation approach (if dont want to use devstack).
After successful installation *neutron agent-list* should show three OpenvSwitch agents:

~~~
[vikas@fed-master devstack]$ neutron agent-list
+--------------------------------------+--------------------+-------------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host        | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-------------+-------------------+-------+----------------+---------------------------+
| 38d42afc-e5c6-4800-a507-bee1982435fd | Metadata agent     | fed-master  |                   | :-)   | True           | neutron-metadata-agent    |
| 6b4b5571-1537-4964-b295-b9ee61dfbc8f | Open vSwitch agent | fed-master  |                   | :-)   | True           | neutron-openvswitch-agent |
| 827dbf7f-1000-4d98-b6a2-b6840cfb082e | Open vSwitch agent | ubuntu-node |                   | :-)   | True           | neutron-openvswitch-agent |
| 9f4be068-e84a-4039-9c63-e6f45e5fd149 | Open vSwitch agent | fed-node    |                   | :-)   | True           | neutron-openvswitch-agent |
| 9fbc8ab8-0678-413f-beaf-534999f98c2b | DHCP agent         | fed-master  | nova              | :-)   | True           | neutron-dhcp-agent        |
| ff08751a-e00f-436f-a34c-1499aa3d1ab6 | L3 agent           | fed-master  | nova              | :-)   | True           | neutron-l3-agent          |
+--------------------------------------+--------------------+-------------+-------------------+-------+----------------+---------------------------+
~~~

## Kubernetes Installation

I refered [standard community guide](http://kubernetes.io/docs/getting-started-guides/fedora/fedora_manual_config/). Documentation does not seem to be up-to-dated and I had to do few additional changes:

1. To get *kubectl* working, had to modify ~/.kube/config. *server* should be configure with api-server endpoint details.
2. Had to do following steps to get pods created. Details [here](https://github.com/kubernetes/kubernetes/issues/11355#issuecomment-127378691):
  * Generate a signing key:

	```
	$ openssl genrsa -out /tmp/serviceaccount.key 2048
	```

    * Update /etc/kubernetes/apiserver with following line:

	```
	KUBE_API_ARGS="--service_account_key_file=/tmp/serviceaccount.key"
	```
     
    * Update /etc/kubernetes/controller-manager with following line:

	```
	KUBE_CONTROLLER_MANAGER_ARGS="--service_account_private_key_file=/tmp/serviceaccount.key
	```


As a validation step, run few commands on *fed-master*. If all is well output should look like as shown:

~~~
[root@fed-master ~]# kubectl get nodes
NAME          STATUS    AGE
fed-node      Ready     9d
ubuntu-node   Ready     2d
[root@fed-master ~]#
[root@fed-master ~]#
[root@fed-master ~]# kubectl  create -f /home/vikas/Downloads/1ubuntu-nginx.yaml
pod "1-ubuntu.nginx" created
[root@fed-master ~]#
[root@fed-master ~]# kubectl get pods
NAME               READY     STATUS    RESTARTS   AGE
1-ubuntu.nginx     1/1       Running   0          38s
[root@fed-master ~]# 
~~~

At this point cluster is using default bridge networking. In the next section, we are going to replace it with Kuryr networking.

## Kuryr Installation

Kuryr has two parts - [Raven service](https://github.com/openstack/kuryr/blob/master/doc/source/devref/k8s_api_watcher_design.rst) and CNI driver which is based on [CNI Network Specification](https://github.com/containernetworking/cni).

Kuryr supports Python3 only.

### Raven Installation, Configuration and Running

There will be a single instance of Raven service running on Kubernetes master node i.e *fed-master*.

~~~
[root@fed-master ~]# git clone https://github.com/midonet/kuryr/tree/k8s.git
[root@fed-master ~]# cd kuryr
[root@fed-master ~]#
[root@fed-master ~]# pip3 install .
[root@fed-master ~]#
[root@fed-master kuryr]# cat ravenrc 
export SERVICE_USER="admin"
export SERVICE_TENANT_NAME="admin"
export SERVICE_PASSWORD="pass"
export IDENTITY_URL="http://fed-master:35357/v2.0"
export OS_URL="http://fed-master:9696"
export K8S_API="http://fed-master:8080"
export SERVICE_CLUSTER_IP_RANGE="10.254.0.0/16"
[root@fed-master kuryr]#
[root@fed-master kuryr]# source ravenrc
[root@fed-master ~]#
[root@fed-master ~]#raven
~~~

Now Raven is lisentening to all the events that api-server is reporting.

### CNI driver installation and configuration

These steps are to be performed on each of the nodes, *fed-node* and *ubuntu-node*

~~~
[root@fed-master ~]# git clone https://github.com/midonet/kuryr/tree/k8s.git   
[root@fed-master ~]# cd kuryr                                                  
[root@fed-master ~]# pip3 install -r requirements.txt                          
[root@fed-master ~]# pip3 install flask                                                      
[root@fed-master ~]# source ravenrc 
[root@fed-master ~]# python3 setup.py install . 
~~~

Verify that cni driver binary, *kuryr*, got created at expected path:

~~~
[root@fed-node cni]# ls /opt/cni/bin/
kuryr
~~~

Create a network configuration file and directories as shown below:

~~~
[root@fed-node cni]# cat /etc/cni/net.d/kuryr.conf 
{ "cniVersion": "0.1.0", "name": "kuryr", "type": "kuryr"}
[root@fed-node cni]# 
~~~

Now run the *kubelet* with *--network-plugin-dir* and *--network-plugin* options:

~~~
[root@fed-node kuryr]# /usr/bin/kubelet --logtostderr=true --v=4 --api-servers=http://fed-master:8080 --address=0.0.0.0 --network-plugin-dir=/etc/cni/net.d --network-plugin=cni --hostname-override=fed-node --allow-privileged=false
~~~

Please note that same steps are to be repeated on *ubuntu-node* as well.


## All done, now final testing

We will launch two pods one on each node and then will verify connectivity between them.

### On *fed-master*:

~~~
[root@fed-master ~]# kubectl  create -f /home/vikas/Downloads/1ubuntu-nginx.yaml
pod "1-ubuntu.nginx" created
[root@fed-master ~]# kubectl  create -f /home/vikas/Downloads/1fedora-nginx.yaml
pod "1-fedora.nginx" created
[root@fed-master ~]#
[root@fed-master ~]# kubectl get pods

NAME               READY     STATUS    RESTARTS   AGE
1-fedora.nginx     1/1       Running   0          5s
1-ubuntu.nginx     1/1       Running   0          10s

[root@fed-master ~]# cat /home/vikas/Downloads/1ubuntu-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: 1-ubuntu.nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
  nodeSelector:
    name: ubuntu-node-label    
[root@fed-master ~]# 
[root@fed-master ~]# cat /home/vikas/Downloads/1fedora-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: 1-fedora.nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
  nodeSelector:
    name: fed-node-label    
[root@fed-master ~]# 
~~~

Lets verify ports in neutron:

~~~
vikas@fed-master devstack]$ neutron port-list 
+--------------------------------------+------------------+-------------------+---------------------------------------------------------------------------------------------+
| id                                   | name             | mac_address       | fixed_ips                                                                                   |
+--------------------------------------+------------------+-------------------+---------------------------------------------------------------------------------------------+
| 0cb5bf72-265c-4794-b331-1748d15c674b |                  | fa:16:3e:ab:f0:85 | {"subnet_id": "fe9a15cb-a089-4d71-9827-cffaf9462317", "ip_address": "11.0.0.1"}             |
| 5b2599d4-59a4-4723-99cb-30f413fdc2b6 | 1-ubuntu.nginx   | fa:16:3e:ba:94:e3 | {"subnet_id": "854d8eb6-bfd9-40fe-b4e1-50b48186086b", "ip_address": "182.168.0.6"}          |
| 92d2ae48-f154-4340-be3b-f01375c34ba6 |                  | fa:16:3e:a8:01:d5 | {"subnet_id": "854d8eb6-bfd9-40fe-b4e1-50b48186086b", "ip_address": "182.168.0.2"}          |
| 93916904-160b-4f89-a37c-e61c65f70166 |                  | fa:16:3e:00:be:2a | {"subnet_id": "fe9a15cb-a089-4d71-9827-cffaf9462317", "ip_address": "11.0.0.2"}             |
|                                      |                  |                   | {"subnet_id": "1a0a502d-20aa-4340-8c3e-9b2718518c24", "ip_address":                         |
|                                      |                  |                   | "fdd5:cdf3:8b50:0:f816:3eff:fe00:be2a"}                                                     |
| a825c064-5305-4c3e-b098-d5e0d2c8d8da |                  | fa:16:3e:dd:02:f0 | {"subnet_id": "854d8eb6-bfd9-40fe-b4e1-50b48186086b", "ip_address": "182.168.0.1"}          |
| d79b8ada-b7ea-4d39-b85c-0d249aa7e256 | 1-fedora.nginx   | fa:16:3e:fc:8f:a1 | {"subnet_id": "854d8eb6-bfd9-40fe-b4e1-50b48186086b", "ip_address": "182.168.0.9"}          |
| d8d0e84c-eb4e-47c6-b579-c68ab08f27b9 |                  | fa:16:3e:fd:c1:d9 | {"subnet_id": "74c44f1f-ff7d-4737-b6ed-cfbd516db6e1", "ip_address": "192.168.56.130"}       |
| debcaf7b-c390-4f62-b0f3-0a78066c93c1 |                  | fa:16:3e:fe:39:ac | {"subnet_id": "ac8bba21-a440-4742-90be-c0ad4af920d8", "ip_address": "10.254.0.1"}           |
+--------------------------------------+------------------+-------------------+---------------------------------------------------------------------------------------------+
[vikas@fed-master devstack]$ 
~~~


### On *fed-node*:

~~~
[root@fed-node ~]# docker ps
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS               NAMES
d251bc90b370        nginx:1.7.9                          "nginx -g 'daemon off"   5 minutes ago       Up 5 minutes                             k8s_nginx.3ccf2b48_1-fedora.nginx_default_ecd8dfcb-66ac-11e6-848b-08002778068c_531a3721
c33dc30b5f56        gcr.io/google_containers/pause:2.0   "/pause"                 5 minutes ago       Up 5 minutes                             k8s_POD.6059dfa2_1-fedora.nginx_default_ecd8dfcb-66ac-11e6-848b-08002778068c_5abd8cdf
[root@fed-node ~]# docker exec -it d251bc90b370 bash
root@1-fedora:/# ifconfig eth0
bash: ifconfig: command not found
root@1-fedora:/# ip a         
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
1674: eth0@if1675: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether fa:16:3e:fc:8f:a1 brd ff:ff:ff:ff:ff:ff
    inet 182.168.0.9/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fefc:8fa1/64 scope link 
       valid_lft forever preferred_lft forever
root@1-fedora:/# 
~~~

### On *ubuntu-node* :

~~~
root@ubuntu-node:/usr/local/lib/python3.5/dist-packages# docker ps
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS               NAMES
7baf8dd55442        nginx:1.7.9                          "nginx -g 'daemon off"   5 minutes ago       Up 5 minutes                            k8s_nginx.3ccf2b48_1-ubuntu.nginx_default_c66f5bb0-6816-11e6-848b-08002778068c_1b0391bc
245b349ef693        gcr.io/google_containers/pause:2.0   "/pause"                 5 minutes ago       Up 5 minutes                            k8s_POD.6059dfa2_1-ubuntu.nginx_default_c66f5bb0-6816-11e6-848b-08002778068c_9dc9b6a6
root@ubuntu-node:/usr/local/lib/python3.5/dist-packages#
root@ubuntu-node:/usr/local/lib/python3.5/dist-packages# docker exec -it 7baf8dd55442 bash 
root@1-ubuntu:/# ip a                                                                                                                                                                                      
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
255: eth0@if256: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether fa:16:3e:ba:94:e3 brd ff:ff:ff:ff:ff:ff
    inet 182.168.0.6/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:feba:94e3/64 scope link 
       valid_lft forever preferred_lft forever
root@1-ubuntu:/# ping 182.168.0.9
PING 182.168.0.9 (182.168.0.9): 48 data bytes
56 bytes from 182.168.0.9: icmp_seq=0 ttl=64 time=2.661 ms
56 bytes from 182.168.0.9: icmp_seq=1 ttl=64 time=0.265 ms
56 bytes from 182.168.0.9: icmp_seq=2 ttl=64 time=1.247 ms
^C--- 182.168.0.9 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.265/1.391/2.661/0.983 ms
root@1-ubuntu:/# 
~~~

There you go !!!!!! 

Pods on different nodes pinging each other over vxlan tunnels !!!

Vxlan tunnels created by neutron between nodes, responsible for overlaying pod traffic, can be seen using ovs-vsctl cmnds:

~~~
root@ubuntu-node:/usr/local/lib/python3.5/dist-packages# ovs-vsctl show
51f4f634-e519-41b2-8dcf-4f8f3730b07f
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-c0a83865"
            Interface "vxlan-c0a83865"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.56.102", out_key=flow, remote_ip="192.168.56.101"}
        Port "vxlan-c0a83801"
            Interface "vxlan-c0a83801"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.56.102", out_key=flow, remote_ip="192.168.56.1"}

~~~

Please note that above is partial output only. Intention is to put focus on vxlan tunnels part.


Thats it !!!!!!!! :)

Hope you enjoyed the post!!!  
