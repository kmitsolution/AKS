# Inside Worker node -AKS
<img width="499" alt="image" src="https://github.com/kmitsolution/AKS/assets/84008107/38bf7df5-7be4-4861-ba26-4230356622c9">

Inside a worker node in an Azure Kubernetes Service (AKS) cluster, several components are typically present to facilitate the execution of Kubernetes workloads:

1. **Container Runtime**: This is the software responsible for running containers on the node. In AKS, Docker was commonly used as the container runtime, but starting with Kubernetes version 1.20, containerd is used by default.

2. **kubelet**: The kubelet is an agent that runs on each node in the cluster. It is responsible for ensuring that containers are running in a Pod. It communicates with the Kubernetes control plane to manage the Pods assigned to the node.

3. **kube-proxy**: The kube-proxy is a network proxy that runs on each node in the cluster. It maintains network rules (iptables in Linux) required to implement Kubernetes Service abstraction. These rules allow network communication to your Pods from network sessions inside or outside of your cluster.

4. **Pod Infrastructure**: Each node has infrastructure components necessary for running Pods, such as network namespaces, the container runtime environment, and mounts for shared volumes.

5. **Networking**: Networking components facilitate communication between Pods running on the node and between nodes in the cluster. This includes CNI (Container Networking Interface) plugins responsible for managing Pod networking and providing network connectivity to Pods.

6. **Operating System**: The worker node runs on a specific operating system (commonly Linux distributions such as Ubuntu or CentOS) where all the Kubernetes components are deployed.

7. **Monitoring and Logging Agents (Optional)**: Depending on your configuration, you may have monitoring and logging agents installed on the worker node. For example, Azure Monitor for containers can be used to collect telemetry data from containers running in the cluster.

8. **Security Agents (Optional)**: Security agents such as Azure Security Center agents or other endpoint protection solutions may be installed on the node to provide additional security features.

These components work together to execute and manage containers and Pods within the AKS cluster. Understanding these components is essential for troubleshooting, optimizing performance, and ensuring the proper operation of your AKS cluster.
