# Pond: CXL-Based Memory Pooling Systems for Cloud Platforms (ASPLOS 2023)

```
Link: https://dl.acm.org/doi/abs/10.1145/3575693.3578835
Authors: Huaicheng Li, Daniel S. Berger, Lisa Hsu, Daniel Ernst, Pantea Zardoshti, Stanko Novakovic, Monish Shah, Samir Rajadnya, Scott Lee, Ishwar Agarwal, Mark D. Hill, Marcus Fontoura, and Ricardo Bianchini.
```

## Summary

Data centers hosting virtual machines (VMs) often suffer from a problem called *memory stranding*. Specifically, resources allocated to the VMs include CPU cores, memory, bandwidth, etc. When there exists unallocated memory but no available cores for a VM, the free (but not allocatable) memory is *stranded*. Memory stranding causes financial and energy waste in cloud platforms. One solution to memory stranding is memory pooling. In memory pooling, different servers are allowed to share the same memory pool from which memory can be dynamically allocated to any server. Since freed memory can be returned to the shared pool, other servers running short of cores can utilize the additional memory from the pool to avoid stranding. However, existing memory pooling systems either introduce access latencies due to expensive page faults or require modification to VM guests. Neither the performance impact nor the implementation complexity is tolerable.

This paper identifies the potential performance improvement opportunities enabled by CXL-based memory, where memory can be accessed without faulting or swapping. It first analyzes the impact of memory pooling on applications running on production clusters at Azure. Results show that a small pool size (number of CPU sockets sharing a pool) can vastly reduce DRAM usage of VMs. The paper then shows 50% of the VMs have 50% untouched memory. It is thus desirable to allocate the active memory locally and untouched memory in the pool. In addition, the paper presents that different applications have different sensitivity to memory access latency caused by pooling. Based on these observations, the authors point out that:

- Memory pooling is useful to save memory
- Proper allocation of VM memory is beneficial to performance

Pond is thus designed with a hardware layer that configures the CXL's topology, a system software layer that supports proper VM resource allocation, and a machine learning model that predicts the optimal allocations of VM memory. The hardware layer leverages an external memory controller (EMC) to support access to CXL ports and designs different connection topology for different number of pooling sockets (8, 16, 32, 64). EMCs expose interrupt interfaces to the system software so that the hypervisor can use them to online/offline the correct amount of memory in the correct cache slice. Since a VM can see the memory pool as zNUMA (CPU-less NUMA) memory, a prediction model is used to determine how much memory should be placed in the pool. The model consists of two parts: 1) the prediction for VM scheduling, i.e., how much memory is deemed untouched based on the historical VM data; and 2) QoS monitoring, which can be used to correct the overprediction of untouched VM memory for sensitive applications.

Pond servers are emulated using two sockets, with one socket's cores being offlined such that memory can be zNUMA. Application traces are used to simulate production VM requests. Experimental results show that dynamically allocating untouched memory on zNUMA node benefits the performance, the combined model they train is accurate (98%, Finding #8, Section 6.4), Pond can further reduce DRAM usage.

## Discussions

Pond presents an implementation of CXL-based pooling evaluated on industrial workload. Its design, including the machine learning module, is fairly simple. The paper attempts to demonstrate the benefits of using CXL in a pooling setting, which I believe very well exist, but with some vagueness. Some questions in my mind after studying their design and evaluation are:

1. The paper assumes (and shows in experiment) that memory pooling can save DRAM usage (Section 3.1 & Figure 3), but it is not clear what it means by "Required Overall DRAM" if I did not miss anything. Applications should run the same amount of memory despite the memory topology, and memory pooling is supposed to allocate all memory statically. How is the DRAM usage measured?
2. Section 4.1 presents hardware design, but there is no evaluation for the hardware. Indeed, the latency of EMCs in Section 4.1 are merely estimations. I understand CXL is not in production, but lack of evaluation undermines the claimed advantage of their design.
3. The paper does not create a strawman that is strong enough to show the advantage of ML. First, the end-to-end experiment chooses a VM where 15% of its memory is allocated in the CXL pool. What about other fixed percentages, or percentage determined by heuristics? Second, there is no evaluation on performance, such as end-to-end VM request latency or throughput. While the paper shows the ML model can guarantee a low PDM with high probability, it does not compare Pond with other approaches. If other approaches can also give that PDM, why use ML?

The paper specifically mentioned avoiding fragmentation in the pool (Section 4.2). Indeed, the granularity of memory pooling is 1GB. I am not sure how the migration is implemented, though it's likely some numactl thing that can only move pages at 4KB granularity. However, this suggests inherent support for huge pages, if the system calls of numactl can be augmented to support huge pages, merely an engineering effort. It also doesn't make any sense to store a subpage in a huge page in the memory pool, because the memory pool is supposed to be large. For these reasons, I still maintain that offloading is more promising than pooling if contiguity is to be combined with work in disaggregated memory.

Another future direction is to combine [TPP](https://dl.acm.org/doi/10.1145/3582016.3582063) with Pond. They use the same technology for different purposes. If some CXL-based memory is plugged in on your node, how much of it will be used for pooling and offloading respectively? Also, can one also offload some VM memory dynamically if they are cold? Will this be better than static prediction? Finally, Meta and Microsoft use different workload to evaluate their work. Will Pond's result change when deployed on Meta's fleet and vice versa?

---

*Created: 2023-05-17; Last updated: 2023-05-17.*