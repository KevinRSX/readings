# TPP: Transparent Page Placement for CXL-Enabled Tiered-Memory (ASPLOS 2023)

```
Link: https://dl.acm.org/doi/10.1145/3582016.3582063
Authors: Hasan Al Maruf, Hao Wang, Abhishek Dhanotia, Johannes Weiner, Niket Agarwal, Pallab Bhattacharya, Chris Petersen, Mosharaf Chowdhury, Shobhit Kanaujia, and Prakash Chauhan
```

## Summary

DRAM is important for performance in data center hyperscaler workloads, especially those requiring large amount of memory. Memory tiering is a solution to the shortage of memory. CXL-based memory enables main memory expansion by providing a cache-coherent interface so that the memory in CXL can be loaded/stored from the CPU with small amount of additional latency. From the software's perspective, when the CXL memory is attached, it can be accessed as a CPU-less NUMA node. However, applications use much smaller amount of memory than they allocate. Some local memory in DRAM might rarely be used (cold), causing a waste of space. Some memory in CXL might be used often (hot) but cannot be efficiently moved back to the local node using current techniques such as NUMA balancing in Linux.

To better understand how applications can benefit from memory tiering, this paper leverages Chameleon, a tool used to characterize the memory behavior of applications, in particular, the frequency heat map of the application memory. Chameleon leverages the Precise Event-Based Sampling (PEBS) mechanism to be notified of the memory access events. It is composed of a collector that collects the event and a worker that sorts the number of events into hash tables. Chameleon's characterization shows the following:

1. A significant amount of memory is cold thus can benefit from memory tiering,
2. File pages are colder than anon pages,
3. Over an application's life time, application's access pattern to page types (anon/file pages) are steady, and
4. Workloads require different page types to achieve better performance.

Based on those observations, the authors propose Transparent Page Placement (TPP). As its name suggests, TPP transparently handles the allocation and movement of pages across the local nodes and the CXL. TPP is [implemented](https://lwn.net/Articles/876993/) as an extension of NUMA memory management in the Linux kernel. TPP includes the following aspects:

1. By default, Linux reclaims memory when the memory is running low on the local node. The reclamation process, when located on the critical path, signficantly impacts the application performance. TPP migrates reclamation candidate before they are swapped out.
2. Although 1 is useful to reduce swapping overhead, if (the modified) reclamation happens too often and the allocation is halted, too much potentially hot memory will be allocated on the CXL node. TPP therefore proactively sets the demotion watermark smaller so that applications will less likely to halt on low memory.
3. Pages in CXL node should be promoted to local if they are hot. The paper points out the default NUMA balancing mechanism promotes immediately when a remote page is sampled, causing Ping-Pong thrashing. I did not know how NUMA balancing is implemented now, but [this presentation in 2014](linux-kvm.org/images/7/75/01x07b-NumaAutobalancing.pdf) already mentioned the notion of a quadratic filter. If that is already in the kernel, what is left for TPP? (I shall return to this part after finishing reading TPP's patch).
4. TPP allows I/O-heavy applications to specify that they wish to allocate file caches on CXL.

TPP is evaluated using an FPGA-based CXL-Memory expansion card attached to CPUs supporting CXL 1.1 specification. Memory shows up as CPU-less remote NUMA node. The authors first evaluate the end-to-end throughput and the local memory usage of Meta's workloads. TPP performs better than Linux in those cases as expected. They also show each of the above components of TPP is important for performance. Finally, they compare TPP to existing solutions including NUMA balancing, AutoTiering, and TMO, all of which result in observable better performance.

## Discussion

According to Section 3, Chameleon "considers the address of a sampled record as a virtual page access where the page size is defined by the OS." The OS can define 2MB or 1GB huge pages as it wishes, potentially reducing the accuracy of Chameleon's characterization. In fact, TPP assumes huge pages are hot and should never be demoted to CXL (Section 4). This claim is not necessarily true for a system free from fragmentation where an application's working set can be all backed by huge pages, e.g., Contiguitas. In that case, 1) not all huge pages are hot, and 2) hotness can be imbalanced within a huge page. Therefore, a more precise measurement of hotness within huge pages is desirable. While the authors claim that [Chameleon is open sourced](https://github.com/facebookresearch/chameleon), the code is still not available. It will be useful if we can play with it to support different granularity.

The comparison with NUMA balancing is very confusing. I do not understand why NUMA balancing has larger overhead if TPP merely extends some of its features. One way to explain it is NUMA balancing promotes so much hot memory from CXL to local node that the memory pressure on local node is always high. Promotion will then fail and the sampling and page faults become a waste (Section 6.3.1). This does not sound like the full story to me.

Overall, to my knowledge, TPP is the first paper that discusses how to properly offload memory to CXL. The design of TPP is decent, making it a good candidate to be upstreamed to the kernel. The results given by Chameleon are expected but important. Optimizations are expected to emerge in the future. For example, how physical contiguity with huge pages can come into play is important and worth working on.

---

*Last updated: 2023-05-16.*

