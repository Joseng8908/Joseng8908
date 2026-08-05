### Focus & Stack

- **Domain:** AI Datacenter Infrastructure — distributed inference serving, GPU cluster orchestration
- **Core Stack:** Go, C, Python, Linux Internals, Kubernetes, Docker
- **Working on:** Large-scale LLM serving on multi-GPU nodes — tensor/pipeline/expert parallelism, KV cache tuning, interconnect topology analysis
- **Interested in:** Kubernetes SIG-Scheduling, Kueue, vLLM internals, multi-node inference

**Computer Science and Engineering @ SeoulTech**

---

## Tech Stacks

![C](https://img.shields.io/badge/C-A8B9CC?style=flat-square&logo=C&logoColor=white)
![Go](https://img.shields.io/badge/Go-00ADD8?style=flat-square&logo=Go&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=Python&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=Linux&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=Docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=Kubernetes&logoColor=white)
![NVIDIA](https://img.shields.io/badge/NVIDIA-76B900?style=flat-square&logo=NVIDIA&logoColor=white)
![vLLM](https://img.shields.io/badge/vLLM-1B1B1B?style=flat-square&logoColor=white)
![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=Git&logoColor=white)
![Backend.AI](https://img.shields.io/badge/Backend.AI-2A2D34?style=flat-square&logoColor=white)
---

## Projects

### Kubernetes SIG-Scheduling

**[chijik-scheduler](https://github.com/Joseng8908/chijik-scheduler/tree/release-1.29/pkg/plugins)** — Custom Kubernetes scheduler
Bandwidth-aware scoring plugin built on the scheduler framework (Filter / Score / Bind).
Explores topology-aware placement for accelerator workloads.

### Container Runtime

**[micro-container](https://github.com/Joseng8908/go_programming/tree/main/micro-container)** — Container runtime from scratch
Namespace, cgroup, and pivot_root implementation in Go. Built to understand what `docker run` actually does.

---

## Experience

**AI Datacenter / HPC Research — Research Intern**

- Deployed a multi-node GPU cluster management stack (manager, agent, storage proxy, NFS) across H100 and Blackwell nodes
- Served large MoE models (700B+ total params, FP4/FP8 quantized) with tensor / pipeline / expert parallelism
- Diagnosed interconnect bottlenecks via GPU topology and NUMA boundary analysis; validated InfiniBand and GPUDirect RDMA paths
- Tuned RAG serving pipeline (search backend, web loader timeouts, chunking) against measured retrieval accuracy
