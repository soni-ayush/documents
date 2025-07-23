# Kubernetes

## Resource Management: CPU and Memory
Kubernetes allows you to manage compute resources for your containers using resource requests and limits for CPU and memory. These settings help ensure fair scheduling and prevent resource contention or overuse.

- **CPU and Memory Requests:** Minimum resources guaranteed for a container.
- **CPU and Memory Limits:** Maximum resources a container can use.

For a detailed explanation of how Kubernetes manages CPU and memory resources, including how Linux cgroups enforce these limits and what happens when limits are exceeded, see:

- [Kubernetes Resource CPU and Memory](pod_resource_cpu_memory.md)
