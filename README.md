# Launch a Kubernetes Cluster with Fleet

## Purpose
The purpose of this repository was my attempt at demonstrating that it is entirely possible to launch a Kubernetes cluster using solely unit files launched using Fleet across CoreOS machines (and it was!)

### Background

There is a boundless bevy of repositories that are used to run Kubernetes, most of which use Ansible, which is a great tool for automating just about anything, including Kubernetes deployments of every shape and size and arrangement. 

In looking at fleet, the author of this repository realized what a great scheduler it is and how it's quite possible to use it to run any number of services, including Kubernetes.

There are a few repositories out there that attempt it but it seems they are limited to a single machine. 


### How this is done
The goal of this repository is to take advantage of the environment that is set up on a stock CoreOS system and a pre-existing etcd cluster.

Using the official, [CoreOS Vagrant](https://github.com/coreos/coreos-vagrant.git) repo, unit files were developed that use shell magic, Kubernete's etcd entries, as well as CoreOS variables to make this work.

All the info is there and with etcd, it's possible to have everything available to do this cross-node (vs. single).

#### Description of unit files

* ```fleet-hosts.service``` - Ensures that every host in the cluster has an entry in ```/etc/hosts``` on every machine
* ```kubernetes-env.service``` -- Ensures that that environment variables are set up that kubernetes requires. This can be edited for the user's environment
* ```kubernetes-api.service``` -- Starts up the kubernetes API server.
* ```kubernetes-controller-manager.service``` -- Starts up the kubernetes controller. Runs on the same machine as the API server. Uses the entry in etcd for ```/registry/services/endpoints/default/kubernetes``` to obtain the IP address of the API server
* ```kubernetes-scheduler.service``` -- Starts the kubernetes scheduler. Runs on the same machine as the API server. Uses the entry in etcd for ```/registry/services/endpoints/default/kubernetes``` to obtain the IP address of the API server
* ```kubelet.service``` -- Starts of the kubelet. This is a global service and runs on all machines. Uses the entry in etcd for ```/registry/services/endpoints/default/kubernetes``` to obtain the IP address of the API server.
* ```kubernetes-proxy.service``` -- Starts a kubernetes proxy server. Runs on any machine with a kubelet service running. 
* ```kubernetes-dns.service``` -- Starts SkyDNS controller and service on the newly minted kubernetes cluster. Uses curl to obtain "template" files and regex the correct API server IP address as well as domain and IP address the service will run on.

### To run

The easiest way to run these is to log into one of the coreos servers and check out the code. Clone this repo, submit in this order:

1. ```fleetctl start fleet-hosts.service```
2. ```fleetctl start kube*```

That's it! Then have at it:

```
core@core-01 ~ $ fleetctl start fleet-hosts.service 
Unit fleet-hosts.service 
Triggered global unit fleet-hosts.service start
core@core-01 ~ $ fleetctl start kubernetes-env.service 
Unit kubernetes-env.service 
Triggered global unit kubernetes-env.service start
core@core-01 ~ $ fleetctl start kube*                        
Unit kubelet.service 
Unit kubernetes-apiserver.service inactive
Unit kubernetes-controller-manager.service inactive
Unit kubernetes-proxy.service 
Triggered global unit kubelet.service start
Triggered global unit kubernetes-proxy.service start
Unit kubernetes-apiserver.service launched on 489ce01d.../172.17.8.103
Unit kubernetes-controller-manager.service launched on 489ce01d.../172.17.8.103
core@core-01 ~ $ fleetctl list-units
UNIT					MACHINE				ACTIVE		SUB
fleet-hosts.service			489ce01d.../172.17.8.103	inactive	dead
fleet-hosts.service			92bbccb6.../172.17.8.101	inactive	dead
fleet-hosts.service			aa3851c0.../172.17.8.102	inactive	dead
kubelet.service				489ce01d.../172.17.8.103	active		running
kubelet.service				92bbccb6.../172.17.8.101	active		running
kubelet.service				aa3851c0.../172.17.8.102	active		running
kubernetes-apiserver.service		489ce01d.../172.17.8.103	active		running
kubernetes-controller-manager.service	489ce01d.../172.17.8.103	active		running
kubernetes-dns.service			489ce01d.../172.17.8.103	activating	start-pre
kubernetes-env.service			489ce01d.../172.17.8.103	inactive	dead
kubernetes-env.service			92bbccb6.../172.17.8.101	inactive	dead
kubernetes-env.service			aa3851c0.../172.17.8.102	inactive	dead
kubernetes-proxy.service		489ce01d.../172.17.8.103	active		running
kubernetes-proxy.service		92bbccb6.../172.17.8.101	active		running
kubernetes-proxy.service		aa3851c0.../172.17.8.102	active		running
kubernetes-scheduler.service		489ce01d.../172.17.8.103	active		running
core@core-01 ~ $ kubectl -s http://172.17.8.103:8080 get pods,rc,services
NAME                    READY        STATUS        RESTARTS   AGE
kube-dns-03r0v          0/3          Pending       0          1d
nginx-198147104-dnpfb   0/1          Pending       0          1d
NAME                    DESIRED      CURRENT       AGE
kube-dns                1            1             2d
NAME                    CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns                10.1.0.100   <none>        53/UDP,53/TCP   2d
kubernetes              10.1.0.1     <none>        443/TCP         5d
core@core-01 ~ $ 
``` 

