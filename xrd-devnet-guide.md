# XRd Devnet Workshop
## Connecting to the Host Machine
1. Open webex and download `xrd-devnet.pem` that was sent to your webex space
2. Open an ubuntu terminal shell.
3. Copy the .pem file to your home directory: `cp ~/Downloads/xrd-devnet.pem ~/`
4. Connect to the host machine using the Ip address in your webex space. The command will be similar to `ssh -i "xrd-devnet.pem" ec2-user@<hostname>`. 



## Configuring the Host Machine
### Host-Check Script
In the [xrd-tools tutorial]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd), we introduced the "host-check" script that is published to github:  

>[https://github.com/ios-xr/xrd-tools/blob/main/scripts/host-check](https://github.com/ios-xr/xrd-tools/blob/main/scripts/host-check)


This script is an extremely handy tool to determine the suitability of the Host machine for either XRd platform - control-plane or vRouter.

Let's start by configuring our host by downloading and running the script:

	git clone https://github.com/ios-xr/xrd-tools
	cd xrd-tools
	pip3 install -r requirements.txt
	cd scripts
	./host-check

Output:

```bash
[ec2-user@ip-172-31-42-213 scripts] ./host-check
==============================
Platform checks
==============================

base checks
-----------------------
PASS -- CPU architecture (x86_64)
PASS -- CPU cores (8)
PASS -- Kernel version (5.4)
PASS -- Base kernel modules
        Installed module(s): dummy, nf_tables
PASS -- Cgroups (v1)
WARN -- Inotify max user instances
        The kernel parameter fs.inotify.max_user_instances is set to 8192 -
        this is expected to be sufficient for 2 XRd instance(s).
        The recommended value is 64000.
        This can be addressed by adding 'fs.inotify.max_user_instances=64000'
        to /etc/sysctl.conf or in a dedicated conf file under /etc/sysctl.d/.
        For a temporary fix, run:
          sysctl -w fs.inotify.max_user_instances=64000
PASS -- Inotify max user watches
        524288 - this is expected to be sufficient for 131 XRd instance(s).
WARN -- Socket kernel parameters
        The kernel socket parameters are insufficient for running XRd in a
        production deployment. They may be used in a lab deployment, but must
        be increased to the required minimums for production deployment.
        Lower values may result in XR IPC loss and unpredictable behavior,
        particularly at higher scale.

        The required minimum settings are:
            net.core.netdev_max_backlog=300000
            net.core.optmem_max=67108864
            net.core.rmem_default=67108864
            net.core.rmem_max=67108864
            net.core.wmem_default=67108864
            net.core.wmem_max=67108864

        The current host settings are:
            net.core.netdev_max_backlog=1000
            net.core.optmem_max=20480
            net.core.rmem_default=212992
            net.core.rmem_max=212992
            net.core.wmem_default=212992
            net.core.wmem_max=212992

        Values can be changed by adding e.g.
        'net.core.rmem_default=67108864' to /etc/sysctl.conf or
        in a dedicated conf file under /etc/sysctl.d/.
        Or for a temporary fix, running e.g.:
          sysctl -w net.core.rmem_default=67108864
WARN -- UDP kernel parameters
        The kernel UDP parameters are insufficient for running XRd in a
        production deployment. They may be used in a lab deployment, but must
        be increased to the required minimums for production deployment.
        Lower values may result in XR IPC loss and unpredictable behavior,
        particularly at higher scale.

        The required minimum settings are:
            net.ipv4.udp_mem=1124736 10000000 67108864
        The current host settings are:
            net.ipv4.udp_mem=767118 1022825 1534236
        Values can be changed by adding
        'net.ipv4.udp_mem=1124736 10000000 67108864' to /etc/sysctl.conf or
        in a dedicated conf file under /etc/sysctl.d/.
        Or for a temporary fix, running:
          sysctl -w net.ipv4.udp_mem='1124736 10000000 67108864'
INFO -- Core pattern (core files managed by XR)
PASS -- ASLR (full randomization)
INFO -- Linux Security Modules (No LSMs are enabled)

xrd-control-plane checks
-----------------------
PASS -- RAM
        Available RAM is 30.9 GiB.
        This is estimated to be sufficient for 15 XRd instance(s), although memory
        usage depends on the running configuration.
        Note that any swap that may be available is not included.

xrd-vrouter checks
-----------------------
PASS -- CPU extensions (sse4_1, sse4_2, ssse3)
PASS -- RAM
        Available RAM is 30.9 GiB.
        This is estimated to be sufficient for 6 XRd instance(s), although memory
        usage depends on the running configuration.
        Note that any swap that may be available is not included.
FAIL -- Hugepages
        Hugepages are not enabled. These are required for XRd to function correctly.
        To enable hugepages, see the instructions at:
        https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt.
PASS -- Interface kernel driver
        Loaded PCI drivers: vfio-pci, igb_uio
INFO -- IOMMU
        vfio-pci is set up in no-IOMMU mode, but IOMMU is recommended for security.
PASS -- Shared memory pages max size (17179869184.0 GiB)

==================================================================
!! Host NOT set up correctly for any XR platforms !!
==================================================================
```

#### System Kernel Parameters
Let's start by modifying `/etc/sysctl.conf`


```bash
sudo bash -c 'cat << EOF >> /etc/sysctl.conf
fs.inotify.max_user_instances=64000
net.core.netdev_max_backlog=300000
net.core.optmem_max=67108864
net.core.rmem_default=67108864
net.core.rmem_max=67108864
net.core.wmem_default=67108864
net.core.wmem_max=67108864
net.ipv4.udp_mem=1124736 10000000 67108864
kernel.sched_rt_runtime_us=-1
net.bridge.bridge-nf-call-iptables=0
vm.max_map_count=524288
EOF'
```

#### IOMMU and HugePages Settings

Next, for XRd vRouter to work, enable iommu and Hugepages for the Host machine by adding the `GRUB_CMDLINE_LINUX` line to the end of the file

	sudo bash -c 'cat << EOF >> /etc/default/grub
	GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt default_hugepagesz=1G hugepagesz=1G hugepages=9"
	EOF'


Now apply these new settings by:

	sudo grub2-mkconfig -o /boot/grub2/grub.cfg
	sudo reboot

After running the `host-check` script again, we see that both platforms are supported on our host

	cd ~/xrd-tools/scripts
 	./host-check
## XRd on Docker
### Loading the XRd images
First let's start the docker daemon:
```bash
sudo systemctl start docker
```

Then we are able to load our XRd images from tarballs:

```bash
cd ~
docker load -i xrd-control-plane-container-x64.dockerv1.tgz
docker load -i xrd-vrouter-container-x64.dockerv1.tgz
```

Outputs:
```
a04ec920c675: Loading layer [==================================================>]  1.175GB/1.175GB
Loaded image: ios-xr/xrd-control-plane:7.9.1
887387959b52: Loading layer [==================================================>]  1.217GB/1.217GB
Loaded image: ios-xr/xrd-vrouter:7.8.2
```
Now we can se that these images are available to docker:

```bash
[ec2-user@ip-172-31-42-213 ~]$ docker image ls
REPOSITORY                 TAG       IMAGE ID       CREATED        SIZE
ios-xr/xrd-control-plane   7.9.1     c62ba3090fc6   5 weeks ago    1.15GB
ios-xr/xrd-vrouter         7.8.2     fc8f44da003b   8 weeks ago    1.19GB
```

### Launching a topology of XRds using Docker

The `xr-compose` script is a wrapper around docker-compose. In addition to the general [docker-compose YAML syntax](https://docs.docker.com/compose/compose-file/), xr-compose also supports some XR-specific fields that 'expand out' to include all the fields required to hide implementation-specific details from the user. It also takes care of boilerplate docker-compose items that are desired for every XR container service.

The `xr-tools` repo comes with several sample xr-compose topologies. Let's navigate to `~/xrd-tools/samples/xr_compose_topos/simple-bgp/`

	cd ~/xrd-tools/samples/xr_compose_topos/simple-bgp/

Let's take a closer look at `docker-compose.xr.yml`, `xrd-1_xrconf.cfg`, `xrd-2_xrconf.cfg`

Now we can build our docker compose file and launch our topology.

1. Build our docker compose file
	```bash
	../../../scripts/xr-compose -f docker-compose.xr.yml -i ios-xr/xrd-control-plane:7.9.1
 	```

2. Launch our topology with docker compose
	```bash
   	docker compose up -d
 	```
Output:	
```bash
$ docker compose up -d
[+] Running 8/8
✔ Network simple-bgp_xrd-2-dest    Created                                                                                                    3.0s 
✔ Network simple-bgp_source-xrd-1  Created                                                                                                    2.6s 
✔ Network simple-bgp_mgmt          Created                                                                                                    2.6s 
✔ Network xr-1-gi1-xr-2-gi0        Created                                                                                                    1.9s 
✔ Container xr-1                   Created                                                                                                   24.0s 
✔ Container dest                   Created                                                                                                   18.3s 
✔ Container xr-2                   Created                                                                                                   22.2s 
✔ Container source                 Created                                                                                                   17.8s 
Attaching to dest, source, xr-1, xr-2
```

Verify that our containers have been launched

```bash
$ docker ps
CONTAINER ID   IMAGE                            COMMAND                  CREATED         STATUS         PORTS                       NAMES
46dfab9dea83   ios-xr/xrd-control-plane:7.9.1   "/usr/sbin/init"         5 minutes ago   Up 4 minutes                               xr-2
abd543b5b31c   alpine:3.15                      "/bin/sh -c 'ip rout…"   5 minutes ago   Up 4 minutes                               source
c53390dcf2e5   ios-xr/xrd-control-plane:7.9.1   "/usr/sbin/init"         5 minutes ago   Up 4 minutes                               xr-1
afa0e3181070   alpine:3.15                      "/bin/sh -c 'ip rout…"   5 minutes ago   Up 5 minutes                               dest
```

We can attach to `xr-1` and verify that we have a bgp session

	docker attach xr-1

And once we login to our router, we can check the status of our bgp session

```bash
RP/0/RP0/CPU0:ios#sh bgp neighbor brief
Wed May 17 01:05:12.904 UTC

Neighbor         Spk    AS  Description                         Up/Down  NBRState
10.2.1.3          0   100                                      00:09:31 Established 
```
To **exit** from the router use the escape sequence: `^P^Q`

Finally, we can attach to the Alpine Linux container `source` and ping `dest`, having our packets routed through `xr-1` and `xr-2`

```bash
$ docker attach source
/ # ping 10.3.1.3
PING 10.3.1.3 (10.3.1.3): 56 data bytes
64 bytes from 10.3.1.3: seq=0 ttl=253 time=3.539 ms
64 bytes from 10.3.1.3: seq=1 ttl=253 time=2.325 ms
64 bytes from 10.3.1.3: seq=2 ttl=253 time=2.248 ms
64 bytes from 10.3.1.3: seq=3 ttl=253 time=2.341 ms
^C
--- 10.3.1.3 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 2.248/2.613/3.539 ms
```

## XRd on Kubernetes
Create a kind cluster using our custom config file, this specifies number of control plane and worker nodes.
```
cd ~
kind create cluster --config=kind.yml
```

Once that command is complete, we should use kubectl to interact our cluster

```
kubectl cluster-info --context kind-xrd-cluster
```

```
kubectl get nodes
```

Now we can load our xrd-vrouter image into kind's image registry:
```
kind load docker-image ios-xr/xrd-vrouter:7.8.2 -n xrd-cluster
```


### Add Helm Repo and Deploy a Multi Router topology with an IGP

```
helm repo add xrd-helm https://ios-xr.github.io/xrd-helm
```

Take a look at the custom chart located in **multi-xrd-helm/**
First apply this smart devices config so we can enable pci passthrough:
```
kubectl apply -f devices.yml
```

Next, to deploy this topology we will install this chart with:

```
helm install multi multi-xrd-helm/
```

Once again, let's view our pods and see which nodes they're running in:

```
kubectl get pods -o wide
```
Output:
```
NAME          READY   STATUS    RESTARTS   AGE   IP           NODE                  NOMINATED NODE   READINESS GATES
multi-xrd1-0   1/1     Running   0          28m   172.17.0.3   xrd-cluster-worker    <none>           <none>
multi-xrd2-0   1/1     Running   0          28m   172.17.0.5   xrd-cluster-worker2   <none>           <none>
multi-xrd3-0   1/1     Running   0          28m   172.17.0.2   xrd-cluster-worker3   <none>           <none>
```

And we can exec into our pods just like before:

```
kubectl exec -it multi-xrd1-0 -- xr
```

Now let's upgrade our helm release, and update the startup config of our XRd devices. This new config will run OSPF between or XRds. However, in a cloud environment, we cannot run IGPs directly over an interface, so we will have to do it over GRE tunnels.

To do this, we'll create a new values file that includes only the parameters that we wish to upgrade from the original deployment.

**igp-values.yaml**

```
sudo bash -c 'cat << EOF > igp-values.yaml
xrd1:
  config:
    ascii: |
      hostname xrd1
      interface TenGigE0/0/0/0
        ipv4 address 20.0.0.10/18
        no shutdown
      !
      interface tunnel-ip1
       ipv4 address 10.0.0.1/30
       tunnel source TenGigE0/0/0/0
       tunnel destination 20.0.0.12
       logging events link-status
      !
      router ospf xrd
       area 0
        interface tunnel-ip1
        !
      !
xrd2:
  config:
    ascii: |
      hostname xrd2
      interface TenGigE0/0/0/0
        ipv4 address 20.0.0.12/18
        no shutdown
      !
      interface tunnel-ip1
       ipv4 address 10.0.0.2/30
       tunnel source TenGigE0/0/0/0
       tunnel destination 20.0.0.10
       logging events link-status
      !
      interface tunnel-ip3
       ipv4 address 10.0.0.5/30
       tunnel source TenGigE0/0/0/0
       tunnel destination 20.0.0.14
       logging events link-status
      !
      router ospf xrd
       area 0
        interface tunnel-ip1
        !
        interface tunnel-ip3
        !
      !

xrd3:
  config:
    ascii: |
      hostname xrd3
      interface TenGigE0/0/0/0
        ipv4 address 20.0.0.14/18
        no shutdown
      !
      interface tunnel-ip3
       ipv4 address 10.0.0.6/30
       tunnel source TenGigE0/0/0/0
       tunnel destination 20.0.0.12
       logging events link-status
      !
      router ospf xrd
       area 0
        interface tunnel-ip3
        !
      !
EOF'
```

And we can modify our deployment like so:

```
helm upgrade multi multi-xrd-helm/ -f igp-values.yaml
```

Finally, delete the current pods so they spin up with the new config

```
kubectl delete pod multi-xrd1-0 multi-xrd2-0 multi-xrd3-0
```
