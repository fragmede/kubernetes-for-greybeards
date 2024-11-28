# **Kubernetes for Greybeards: A Pragmatic Guide**

## **Introduction**

How grey is my beard? I started with C++ on SunOS 5.7 on SPARC stations. When I picked up Linux, it had a 2.2 kernel, floppies were just getting out of style, 
and Slackware was the distro de jour. I've come a long way since then, but I've been huffing the Kubernetes glue 
at my latest job and decided to write this guide to see how well Cunningham's law works.

This guide is targeted towards UNIX greybeards that want a concise but comprehensive guide to Kubernetes. It
assumes you have opinions on on POSIX and how much we have strayed from the light, which regular expression library is the best, and have a strong 
Opinion about `curl | bash`. You've also argued about the finer points of GPL vs LGPL vs MIT on the Internet at some point.

Docker is a whole thing, you should know what containers are and understand why they aren't VMs.
This guide is going to use docker so if you're allergic to it, this isn't the guide for you.

You don't have have put away POSIX tools in favor of newer ones, like ag, ripgrep, jq, gron, etc but
we will be picking some of those up.

I'll be doing this on MacOS, because the M1's just too nice a laptop, and it runs a certified Unix.
[which is a fun read as to how that happened](https://www.quora.com/What-goes-into-making-an-OS-to-be-Unix-compliant-certified).

This is one third guide, one third rant, and one third lab.
It cover some high level concepts and then we jump into doing and I'll explain along the way.

I'm ornery about technology; the Internet was a mistake, and it's time to give up computers and go live in the woods.

I've tried not to assume any prior kubernetes knowledge, but I can't mind wipe myself and try this guide on myself.


with some tools
then push a service to prod, and 
some random tools I picked up along the way. (Patches welcome!)

Onward!

## **1. What Kubernetes Is (and Isn’t)**  
- **What it is**: an open-source (Apache) An open-source container orchestration platform.

WTF does that mean?

It automates management, scaling, and deployment of a service. It deals with monitoring the health of a service and
restarts if theres a hardware failure.
 
  - Kubernetes orchestrates containerized applications across multiple hosts, manages workloads, and automates repetitive tasks such as scaling, deployments, and health monitoring.
  - It uses a declarative model: you describe your desired state, and Kubernetes ensures the system matches it.
  - Handles service discovery, load balancing, storage orchestration, and resource management for containerized applications.
  - Kubernetes abstracts the underlying infrastructure, whether it's on-premise or cloud, to ensure consistent application behavior.

- **What it isn’t**: A replacement for configuration management tools (e.g., Ansible), PaaS, or simple VMs.  
  - Kubernetes doesn't directly manage your infrastructure or handle OS-level configuration.
  - It isn't a traditional Platform as a Service (PaaS); although it has features that overlap, like scaling and networking, it provides more control but also requires more management compared to PaaS.
  - Kubernetes is not a virtualization platform like VMware or a VM replacement. It focuses on container orchestration, not virtual machines.
  - Opinated on there being one right program for the job. You can uses the ingress controller you want; nginx, trafik, HAProxy, an Amazon ALB, a Google L7LB.
    The container runtime you want; docker, container-d,CRI-O, even FreeBSD jails if you're a psychopath. There are a bunch of different ways to do networking.
    - but only sometimes. etcd KV store is not pluggable. Bastards.  
  - Complete. Just like the unix philosophy of doing one thing well and don't complicate old programs with new features, there is a whole ecosystem 
  of bits of software built on top of Kubernetes that you'll end up using.

---
## **2. History**

First a bit of history. Google's internal cluster management system, Borg, was built to manage Google's workloads across Google's vast infrastructure.
Google took what they learning from there and developed and open-sourced Kubernetes in 2014 to bring this orchestration 
capability to the broader community.

Cynically, Amazon came out with AWS and Google needed a way to compete, so they made, and then open sourced kubernetes to 
commiditize the cloud.
Feel freel to move your entire infrastructure off Amazon and onto GCP since it's all kubernetes anyway.

Anyway, Kubernetes got donated to the Cloud Native Computing Foundation (CNCF) in 2015, which has since fostered its growth as a
critical part of the cloud-native ecosystem.

Kubernetes is now widely considered the de facto standard for container
orchestration, used by companies of all sizes to manage workloads across cloud,
on-premise, and hybrid environments.

As of this writing, Kubernetes is at v1.31.2 or so.
Cloud providers are a bit behind that, but we're not doing anything too complicated.
---

## **3. Starting Terminology **

There's a ton more terminology to learn, but here are the N ones I think are necessary to start.

- **Containers**: Yeah, docker happened. Your service's binary and all its userland crap goes here.
- **Pods**: A collection of one or more containers. It's the smallest deployable unit in Kubernetes.
You'll often have multi-container pods if you have a sidecar for logging or a proxy.
- **Service**: An abstraction that defines a logical set of Pods and a policy by which to access them, 
typically used for load balancing and service discovery. There are 3 types.
  - **ClusterIP**, This exposes a service on a cluster-internal IP. It's only acccessible within the cluster.
  - **NodePort**: Exposes the service on each node's IP at a static port. Allows external traffic but like, ew.
  - **LoadBalancer**: Creates an external load balancer in cloud environments. This is the way.
- **Node**: A worker machine, either virtual or physical, that runs containerized applications.

## **4. Pods, Nodes, and namespaces, oh my!**

So you've written a cute little app and want to give users access to it. In Kubernetes world, you stick it in a container, 
and that container goes in a pod and then you wrap it up in a service.

That runs on a node, segregated by namespace, so you can do your thing in there and be ignorant about your noisy neighbors (more on that later).

---

## **1. Core concepts.**
- **Cluster Architecture**:
  - **Control Plane**: responsible for maintaining the desired state of the system.
    - **API Server**: The front-end to the control plane.
    - **Scheduler**: This is not cron. This assigns work to nodes based on resource availability, with very limited smarts on bin packing.
    - **Controller Manager**: Runs controllers to regulate the state of the cluster. Controllers include the node controller, replication controller, and endpoint controller.
    - **etcd**: distributed /etc for the whole cluster
  - **Worker Nodes**: Machines (physical or virtual) that run containerized applications.
    - **Kubelet**: The primary node agent, it ensures that containers are running in a Pod.
    - **kube-proxy**: Handles network proxying and load balancing for services on the node.
    - **Container Runtime**: AKA Docker, but there are alternatives. (containerd, or CRI-O).

- **Networking**: Don't get me started on ipv6. 
  We need to get packets from your daemon inside the container inside the pod running on the node to provide a service, which then gets exposed
  to the outside world for the Internet to DDos.
  - **Pod-to-Pod Communication**: Each pod gets its own IP address. Pods communicate with each other using this address without NAT, unless you're insane. (There are sometimes reasons to be insane.)
  - **Service Networking**: Services are an abstraction layer so you get a stable IP address and a DNS names for accessing Pods. This simplifies the communication between different components of an application.
  - **Network Policies**: Rules to control traffic between Pods or between Pods and other network endpoints. This ensures secure communications and limits traffic paths as needed.
  - **CNI Plugins**: Kubernetes relies on Container Network Interface (CNI) plugins (e.g., Calico, Flannel, Weave) to implement networking capabilities and configure network routes within the cluster.

- **DNS**: Kubernetes has an internal DNS service that allows Pods and Services to communicate with each other by name instead of IP addresses.
  - Every Service created in Kubernetes is assigned a DNS name, making it easy to refer to services within your cluster.
  - The DNS server in Kubernetes resolves these service names to the corresponding ClusterIP address, simplifying the management of network communications.

- **Ingress**: A resource for managing external access to services within a cluster, typically HTTP/HTTPS.
  - **Ingress Controller**: A special controller that manages routing rules defined by Ingress resources and provides load balancing, SSL termination, and name-based virtual hosting.
  - Ingress allows you to define how external traffic reaches your services, providing more sophisticated routing than a simple Service (e.g., path-based or host-based routing).

- **Certificate Authorities (CA)**: After Snowden, we're obviously we're not going to send packets in the clear between nodes, even if the nodes are next to each other inside our secure DC.
  - **Custom CA**: In order for this to work, we create our own CA and stick that on all the nodes so we can TLS everything.
  - **TLS Secrets**: Kubernetes Ingress uses **TLS Secrets** to store the certificate and private key used for SSL/TLS. You create these secrets to enable encrypted communications between your services and clients.
  - **Cert-Manager**: Cert-Manager is an add-on that helps manage certificates within Kubernetes. It can automatically request and renew certificates from Certificate Authorities (CAs) like Let's Encrypt, reducing manual intervention and ensuring your cluster's certificates are always valid.

- **Persistent Storage**: Containers are stateless but we need to store state somewhere. Your database might be external to your kube cluster, or it might not.
  - **Persistent Volumes (PV)**: A storage resource in the cluster that has been provisioned by an administrator or dynamically created. It abstracts details of how storage is provided, making it possible to work with different types of storage (e.g., NFS, cloud storage).
  - **Persistent Volume Claims (PVC)**: A request for storage by a user. PVCs consume PV resources, providing a mechanism to manage storage allocation and ensure that Pods have consistent and durable storage as needed.
  - PVs and PVCs enable stateful applications to maintain their data across restarts, ensuring data persistence and availability.

- **Service Accounts and Authentication**:
  - **Service Accounts**: A service account provides an identity for processes that run in a Pod, allowing them to interact with the Kubernetes API. Service accounts are typically used by Pods to authenticate to the cluster and perform actions such as accessing resources or interacting with other services.
  - **Authentication and Authorization**: AuthN and AuthZ. Kubernetes has mechanisms to ensure only authenticated users and processes can interact with the cluster.
    - **Authentication**: Kubernetes supports multiple authentication methods, including certificates, bearer tokens, and external identity providers (e.g., OIDC).
    - **Authorization**: Once authenticated, Kubernetes determines what the user or process can do via Role-Based Access Control (RBAC). Roles and ClusterRoles define sets of permissions, and RoleBindings associate users or service accounts with these roles.

- **Key Objects**: I brought some of these up before but here's a bit more detail on them.
  - **Pod**: The smallest deployable unit in Kubernetes, representing a single instance of a running process.
    - **Multi-Container Pods**: Can run multiple containers that share the same network and storage. Often used for sidecar patterns (e.g., logging, proxy).
  - **Service**: An abstraction that defines a logical set of Pods and a policy by which to access them, 
typically used for load balancing and service discovery. There are 3 types.
    - **ClusterIP**, This exposes a service on a cluster-internal IP. It's only acccessible within the cluster.
    - **NodePort**: Exposes the service on each node's IP at a static port. Allows external traffic but like, ew.
    - **LoadBalancer**: Creates an external load balancer in cloud environments. This is the way.
- **Deployment**: Manages a set of identical Pods, allowing you to roll out updates, rollbacks, and scaling easily.
    - **ReplicaSets**: Ensures the specified number of Pod replicas are running at all times.
    - **Rolling Updates**: Deployments support rolling updates to avoid downtime during updates.
  - **ConfigMap/Secret**: Externalize application configuration.
    - **ConfigMap**: Stores non-sensitive configuration data, such as environment variables.
    - **Secret**: Similar to ConfigMap but designed to hold sensitive information like passwords or API tokens, and data is base64 encoded for additional security.
---

## **5. Why Kubernetes?**

- **NIH is the devil** You're not too proud to use someone else's code. Because, you're a believer that NIH is the devil, realize that reinventing the wheel poorly is a waste of everyone's time (especially yours, and you really hate that)
and understand that by the time that you've build a system with a reverse proxy with a load balancer, blue/green deployment, autoscaler, service discovery mechanism, health monitoring system
that, [dear friend, you have built a kubernetes](https://www.macchaffee.com/blog/2024/you-have-built-a-kubernetes/)

- **Portability**:  Because all the clouds have Kube, you can use the expertise you have with kube at your next job that uses kube, instead of
having a Rosetta stone to match AWS primitives to GCP to Azure to OCI's naming.

Kubernetes offers a powerful way to run and manage your containerized workloads at scale, but there's more to it than just containers and automation. Here's why Kubernetes has become the industry standard for deploying, scaling, and managing containerized applications:

- **Scalability**: One of its entire reasons for being. Kube will add Kubernetes is built to scale. It can automatically adjust the number of running instances of your application based on demand using **Horizontal Pod Autoscaling**. You define resource thresholds (like CPU and memory), and Kubernetes scales your application up or down to meet those needs, ensuring cost-effective resource utilization.

Kubernetes abstracts away the underlying infrastructure, meaning it doesn't matter whether you're running on Google Cloud, AWS, Azure, or even on-premises. Kubernetes gives you the flexibility to run workloads across any environment, making it easy to switch providers or use a multi-cloud strategy without vendor lock-in.

- **Self-Healing**: Kubernetes is highly available (HA!) and designed with resilience in mind. So restart the Container if it crashes, plus a crash loop detector to not spin endlessly.
If a node goes down, mark it as bad and schedule replacement jobs on healthy nodes.

- **Declarative Configuration**: Kubernetes uses YAML or JSON to describe the desired state of the system.
This means you tell Kubernetes what you want (e.g., "I want 5 replicas of this service running") rather than
how to do it. Kubernetes takes care of the rest to ensure your application's desired state matches reality, and
it tracks changes over time, making it easier to manage infrastructure as code.
This is the source of all the suck with Kubernetes.
You'll end up encoding JSON in YAML or YAML in JSON at some point if you look into the void too long.

- **Service Discovery and Load Balancing**: Kubernetes automatically assigns IP addresses and DNS names to services, 
making it easy to discover other services. It also distributes traffic across multiple instances of a service, 
providing load balancing out of the box. This means you don't need an external load balancer or manual configuration
to achieve redundancy and scalability.

- **Secrets and Configuration Management**: hunter2
Kubernetes provides built-in ways to manage sensitive information and 
configuration through **Secrets** and **ConfigMaps**. These tools enable you to decouple configuration from the 
application code, making deployments more flexible and secure.

- **Rolling Updates and Rollbacks**: Of course, you would never write code with bugs in it, so this is largely theoretical.
We still don't want downtime in prod. How many nines can we get!?
Kubernetes enables **rolling updates** to minimize downtime during upgrades by incrementally updating 
instances of your application. If something goes wrong, you can **rollback** to the previous stable version easily.

- **Ecosystem and Extensibility**: Everybody's doing it, don't you wanna be cool?
Kubernetes isn't just a platform, it's an entire ecosystem.
Tools like **Helm** (a package manager), **Kustomize** (for configuration management), and 
others extend Kubernetes' capabilities, making it a one-stop-shop for all your deployment and infrastructure needs. 
You can also extend Kubernetes using **Custom Resource Definitions (CRDs)** to add new features or integrate with 
existing systems.

- **Multi-Tenancy and Resource Isolation**: Kubernetes allows you to partition a cluster using **Namespaces**. 
Namespaces enable you to divide cluster resources between different teams or projects, providing logical isolation 
without the need for completely separate clusters. **Role-Based Access Control (RBAC)** ensures that users or 
services only have the permissions they need, improving security and control.

- **Built for Cloud-Native Development**: Yeah I cringed at having to use the phrase Cloud Native, but there's no escaping it.
Kubernetes encourages cloud-native development patterns such as 
microservices. Rather than building monolithic applications, Kubernetes makes it easier to run small, 
focused services that interact with each other. This architecture makes resumes more tasty, systems more resilient, maintainable, and scalable.

- **Community and Support**: There's a bunch of companies using it, so there's a bunch of resources out there for you.
Somehow we've managed to escape the problem of having to pick teams, like what's your power tool color? (Also XKCD 927)
The community has likely faced your challenge before and there's open-source tools, plugins, and community-supported practices, 
making Kubernetes a constantly evolving, well-supported platform.

In summary, Kubernetes helps you take full advantage of cloud and containerization technologies by abstracting infrastructure complexities, automating operational tasks, and providing a flexible, scalable framework for modern applications. It's the foundation for deploying resilient, scalable services in production environments.
---
## **6. Labwork
### **Setup
1. Install kubectx
  1. brew install kubectx
1. Install [kns]
  1. brew tap blendle/blendle
  1. brew install kns

1. Update PS1 to have the currently selected cluster and namespace with [starship.rs](http://starship.rs)
  1. PS1 prompt
  You'll want to update your PS1 to display your current namespace, and if you deal with multiple kube clusters, which one you've
currently got selected.
1. Install a docker runtime.
  1. I sold out and am on MacOS. If you're also on a Mac, I use colima which runs a VM.
    1.  ```bash
    brew install colima
    ```
    1.  ```bash
    colima start --network-address=true
    ```

1. Install [Kind](https://kind.sigs.k8s.io/).  
Per its homepage: 

> kind is a tool for running local Kubernetes clusters using Docker container “nodes”.
kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

This makes it easy to create a cluster, play with it, break it, tear it down and start fresh.

  1. brew install kind

1. Setup some aliases:
     ```bash
    alias k=kubectl
    alias kpods='kubectl get pods'
    alias k1pod='kubectl get pods | tail -n +2 | head -1| awk '{print $1}'

1. **Local**: 
   - Example:  
     ```bash
     kind create cluster
     ```
     
     ✨
1. Welcome to your first kubernetes cluster!
Since we're using kind, we can run
```bash
 docker exec -it kind-control-plane bash
```
to get a shell in our cluster, and then run 
```bash
ps auxwf
```
Whew, that's a bunch of stuff! What the hell is it all doing?


add see there's a bunch of stuff running.

1. **Create a namespace**
     ```bash
     k create namespace foo
     ```
  1.     Switch to that namespace:
     ```bash
     kns foo
     ```
1. **Deploy hello-world app**

  Do you trust me? Hell no you shouldn't, but if you're insane enough to just run
     ```bash
     curl   https://raw.githubusercontent.com/fragmede/kubernetes-for-greybeards/refs/heads/master/hello-world.yaml
 | k apply -f -
```
you can skip the next couple steps.

This will deploy hello world to your cluster.

For those of you who aren't going to just run that, let's take the slow road there.

Clone this repo, poke around, build the docker container
which includes the source to a trivial hello world written in go.
Feel free to use it, or adapt it to your preferred language.

     ```bash
git clone http://github.com/fragmede/kubernetes-for-greybeards.git
cd kubernetes-for-greybeards/hello-world
cat hello.go
cat Dockerfile
docker build -t hello .
# Run the built container to verify it worked:
docker run  --rm --network=host hello
# In another terminal:
curl localhost:8080
# Copy the image to our kind cluster
kind load docker-image hello
```

With the image loaded, we can now do 
```bash
k apply -f hello-world.yaml
```

After that, do `kubectl get pods`
You should see something like
```bash
NAME                                      READY   STATUS    RESTARTS   AGE
hello-world-deployment-55d56654d8-4lwxj   1/1     Running   0          10s
hello-world-deployment-55d56654d8-kmvpv   1/1     Running   0          9s
```

Now in a terminal, run: `kubectl port-forward service/hello-world 1025:80`
and in another, run `curl localhost:1025`. It should connect and return a message. Huzzah!


In another, run `kubectl logs -f hello-world-deployment-55d56654d8-4lwxj` and run curl again.
You should see it log that it got connected to.

Okay great, but that was a lot of bullshit to get that!
Yes it is, but now we're serving it via a pod, and it's highly available.
We can shoot one of those pods in the head and it'll keep going.

``` bash
k delete pod hello-world-deployment-55d56654d8-4lwxj
```
If you run `k get service`, you'll get 
``` bash
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
hello-world   NodePort    10.96.78.233   <none>        80:30001/TCP   73m
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP        78m
```
but don't get excited, those IPs aren't reachable.

Looking at hello-world.yaml, we have a couple of key pieces to understand.

grep 

 
cat 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
  labels:
    app: hello-world
spec:
  selector:
    matchLabels:
      app: hello-world
  replicas: 2 
> foo.yaml
```


Let's configure ingress:

Destroy the kind cluster with
`kind delete cluster`
and create a new one with a few additional settings:
```bash
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

Then install nginx as an ingress controller:

```bash
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
```




What can you do with this?



---
## **6. Hands-On Basics**
### **Install Kubernetes**

     
2. **Cloud**: Each of the cloud providers have managed solutions to make your life easier. You'd better have a good reason to not just
use theirs if you're on a cloud. Use managed solutions like GKE, AKS, or EKS.



### **kubectl Cheat Sheet**
- Deploy an app:  
  ```bash
  kubectl apply -f deployment.yaml
  ```
- View resources:  
  ```bash
  kubectl get pods,svc
  ```
- Debug:  
  ```bash
  kubectl logs <pod-name>
  kubectl exec -it <pod-name> -- /bin/bash
  ```
  This only runs in the pod, which is not the container, which is not the node.

### **Minimal YAML Example**
- `deployment.yaml`:  
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:1.21
          ports:
          - containerPort: 80
  ```

- Deploy it:  
  ```bash
  kubectl apply -f deployment.yaml
  ```

---

## **7. Production-Grade Setup**
- Use a **Managed Kubernetes Service** to simplify setup.
- Implement **Namespaces** for resource isolation.
- Use **Ingress** for routing external traffic.
- Set up **Persistent Volumes (PV)** and **Persistent Volume Claims (PVC)** for storage.
- Add monitoring and logging tools:
  - **Prometheus** + **Grafana** for metrics.
  - **Fluentd** or **ELK stack** for logs.

---

## **8. Best Practices**
- **Resource Requests and Limits**: Avoid resource starvation.  
- **Readiness and Liveness Probes**: Ensure app health.  
- **Immutable Images**: No changes inside running containers.  
- **Version Control**: YAML in GitOps pipelines.  
- **Security**:  
  - Enable RBAC.  
  - Limit container privileges (`runAsUser`, `readOnlyRootFilesystem`).

---

## **9. When to Skip Kubernetes**
- Overkill for small apps or static workloads.
- When resources/skills for operational overhead are unavailable.

---

## **10. Tools for the Greybeard’s Toolkit**
- **Helm**: Helm is a package manager for Kubernetes that simplifies the deployment, management, and versioning of complex applications.
  - **Charts**: Helm packages are called charts, which contain all of the resource definitions needed to run an application or service. Charts can be shared and customized to suit your environment.
  - **Repositories**: Helm charts are stored in repositories, much like packages in Linux distributions. You can create your own repository or use popular public ones like the Helm Hub.
  - **Release Management**: Helm tracks versions of deployed charts, making it easy to upgrade, rollback, or delete Kubernetes resources. This is particularly useful for managing multiple environments (e.g., development, staging, production).
  - **Templating**: Helm allows you to use templating within your YAML files, which makes it easier to reuse and manage complex, dynamic configurations. This reduces duplication and improves maintainability.
  - **Helm Commands**:
    - **Install a Chart**: Deploy an application using a chart with `helm install <release-name> <chart-name>`.
    - **Upgrade a Release**: Apply changes to an existing deployment using `helm upgrade <release-name> <chart-name>`.
    - **Rollback**: Easily roll back to a previous release using `helm rollback <release-name> <revision>`.
- **Kustomize**: Native Kubernetes configuration management.
- **kubectx/kubens**: Quick namespace and cluster switching.
- **Lens**: GUI for managing clusters.

---

## **11. Next Steps**
- Experiment with real-world apps.
- Learn advanced topics:
  - StatefulSets for stateful apps.
  - Horizontal Pod Autoscalers.
  - Custom Resource Definitions (CRDs).

---


## **1. Terminology

Core Kubernetes Components
Pod: The smallest deployable unit in Kubernetes; can contain one or more containers.
Node: A worker machine in Kubernetes, either a physical server or a virtual machine.
Cluster: A collection of nodes managed by Kubernetes.
Control Plane: The components that control and manage the Kubernetes cluster.
Kubernetes API Server: Exposes Kubernetes APIs for interacting with the cluster.
Controller Manager: Runs controllers to manage cluster state.
Scheduler: Assigns pods to nodes based on resource needs and constraints.
etcd: A distributed key-value store for storing cluster configuration and state.
Cloud Controller Manager: Integrates with cloud provider APIs for node management, load balancing, and storage.
Workload Resources
Deployment: Manages the lifecycle of replicated pods and ensures the desired state.
ReplicaSet: Ensures a specified number of pod replicas are running at all times.
StatefulSet: Manages stateful applications, providing stable network identities and persistent storage.
DaemonSet: Ensures a pod runs on all (or specified) nodes in the cluster.
Job: Manages the execution of one-off tasks.
CronJob: Schedules Jobs to run at specified times or intervals.
Networking
Service: Exposes a set of pods as a network service.
ClusterIP: Default service type for internal cluster communication.
NodePort: Exposes a service on a static port on each node.
LoadBalancer: Exposes a service externally using a cloud provider’s load balancer.
Ingress: Manages HTTP and HTTPS access to services.
DNS: Automatically resolves services and pods into DNS names.
Storage
Persistent Volume (PV): Abstracts storage resources in the cluster.
Persistent Volume Claim (PVC): A request for storage by a pod.
StorageClass: Defines different types of storage and allows dynamic provisioning.
ConfigMap: Stores configuration data as key-value pairs.
Secret: Stores sensitive data, such as passwords or tokens.
Scaling and Autoscaling
Horizontal Pod Autoscaler (HPA): Automatically adjusts the number of pod replicas based on resource utilization.
Vertical Pod Autoscaler (VPA): Adjusts resource requests and limits for containers.
Cluster Autoscaler: Automatically scales the number of nodes in a cluster.
Security and Access
Role-Based Access Control (RBAC): Manages permissions for users and resources.
Service Account: Provides an identity for processes running in a pod.
Network Policy: Controls traffic flow between pods.
Pod Security Policy: Defines security-related configurations for pods.
Observability
Labels: Key-value pairs attached to objects for identification and grouping.
Annotations: Key-value pairs used to attach metadata to objects.
Metrics Server: Collects and exposes cluster metrics for monitoring and autoscaling.
Logs: Output from applications or Kubernetes components.
Events: Notifications about changes or issues in the cluster.
Advanced Kubernetes Concepts
Custom Resource Definition (CRD): Extends Kubernetes APIs to define custom objects.
Operator: A controller that manages complex stateful applications using CRDs.
Init Container: A special container that runs before the main application container starts.
Sidecar Container: A helper container that runs alongside the main application container.
Helm: A package manager for Kubernetes applications.
Kubelet: Agent that runs on each node and ensures containers are running as expected.
Kube-proxy: Manages networking rules to allow communication to/from pods.
Deployment Strategies
Rolling Update: Gradually replaces old pods with new ones.
Blue-Green Deployment: Runs two separate environments (blue: current, green: new).
Canary Deployment: Gradually introduces changes to a subset of users before full rollout.
Other Concepts
Namespaces: Logical partitions for isolating resources within a cluster.
Context: A set of access parameters for communicating with a cluster.
Taints and Tolerations: Controls which pods can run on which nodes.
Affinity and Anti-affinity: Defines pod placement preferences.
Ephemeral Containers: Temporary containers for debugging running pods.
- **Daemon**: A background process running in the cluster to provide system-level services.
- **Service Account**: Provides an identity for processes running in a Pod, used for API interactions.


thanks to 
https://iximiuz.com/en/posts/kubernetes-kind-load-docker-image/
