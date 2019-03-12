THE WHAT AND WHY OF CONTAINERS
------------------------------

Containers are fundamentally changing the way we develop, distribute, and run soft‐
ware. Developers can build software locally, knowing that it will run identically
regardless of host environment—be it a rack in the IT department, a user’s laptop, or
a cluster in the cloud.

Containers are an encapsulation of an application with its dependencies. At first
glance, they appear to be just a lightweight form of virtual machines (VMs)—like a
VM, a container holds an isolated instance of an operating system (OS), which we
can use to run applications.

However, containers have several advantages that enable use cases that are difficult or
impossible with traditional VMs:
* Containers share resources with the host OS, which makes them an order of magnitude more efficient
* The portability of containers has the potential to eliminate a whole class of bugs caused by subtle changes in the running environment
* The lightweight nature of containers means developers can run dozens of containers at the same time, making it possible to emulate a production-ready distributed-system
* Containers also have advantages for end users and developers outside of deploying to the cloud. 


**Containers Versus VMs**
Though containers and VMs seem similar at first, there are some important differences

Unlike VMs, the host’s kernel 2 is shared with the running containers. This means that containers are always 
constrained to running the same kernel as the host
Both VMs and containers can be used to isolate applications from other applications
running on the same host. 
1. VMs have an added degree of isolation from the hypervisor and are a trusted and battle-hardened technology. 
1. Containers are comparatively new, and many organizations are hesitant to completely trust the 
    isolation features of containers before they have a proven track record.

