---
layout: post
title: "Virtualization and Caches in RISC-V"
categories: RISC-V Virtualization
---

# Concepts

## Virtually Indexed Physically Tagged Cache
The cache can be indexed using the virtual address, and verified using the physical address when the translation is ready.

## Two-Staged Address Translation
![2D Page Table Walk](/images/Virtual-Memory-and-Caches/2DPageTableWalk.png)
[[3]](#references)

## Page Walk Caches (PWC) and Nested TLB (NTLB)
![PWC and TLB](/images/Virtual-Memory-and-Caches/2DPTW_Caches_NTLB.png)
[[3]](#references)

## The Sv39 Page-Based 39-bit Virtual Memory System
Rocket Chip implements the Sv39 VM scheme [[2]](#references).
![Sv39VMAddressMap](/images/Virtual-Memory-and-Caches/Sv39VMAddressMap.png)
A page is usually 4KB and byte-addressable, so the offset is 12 bits ($2^{12} = 4096$).

The virual page number (VPN) and physical page number (PPN) are all partitioned into three parts. This means that the page walk cache (PWC) from gVA to gPA would have two levels. Similarly, the PWC from gPA to hPA would have two levels as well.

The PWC for gVA to gPA translation uses prefixes of gVA to skip to gPA that points to a page table.

The PWC for gPA to hPA translation is the second stage, uses prefixes of gPA to skip to hPA that points to a page table.

## Given a virtual address, how to make use of translation cache?
For a 5-level page table:
1. If the top $(n)$ bits match, then the translation result can be directly obtained (TLB).
2. If the top $(n-9)$ bits match, then we can go to the last level of page table directly.
3. If the top $(n-2*9)$ bits match, then we go to the second last level of page table.
4. If the top $(n-3*9)$ bits match, thn we go to the third last level.
5. Otherwise, use the top $(n-4*9)$ bits to index into the top most level of page table.

The cached value is always host physical address (hPA).

## Rocket-Chip: Two Level TLB with Extended Entries
Rocket chip implements two level of TLB, similar to two level of caches [[1]](#references).
Source code at `rocket-chip/src/main/scala/rocket/PTW.scala`.
Rocket chip caches translations. (pte_cache and s2_pte_cache).
Rocket chip stores not only the gVA to hPA translation in TLB, but also the gVA to gPA translations [[4]](#references).

![RocketChipAddressTranslation](/images/Virtual-Memory-and-Caches/RocketChipAddressTranslation.png)

# References:
1. N. C. Papadopoulos, V. Karakostas, K. Nikas, N. Koziris and D. N. Pnevmatikatos, "A Configurable TLB Hierarchy for the RISC-V Architecture," 2020 30th International Conference on Field-Programmable Logic and Applications (FPL), Gothenburg, Sweden, 2020, pp. 85-90, doi: 10.1109/FPL50879.2020.00024. URL: https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9221630&isnumber=9221504

2. "The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Document Version 20211203", Editors Andrew Waterman, Krste Asanovi´c, and John Hauser, RISC-V International, December 2021. URL: https://riscv.org/technical/specifications/

3. Ravi Bhargava, Benjamin Serebrin, Francesco Spadini, and Srilatha Manne. 2008. Accelerating two-dimensional page walks for virtualized systems. In Proceedings of the 13th international conference on Architectural support for programming languages and operating systems (ASPLOS XIII). Association for Computing Machinery, New York, NY, USA, 26–35. https://doi.org/10.1145/1346281.1346286

4. Sá, B., Martins, J., and Pinto, S., “A First Look at RISC-V Virtualization from an Embedded Systems Perspective”, <i>arXiv e-prints</i>, 2021. doi:10.48550/arXiv.2103.14951.

# Appendix

## Adding AutoCounters to Rocket Chip
Import `midas.targetutils.PerfCounter`, then use `PerfCounter.apply(UInt, LabelName, HumanReadableDescription)`. `LabelName` must be globally unique and contain no space.

## Dcache Grant, Acquire and Release

What is dcache grant, dcache acquire and dcache release? (RISC-V)

ChatGPT:

In the context of the RISC-V architecture, the terms "dcache grant," "dcache acquire," and "dcache release" are related to the cache coherence protocol and memory synchronization operations. These terms specifically refer to the handling of cache ownership and synchronization between different cache levels.

Dcache Grant: In a multi-level cache hierarchy, when a cache line is modified in one cache, it becomes the "owner" of that cache line. The dcache grant operation refers to the process of granting ownership of a cache line to a particular cache level. It typically involves transferring the ownership and associated data from one cache level to another.

Dcache Acquire: The dcache acquire operation is used to acquire ownership of a cache line from a lower-level cache. When a cache level needs to read or modify a cache line that is owned by a lower-level cache, it performs the dcache acquire operation to obtain ownership of that cache line. This ensures that the cache line is up-to-date and consistent across all cache levels before performing any operations on it.

Dcache Release: The dcache release operation is used to release ownership of a cache line. When a cache level has finished accessing or modifying a cache line, it releases ownership of that cache line using the dcache release operation. This allows other cache levels to acquire ownership and perform operations on the cache line as needed.

These operations are part of the cache coherence protocol implemented in the RISC-V architecture to maintain data consistency across different levels of cache. They ensure that cache copies of the same data are kept coherent and synchronized, preventing data inconsistencies and ensuring correct program execution in multi-core or multi-processor systems.