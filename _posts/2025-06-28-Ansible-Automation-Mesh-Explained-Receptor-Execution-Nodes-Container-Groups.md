---
title: "Ansible Automation Mesh Explained: Receptor, Execution Nodes, and Container Groups"
author: bbgrimmett2
date: 2025-06-28 12:00:00 -0600
categories: [Ansible, Infrastructure]
tags: [homelab]
render_with_liquid: false
mermaid: true
---

## Introduction
Ansible Automation Platform (AAP) is designed to be a centralized platform for an entire organization's IT operations needs. Either deployed on virtual machine(s) (VM) or OpenShift Container Platform (OCP), the deployment normally happens within a single network; so, how do you get AAP to connect to all of your devices? Do you have to open firewall rules to every subnet, IP address, location, everywhere? Thats where Automation Mesh comes into play— where you can setup remote nodes with single entrypoints into these remote networks. Automation Mesh is "designed to help you scale automation from on-premise environments, throughout hybrid clouds, to edge locations—all centrally managed via automation controller" ([Red Hat - Product Feature Automation Mesh](https://www.redhat.com/en/technologies/management/ansible/automation-mesh)). Where this becomes relevant is when you are trying to automate manufacturing devices, point of sale systems, your homelab's remote network, or even a Steam Deck (future post spoiler warning). Our goal here is to bring the automation execution, the `ansible-playbook`, closer to our managed nodes resulting in more resilient automation executions.

## Automation Mesh

### Receptor
The tool behind the magic is Ansible Receptor; this is the engine designed to distribute job data and coordinate messages from the platform across various networks. Receptor works by establishing peer-to-peer connections via existing network paths and provides secure communication paths ([GitHub - Receptor](https://github.com/ansible/receptor)). Once implemented, Receptor becomes the backbone for AAP's automation job execution across control, hybrid, execution, and hop nodes. A control node is responsible for running jobs such as project and inventory updates. An execution node is responsible for running the automation itself; it executes commands such as `ansible-playbook` to manage target systems across the network. A hybrid node combines the responsibilities of both a control and execution node which is capable of scheduling jobs and running automation tasks locally. Finally, a hop node acts as a network relay or "jump point" to reach remote or restricted environments; this is similar to how a bastion host enables SSH access to isolated systems. It doesn’t run automation itself but instead routes job traffic through the mesh to reach execution nodes that may not be directly accessible from control nodes ([Red Hat Docs - Automatin Mesh](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/automation_mesh_for_vm_environments/assembly-planning-mesh#con-automation-mesh)).

Receptor is designed to be resilient, so, to enable this, it works by using a peer-to-peer topology. This does not mean that the mesh self-heals dynamically but rather re-evaluates routes when needed based on available peer connectivity. Through this there is no central broker; therefore, every node is equal and connections are made directly to peers. Think of it like an automation spider web: if one strand breaks, network traffic can reroute through another path. This built-in mesh behavior is how Receptor creates resiliency and maintains communication even when parts of the mesh go offline or become unreachable. Additionally, it is important to clarify when saying, "living on remote systems", that Receptor is not an agent; therefore, it is not going against the Ansible philosophy of agentless ([Red Hat - How Ansible Works?](https://www.redhat.com/en/ansible-collaborative/how-ansible-works)). Rather, Receptor is designed to be a peer service; this means there is no centralized agent controller, but, with the peer-to-peer design, it allows nodes to independently accept jobs, forward messages, run secure workloads, and return results. 

Of course, distributing automation across networks raises a key question: how is this kept secure? Receptor doesn’t just connect nodes... it does so with built-in mechanisms to protect credentials, data, and communication. Every Receptor connection can be secured via TLS encryption; this means that communication can be encrypted in transit and can only be decrypted by a peer with the correct certificate and key ([Internet Society - TLS Basics](https://www.internetsociety.org/deploy360/tls/basics/)). Therefore, all job data, inventory, and logs are protected from interception during transmission. Receptor also supports mutual TLS (mTLS) which requires both the sender and receiver to present valid certificates to verify each other’s identity. Through these TLS handshakes, Receptor defends against common network threats such as man-in-the-middle attacks, impersonation, data tampering, and unauthorized access. While TLS secures communication channels, Receptor can add additional security via optional GPG job signing; this ensures the authenticity and integrity of job payloads even when routed through multiple hops ([Ansible Docs - Receptor TLS Support](https://ansible.readthedocs.io/projects/receptor/en/latest/user_guide/tls.html)).

### Instance Groups and Container Groups

Now that we understand how Automation Mesh enables communication between nodes, let’s talk about how Ansible Automation Platform (AAP) decides *where* automation runs.

#### What Are Instance Groups?
Instance groups are logical collections of nodes (execution, hybrid, or control) that Ansible Automation Platform (AAP) uses to determine where automation jobs are executed. When a job is launched, AAP selects an available node within the assigned instance group to run it.

Instance groups can be scoped at the organization level which allows for teams to have dedicated infrastructure for their specific responsibilities. For example, you might configure one instance group that only has access to network devices and assign it to the **networking team** while another instance group is restricted to Linux servers and used exclusively by the **Linux team**. This separation helps enforce organizational policy around access and reduces the risk of cross-environment errors.

By leveraging AAP’s built-in role-based access control (RBAC), administrators can tightly control which teams can see and use specific instance groups—ensuring that job execution only happens in the environments teams are authorized to manage.

You can also apply instance groups at the inventory level for finer control.  
For example:
> The Linux team is responsible for two data centers: **DC-East** and **DC-West**. Each has its own isolated execution node group. By assigning the corresponding instance group to each inventory, AAP ensures jobs targeting **DC-East** only run on nodes with network access to **DC-East**; the same process occurs for **DC-West**. This guarantees geographic and network isolation without needing complex job templates or credential rules.

```mermaid
graph TD
    subgraph Organization
        NET[Networking Team]
        LNX[Linux Team]
    end

    subgraph Instance Groups
        IG_NET[Instance Group: Network Devices]
        IG_LNX_DC1[Instance Group: Linux - DC-East]
        IG_LNX_DC2[Instance Group: Linux - DC-West]
    end

    subgraph Inventories
        INV_NET[Inventory: Routers & Switches]
        INV_LNX_DC1[Inventory: Linux Servers DC-East]
        INV_LNX_DC2[Inventory: Linux Servers DC-West]
    end

    NET --> IG_NET
    IG_NET --> INV_NET

    LNX --> IG_LNX_DC1
    LNX --> IG_LNX_DC2
    IG_LNX_DC1 --> INV_LNX_DC1
    IG_LNX_DC2 --> INV_LNX_DC2
```

These instance groups rely on **Receptor** to move automation job data and coordination messages across the mesh. Receptor ensures that communication between the controller and execution nodes is secure, resilient, and network-aware.

#### Execution Nodes

Execution nodes are persistent infrastructure like VMs or bare metal machines that run `ansible-playbook` to manage your target systems. These nodes are running automation tasks directly, and, like we talked about earlier, these machines are using Receptor to communicate with the controller or other Automation Mesh peers. These servers (control, execution, and hop nodes) are all communicating via port 27199 by default; as a result, this is the only port required to be open to allow for remote job execution enabled by Receptor.

A use case for execution nodes:
> I manage a remote network network environment that connects back to my primary datacenter via a site-to-site VPN; I’ve deployed an execution node to handle automation tasks locally. This remote site is connected to the internet using Starlink which introduces unique challenges like high latency and variable network performance due to its satellite-based infrastructure and frequent constellation handovers. To maintain reliable automation under these conditions, I placed a dedicated execution node inside the Starlink-connected environment. These nodes run Receptor allowing for a secure mesh connection with my central automation controller. From nightly health check of remote servers and building a self healing network, the jobs are able to execute locally which reduces the constant real-time communication high-latency link to the primary datacenter where the control node lives. By bringing automation execution closer to the managed devices, I gain resiliency, reduce job failure risk, and maintain operational stability—even when Starlink's performance fluctuates.

#### Container Groups

Container groups represent a dynamic, cloud-native method for running automation tasks in Ansible Automation Platform (AAP). Unlike execution nodes, container groups do not rely on persistent infrastructure—instead, they spin up ephemeral containers inside a Kubernetes or OpenShift cluster to execute jobs. Despite being ephemeral, container group pods stream live job output back to the automation controller where logs, status, and artifacts are captured and stored for review and auditing.

Container groups work by directly communicating with the Kubernetes/OpenShift API. When a job is launched, an ephemeral pod is created using a specified execution environment image. The container runs the automation and **is destroyed after the completion of the job execution**. Since container groups directly communicated with the kubernetes API, this is happening without Receptor and is outside of Automation Mesh. Through configuration of your container platform, network policies can be managed at specific namespace level (see sources for more information). The AAP controller uses a credential called "OpenShift or Kubernetes API Bearer Token" which is used to start up the pod and establish communication where the pod spec can be customized at the creation of the container group. The default can be seen below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  labels:
    ansible_job: ''
spec:
  serviceAccountName: default
  automountServiceAccountToken: false
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: ansible_job
                  operator: Exists
            topologyKey: kubernetes.io/hostname
  containers:
    - image: >-
        registry.redhat.io/ansible-automation-platform-25/ee-supported-rhel8:latest
      name: worker
      args:
        - ansible-runner
        - worker
        - '--private-data-dir=/runner'
      resources:
        requests:
          cpu: 250m
          memory: 100Mi
```
The main advantage of these container groups is for scalability and flexibility of automation execution allowing them to take advantage of native scaling through kubernetes. These features are best served under more demanding workloads like system patching;  this requires more CPU and RAM for software installs when working with hundreds or thousands of systems at one time. If OpenShift/Kubernetes is a part of your environment, container groups can be super useful in the handling and scaling of your automation jobs in addition to taking advantage of resources given.

## What is Best for You?
Choosing between execution nodes and container groups depends on your environment, infrastructure strategy, and automation needs.

### Use Execution Nodes When:
- You need persistent infrastructure for long-running or stateful jobs.
- Your jobs require access to local file systems, attached storage, or specific network routes.
- You're operating in **edge environments** or over **constrained links**.
- You want **Receptor-based resiliency** and mesh connectivity across networks.
- You do not have access to a kubernetes platform.

### Use Container Groups When:
- You’re running automation in Kubernetes or OpenShift environments.
- You want **on-demand, scalable job execution** without maintaining persistent nodes.
- Your workloads are **ephemeral, burstable, or compute-intensive**.
- You want to isolate job execution and take advantage of **Kubernetes-native controls**.

| Component         | Uses Receptor? | Persistent Node? | Runs Jobs? | Purpose                                      |
|------------------|----------------|------------------|------------|----------------------------------------------|
| Control Node      | ✅ Yes         | ✅ Yes           | ✅ Yes     | Schedules jobs, SCM access, runs system jobs |
| Execution Node    | ✅ Yes         | ✅ Yes           | ✅ Yes     | Runs automation workloads                    |
| Hybrid Node       | ✅ Yes         | ✅ Yes           | ✅ Yes     | Combines control and execution capabilities  |
| Hop Node          | ✅ Yes         | ✅ Yes           | ❌ No      | Relays traffic across network boundaries     |
| Container Group   | ❌ No          | ❌ No (ephemeral)| ✅ Yes     | Runs jobs in temporary containers            |

### Mixed Environments

In many cases, you don’t have to choose just one. Ansible Automation Platform supports **hybrid models** which allows you to be able to run container groups for scale-out workloads and execution nodes for edge or network-isolated systems all from a single automation controller.

> Design your automation execution strategy based on **network layout**, **workload type**, and **available infrastructure**.

## Final Thought
No matter where your automation needs to run, from the data-center to the edge or across the cloud platforms, Ansible Automation Platform gives you the flexibility to meet those demands with the right execution model. Whether you lean on the resilience of execution nodes or the scalability of container groups, the real power lies in choosing what best fits your environment. 

## Sources:




- [Red Hat - Automation Architect Handbook](https://www.redhat.com/en/resources/automation-architect-handbook-ebook)
- [Red Hat Docs - OpenShift Network Policy](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/network_observability/network-observability-network-policy)
- [Red Hat - How Ansible Works](https://www.redhat.com/en/ansible-collaborative/how-ansible-works)
- [Internet Society - TLS Basics](https://www.internetsociety.org/deploy360/tls/basics/)
- [Red Hat Docs - Automation Mesh for VM Environments](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/automation_mesh_for_vm_environments/assembly-planning-mesh#con-automation-mesh)
- [Red Hat Docs - Automation Mesh for Operator-Based Red Hat Ansible Automation Platform](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/automation_mesh_for_managed_cloud_or_operator_environments/assembly-automation-mesh-operator-aap#ref-operator-mesh-prerequisites)
- [Ansible Docs - Container and Instance Groups](https://docs.ansible.com/automation-controller/4.4/html/administration/containers_instance_groups.html#ag-instance-group-policies)
- [YouTube Alex Dworjan - Ansible Automation Mesh](https://youtu.be/T4YYmT-isTQ?si=nBappnwWCv7rQvQA)
- [Red Hat Product Feature - Automation Mesh](https://www.redhat.com/en/technologies/management/ansible/automation-mesh)
- [Ansible Docs - Instance Groups](https://docs.ansible.com/automation-controller/4.4/html/userguide/instance_groups.html)
- [Ansible Docs - Ansible Receptor](https://ansible.readthedocs.io/projects/receptor/en/latest/index.html)
- [GitHub - Receptor](https://github.com/ansible/receptor)
