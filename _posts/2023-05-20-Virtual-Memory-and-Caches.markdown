---
layout: post
title: "Virtualization and Caches in RISC-V"
categories: RISC-V Virtualization
---

## Given a virtual address, how to make use of translation cache?
1. If the top $(n)$ bits match, then the translation result can be directly obtained (TLB).
2. If the top $(n-9)$ bits match, then we can go to the last level of page table directly.
3. If the top $(n-2*9)$ bits match, then we go to the second last level of page table.
4. If the top $(n-3*9)$ bits match, thn we go to the third last level.
5. Otherwise, use the top $(n-4*9)$ bits to index into the top most level of page table.

The cached value is always host physical address (hPA).

## Rocket-Chip: Two Level TLB with Extended Entries
Rocket chip implements two level of TLB, similar to two level of caches.
Rocket chip stores not only the gVA to hPA translation in TLB, but also the gVA to gPA translations.


## AMD: Page Walk Cache

# References:
1. N. C. Papadopoulos, V. Karakostas, K. Nikas, N. Koziris and D. N. Pnevmatikatos, "A Configurable TLB Hierarchy for the RISC-V Architecture," 2020 30th International Conference on Field-Programmable Logic and Applications (FPL), Gothenburg, Sweden, 2020, pp. 85-90, doi: 10.1109/FPL50879.2020.00024. URL: https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9221630&isnumber=9221504

