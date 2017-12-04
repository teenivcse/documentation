#### Official Definition - What is Kubernetes?

>  "Kubernetes is an open-source system for automating deployment, scaling, and management of containerized applications."

#### Introduction - Basic Principle

The foundation of _**Kubernetes Clusters as a Service**_ is Kubernetes itself, because Kubernetes is the go-to solution to manage software in the Cloud, even when it's Kubernetes itself (see also OpenStack which is provisioned more and more on top of Kubernetes as well).

While self-hosting, meaning to run Kubernetes components inside Kubernetes, is a popular topic in the community, we apply a special pattern catering to the needs of our cloud platform to provision hundreds or even thousands of clusters. We take a so-called "seed" cluster and seed the control plane (such as the API server, scheduler, controllers, etcd persistence and others) of an end-user cluster, which we call "shoot" cluster, as pods into the "seed" cluster. That means one "seed" cluster, of which we will have one per IaaS and region, hosts the control planes of multiple "shoot" clusters. That allows us to avoid dedicated hardware/virtual machines for the "shoot" cluster control planes. We simply put the control plane into pods/containers and since the "seed" cluster watches them, they can be deployed with a replica count of 1 and only need to be scaled out when the control plane gets under pressure, but no longer for HA reasons. At the same time, the deployments get simpler (standard Kubernetes deployment) and easier to update (standard Kubernetes rolling update). The actual "shoot" cluster consists only out of the worker nodes (no control plane) and therefore the users may get full administrative access to their clusters.

#### Setting The Scene - Components and Procedure

We provide a central operator UI, which we call the "Gardener". It talks to a dedicated cluster, which we call the "Garden" cluster and uses CRDs ([CustomResourceDefinitions](https://kubernetes.io/docs/concepts/api-extension/custom-resources/#customresourcedefinitions), the general extension concept of Kubernetes) to represent "shoot" clusters. In this "Garden" cluster runs the "Garden Operator", which is basically a Kubernetes controller that watches the CRDs and acts upon them, i.e. creates, updates/modifies, or deletes "shoot" clusters. The creation follows basically these steps:
* Create a namespace in the "seed" cluster for the "shoot" cluster which will host the "shoot" cluster control plane
* Generate secrets and credentials which the worker nodes will need to talk to the control plane
* Create the infrastructure (using [Terraform](https://www.terraform.io/)), which basically consists out of the network setup and the scaling group definition (including the cloud config, so that the IaaS can create the virtual machines headless whenever a node fails or new nodes are requested by the cluster auto-scaler)
* Deploy the "shoot" cluster control plane into the "shoot" namespace in the "seed" cluster
* Wait for the "shoot" cluster API server to become responsive (VMs will be automatically provisioned without our intervention, pods will be scheduled, persistent volumes and load balancers are created by Kubernetes via the respective cloud provider)
* Finally we deploy `kube-system` daemons like `kube-proxy` and further add-ons like the `dashboard` into the "shoot" cluster and the cluster becomes active

#### Overview Architecture Diagram

![Missing](https://github.com/gardener/gardener-docs/blob/master/images/tam-block-diagram-overview.png)

Note: While the `kubelet` talks through the front-door (public Internet) to its "shoot" cluster API server running in the "seed" cluster, the pods inside the "shoot" cluster use the Kubernetes service and its load balancer IP to reach the API server. This communication and the reverse communication from the API server to the pod and service  IPs happens through a VPN connection that we deploy into "seed" and "shoot" clusters.
