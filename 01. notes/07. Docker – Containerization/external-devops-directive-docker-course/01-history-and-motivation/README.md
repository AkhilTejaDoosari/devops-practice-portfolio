[Home](../README.md) | [History and Motivation](../01-history-and-motivation/README.md)
| [Technology Overview](../02-technology-overview/README.md)
| [Installation and Set Up](../03-installation-and-set-up/README.md)
| [Using 3rd Party Containers](../04-using-3rd-party-containers/README.md)
| [Example Web Application](../05-example-web-application/README.md)
| [Building Container Images](../06-building-container-images/README.md)
| [Container Registries](../07-container-registries/README.md)
| [Running Containers](../08-running-containers/README.md)
| [Container Security](../09-container-security/README.md)
| [Interacting with Docker Objects](../10-interacting-with-docker-objects/README.md)
| [Development Workflows](../11-development-workflow/README.md)
| [Deploying Containers](../12-deploying-containers/README.md)

---

# History and Motivation

<!-- no toc -->
  - [What is a container?](#what-is-a-container)
  - [History of virtualization](#history-of-virtualization)
    - [Bare Metal](#bare-metal)
    - [Virtual Machines](#virtual-machines)
    - [Containers](#containers)
    - [Tradeoffs](#tradeoffs)

---

## What is a container?

A Docker container image is a lightweight, standalone, executable package of software that includes everything needed to run an application (https://www.docker.com/resources/what-container/).

## History of virtualization

### Bare Metal

**What this means?**  
In a bare metal setup, applications run directly on the same operating system without strong separation. All applications share the same OS, system libraries, CPU, and memory. Because there are no clear boundaries, one application can directly affect others.

**Why this is a problem?**  
If one app installs or upgrades a library, it may break another app. If one app consumes too much CPU or memory, it can slow down the entire system. If one app crashes, the impact can spread beyond just that app. Over time, this makes systems fragile and hard to manage.

**Simple analogy!**  
Imagine multiple people cooking in the same kitchen with **one stove and one pantry**. Everyone uses the same ingredients and tools. If one person uses all the ingredients or burns the stove, everyone else is affected. There is no separation, so one person’s mistake becomes everyone’s problem.

![](./readme-assets/bare-metal.jpg)

**Why the industry moved on:**
- Apps break each other  
Different apps need different versions of the same software, so installing or updating one app can break another.

- Machine resources are wasted  
CPU and memory are not used well; one app may use too much while others sit idle.

- One problem affects everything  
If one app crashes or misbehaves, it can impact the whole system.

- Starting and stopping is slow  
Services take minutes to start or stop.

- Creating and removing systems is very slow  
Setting up or removing a machine takes hours or even days.

---

### Virtual Machines

**What this means?**  
In a virtual machine setup, applications do not run directly on the host OS.
Instead, a hypervisor creates multiple virtual computers on one physical machine.
Each virtual machine has its own operating system, libraries, CPU share, and memory.
Because each VM is separated, one VM cannot directly mess with another.

**Why this is better than bare metal?**  
Since every VM has its own OS and environment:
- Apps don’t fight over libraries
- Crashes usually stay inside one VM
- Resources are more controlled

This makes systems more stable and predictable than bare metal.

**Simple analogy!**  
Imagine an apartment building.
- Each family lives in their own apartment
- Everyone has their own kitchen and bathroom
- If one family burns food, it doesn’t destroy the whole building

There is separation, but the building itself is still shared.

![](./readme-assets/virtual-machine.jpg)

**What problems still exist?**.  

Even though VMs fix many bare-metal issues, they introduce new ones:

- Each VM runs a full operating system
- OS takes memory, CPU, and disk even if the app is small
- Starting a VM takes minutes, not seconds
- Creating or deleting VMs is still slow
- Running many VMs becomes expensive and heavy

**Why the industry moved forward again**

- Too much overhead per app (full OS every time)
- Slower startup compared to containers
- Lower density (fewer apps per machine)
- Not ideal for fast development and scaling

**Virtual machines solved isolation and stability, but they are still heavy, slow, and resource-hungry.**  
That gap is exactly where containers come in next.

---

### Containers

**What this means?**  
In a container setup, applications do not get their own operating system. There is one operating system on the machine, and all containers use that same OS core (kernel). 
Each application runs inside its own container, which gives it:
- its own files
- its own settings
- its own view of the system
So even though apps share the same OS underneath, they cannot see or touch each other.
This separation is created using built-in Linux features, not fake hardware and not extra operating systems.

**Why this is an improvement?**  
Compared to virtual machines:
- No extra OS to install
- No OS to boot for every app
- Much less memory and CPU usage
- Apps start almost instantly
You can run many containers on one machine without wasting resources.

**Simple analogy!**
Imagine an apartment building. One building, One plumbing system, One power connection

Each apartment:
- has its own door
- its own rooms
- its own locks

People inside one apartment cannot see or affect people in another apartment.  
The building = host operating system  
The apartments = containers  
Everyone shares the same building, but lives separately.

![](./readme-assets/container.jpg)

**Why the industry moved here**. 

- Apps no longer break each other
- Resources are used more efficiently
- Starting and stopping apps takes seconds
- Easy to create, delete, and move apps
- Perfect for development and modern cloud systems



---

### Tradeoffs

![](./readme-assets/tradeoffs.jpg)

***Note:*** There is much more nuance to “performance” than this chart can capture. A VM or container doesn’t inherently sacrifice much performance relative to the bare metal it runs on, but being able to have more control over things like connected storage, physical proximity of the system relative to others it communicates with, specific hardware accelerators, etc… do enable performance tuning
