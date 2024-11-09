# vrnetlab - VR Network Lab

This is a fork of the original [plajjan/vrnetlab](https://github.com/plajjan/vrnetlab) project. The fork has been created specifically to make vrnetlab-based images to be runnable by [containerlab](https://containerlab.srlinux.dev).

The documentation provided in this fork only explains the parts that have been changed in any way from the upstream project. To get a general overview of the vrnetlab project itself, consider reading the docs of the upstream repo.

## What is this fork about?
At [containerlab](https://containerlab.srlinux.dev) we needed to have [a way to run virtual routers](https://containerlab.srlinux.dev/manual/vrnetlab/) alongside the containerized Network Operating Systems.

Vrnetlab provides a perfect machinery to package most-common routing VMs in the container packaging. What upstream vrnetlab doesn't do, though, is creating datapath between the VMs in a "container-native" way.  
Vrnetlab relies on a separate VM (vr-xcon) to stich sockets exposed on each container and that doesn't play well with the regular ways of interconnecting container workloads.

This fork adds additional option for `launch.py` script of the supported VMs called `connection-mode`. This option allows to choose the way vrnetlab will create datapath for the launched VMs.

By adding a few options a `connection-mode` value can be set to, we made it possible to run vrnetlab containers with the networking that doesn't require a separate container and is native to the tools like docker.

### Container-native networking?
Yes, the term is bloated, what it actually means is that with the changes we made in this fork it is possible to add interfaces to a container that hosts a qemu VM and vrnetlab will recognize those interfaces and stitch them with the VM interfaces.

With this you can just add, say, veth pairs between the containers as you would do normally, and vrnetlab will make sure that these ports get mapped to your router' ports. In essence, that allows you to work with your vrnetlab containers like with a normal container and get the datapath working in the same "native" way.

> Although the changes we made here are of a general purpose and you can run vrnetlab routers with docker CLI or any other container runtime, the purpose of this work was to couple vrnetlab with containerlab.  
> With this being said, we recommend the readers to start their journey from this [documentation entry](https://containerlab.srlinux.dev/manual/vrnetlab/) which will show you how easy it is to run routers in a containerized setting.

## Connection modes
As mentioned above, the major change this fork brings is the ability to run vrnetlab containers without requiring vr-xcon and by using container-native networking.

The default option that we use in containerlab for this setting is `connection-mode=tc`. With this particular mode we use tc-mirred redirects to stitch container's interfaces `eth1+` with the ports of the qemu VM running inside.

![tc](https://gitlab.com/rdodin/pics/-/wikis/uploads/4d31c06e6258e70edc887b17e0e758e0/image.png)

Using tc redirection we get a transparent pipe between container's interfaces and VM's.

We scrambled through many alternatives, which I described in [this post](https://netdevops.me/2021/transparently-redirecting-packets/frames-between-interfaces/), but tc-redirect works best of them all.

Other connection mode values are:

* bridge - creates a linux bridge and attaches `eth` and `tap` interfaces to it. Can't pass LACP traffic.
* ovs-bridge - same as a regular bridge, but uses OvS. Can pass LACP traffic.
* macvtap

## Management interface

There are two types of management connectivity for NOS VMs: _pass-through_ and _host-forwarded_ (legacy) management interfaces.

_Pass-through management_ interfaces allows the use of the assigned management IP within the NOS VM, management traffic is transparently passed through to the VM, and the NOS configuration can accurately reflect the management IP. However, it is no longer possible to send or receive traffic directly in the vrnetlab container (e.g. for installing additional packages within the container), other than to pre-defined exceptions, such as the QEMU serial port on TCP port 5000.

NOSes defaulting to _pass-through_ management interfaces are:
- All vJunos routers

In case of _host-forwarded_ management interfaces, certain ports are forwarded to the NOS VM IP, which is always 10.0.0.15/24. The management gateway in this case is 10.0.0.2/24, and outgoing traffic is NATed to the container management IP. This management interface connection mode does not allow for traffic such as LLDP to pass through the management interface.

NOSes defaulting to _host-forwarded_ management interfaces are:
- Every NOS not listed as pass-through management

It is possible to change from the default management interface mode by setting the `CLAB_MGMT_PASSTHROUGH` environment variable to 'true' or 'false', however, it is left up to the user to provide a startup configuration compatible with the requested mode.

## Which vrnetlab routers are supported?
Since the changes we made in this fork are VM specific, we added a few popular routing products:

* Arista vEOS
* Cisco XRv9k
* Cisco XRv
* Cisco FTDv
* Juniper vMX
* Juniper vSRX
* Juniper vJunos-switch
* Juniper vJunos-router
* Juniper vJunosEvolved
* Nokia SR OS
* OpenBSD
* FreeBSD
* Ubuntu

The rest are left untouched and can be contributed back by the community.

## Does the build process change?
No. You build the images exactly as before.
