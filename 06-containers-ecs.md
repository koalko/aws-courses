# Containers & ECS

## Introduction to containers

In a traditional virtualization approach Guest OS takes up a lot of space and resources in each VM. Containerization also provides an isolated environments (filesystem, processes), but container runs as a process in the Host OS; so, different containers use the same Host OS as an operating system. So, containers are significantly lighter than VMs.

Container is a running instance of Docker image. Image is stack of layers. Docker images are created from `Dockerfile`s. Each step in a `Dockerfile` creates a read-only filesystem layer. Images are created from a base image or from scratch. Docker container has an additional read/write filesystem layer.

Container Registry (CR) is a registry of container images. Most popular CR is Docker Hub.

Actually, "Docker" is just one type of the container, but the most popular one.

### Tips

- `Dockerfile`s are used to build images
- containers are portable: self-contained, always run as expected
- containers are lightweight: parent OS used, filesystem layers are shared
- container only runs the application and environment it needs
- container provides much of the isolation VM do
- ports are exposed to the host and beyond
- application stacks can be multi-container

### Some Docker-related commands

```sh
sudo dnf install docker # install Docker on AL
sudo service docker start # start Docker service
sudo usermod -a -G docker ec2-user # add ec2-user to a Docker group (so sudo won't be needed)

docker build -t image-name . # build the Docker image `image-name` from the current directory
docker images # list all the local Docker images
docker images --filter reference=image-name # see image list filtered by `image-name`
docker run -t -i -p 80:80 image-name # run the Docker container from the `image-name` image exposing 80 port

docker login --username=docker-hub-username # login to Docker Hub using `docker-hub-username`
docker tag image-id docker-hub-username/image-name # tag the image `image-id` as `docker-hub-username/image-name`
docker push docker-hub-username/image-name:latest # push the Docker image into the Docker Hub
```

## Elastic Container Service (ECS) – Concepts

AWS-managed infrastructure to run containers. ECS to containers is that EC2 is to VMs. ECS uses clusters which run in one of two modes:
- EC2 (EC2 instances used as a container hosts)
- Fargate (serverless mode)

ECS lets you create a cluster, from where your containers are running from. ECS uses:
- **container definition**
  - where the container image is (including the container registry)
  - which port the container uses
- **task definition** to describe the (potentially multi-container) application:
  - container(s)
  - resources, used by the task (CPU & memory)
  - networking mode
  - compatibility (EC2/Fargate)
  - task role: IAM role, which task can assume; used to give containers permissions to access AWS resources
- **service definition** to describe how to scale the task

## ECS - Cluster Mode

Common components:
- Scheduling and Orchestration
- Cluster Manager
- Placement Engine

### EC2 mode

EC2 instances used to run containers. Autoscaling Group (ASG) used for horizontal scaling. You are still paying for EC2 instances even if no containers are running on them.

### Fargate mode

ECS tasks are running on an AWS-managed shared infrastructure. Each ECS task is injected into your VPC and has an ENI. You only pay for the containers you are using based on the resources they consume. You have no visibility of underlying instances/hosts.

### Choosing between ECS (EC2) and Fargate

In general, you should use `ECS` in either mode (over simply using `EC2`) if you use containers. Choosing between modes:
- `ECS (EC2)`
  - large workload and price conscious (use spot instances etc.)
- `Fargate`
  - large workload and overhead conscious
  - small/burst workloads
  - batch/periodic workloads

## Elastic Container Registry (ECR)

ECR is a managed container image registry service (like Dockerhub, but for AWS). Each AWS accounts has a public and a private registry. Inside each registry you can have many repositories. Each repository can contain many images. Images can have several tags (unique within the repository). Public registry: public R/O; R/W requires permissions. Private registry: both R/O and R/W requires permissions.

Benefits:
- integrated with IAM
- image scanning (for OS or app issues): basic and enhanced
- near real-time metrics, delivered to CloudWatch (auth, push, pull)
- API actions are logged into CloudTrail
- generates events delivered into EventBridge
- cross-region and cross-account replication

## Kubernetes (K8S) 101

Open-source container orchestration system. Used to automate the deployment, scaling and management of containerized applications. K8S is a cloud-agnostic product.

### Cluster Structure

Usually K8S operates in a highly available cluster of compute resources which are organized to work as one unit.

`Cluster Control Plane` manages the cluster, scheduling, applications, scaling and deploying.

`Cluster Node` – VM or physical server which functions as a worker in the cluster.

Each Node runs:
- `containerd` or another container runtime for handling container operations.
- `kubelet` – agent to interact with the cluster control plane via the K8S API.
- `kube-proxy` – network proxy. Coordinates networking with the control plane. Helps to implement services and configures rules allowing communication with pods from inside or outside of the cluster.

`Pods` are the smallest units of compute in K8S. Pods are **non-permanent**, temporary. "One-container-one-pod" architecture is very common; multiple containers per pod usually used when these containers are tightly coupled.

Some important Control Plane components:
- `kube-apiserver`: the frontend for the K8S control plane is  (nodes and other cluster elements are interacting with it). Can be horizontally scaled for HA and performance.
- `etcd`: provides a highly-available key value store used within the cluster. It's used as the main backing store for the cluster.
- `kube-scheduler`: identifies any pods within the cluster with no assigned node and assigns a node based on resource requirements, deadlines, affinity/anti-affinity, data locality and any constraints.
- `cloud-controller-manager` (optional): provides cloud-specific control logic. Allows to link K8S with a cloud providers APIs (AWS/Azure/GCP).
- `kube-controller-manager`: cluster controller processes
  - `Node Controller`: monitoring and responding to node outages
  - `Job Controller`: one-off tasks (jobs) => pods
  - `Endpoint Controller`: populates endpoints (services <=> pods)
  - `Service Account & Token Controllers`: accounts / API tokens

### Summary
- `Cluster` – a deployment of K8S, management, orchestration, etc.
- `Node` - resources; pods are placed on nodes to run
- `Pod` – 1+ containers; smallest unit in K8S; often 1 container per 1 pod
- `Service` – abstraction (roughly equal to application); service running on 1 or more pods
- `Job` – ad-hoc; creates one or more pods until completion
- `Ingress` – exposes a way into a service (`Ingress` => `Routing` => `Service` => 1+ `Pod`s)
- `Ingress Controller` – used to provide ingress (e.g. AWS LB Controller uses ALB/NLB)
- `Persistent Storage` (`PV`) – volume whose lifecycle lives beyond any 1 pod using it

## Elastic Kubernetes Service (EKS) 101

AWS' implementation of K8S as-a-service. Control Plane scales and runs on multiple AZs. Integrates with AWS services (ECR, ELB, IAM, VPC). `EKS Cluster` = EKS Control Plane + EKS Nodes. `etcd` is distributed across multiple AZs.

Nodes management options:
- self-managed (EC2-based)
- managed node groups (EC2-based)
- Fargate pods
It is important to check node type when you have extra requirements (Windows, GPU, Inferentia, Bottlerocket, Outposts, Local Zones, etc).

Storage providers include EBS, EFS, FSx Lustre, FSx for NetApp ONTAP.

Typical EKS architecture:
- AWS-managed VPC for Control Plane (multi-AZ)
- Customer-managed VPC for Worker Nodes

## Quiz

- What is the best practice way of providing permissions to running containers on ECS
  - Task roles
- Which mode of ECS should be used if you want as little admin overhead as possible
  - Fargate
- What part of ECS is used to configure scaling and HA for containers
  - Service
- What cluster modes are available within ECS
  - Network Only (Fargate)
  - EC2 Linux + Networking
  - EC2 Windows + Networking
- What container standard does ECS support
  - Docker
- What does a container registry do?
  - Store container images
- What are the advantages/benefits of containers (Choose 3)
  - Fast to startup
  - Portable
  - Lightweight