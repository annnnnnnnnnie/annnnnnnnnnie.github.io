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
[[3]](#3)

## Page Walk Caches (PWC) and Nested TLB (NTLB)
![PWC and TLB](/images/Virtual-Memory-and-Caches/2DPTW_Caches_NTLB.png)
[[3]](#3)

## The Sv39 Page-Based 39-bit Virtual Memory System
Rocket Chip implements the Sv39 VM scheme [[2]](#2).
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
Rocket chip implements two level of TLB, similar to two level of caches [[1]](#1).
Source code at `rocket-chip/src/main/scala/rocket/PTW.scala`.
Rocket chip caches translations. (pte_cache and s2_pte_cache).
Rocket chip stores not only the gVA to hPA translation in TLB, but also the gVA to gPA translations [[4]](#4).

![RocketChipAddressTranslation](/images/Virtual-Memory-and-Caches/RocketChipAddressTranslation.png)

# References:
1. N. C. Papadopoulos, V. Karakostas, K. Nikas, N. Koziris and D. N. Pnevmatikatos, "A Configurable TLB Hierarchy for the RISC-V Architecture," 2020 30th International Conference on Field-Programmable Logic and Applications (FPL), Gothenburg, Sweden, 2020, pp. 85-90, doi: 10.1109/FPL50879.2020.00024. URL: https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9221630&isnumber=9221504

2. "The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Document Version 20211203", Editors Andrew Waterman, Krste Asanovi´c, and John Hauser, RISC-V International, December 2021. URL: https://riscv.org/technical/specifications/

3. Ravi Bhargava, Benjamin Serebrin, Francesco Spadini, and Srilatha Manne. 2008. Accelerating two-dimensional page walks for virtualized systems. In Proceedings of the 13th international conference on Architectural support for programming languages and operating systems (ASPLOS XIII). Association for Computing Machinery, New York, NY, USA, 26–35. https://doi.org/10.1145/1346281.1346286

4. Sá, B., Martins, J., and Pinto, S., “A First Look at RISC-V Virtualization from an Embedded Systems Perspective”, <i>arXiv e-prints</i>, 2021. doi:10.48550/arXiv.2103.14951.

# Appendix

## Adding AutoCounters to Rocket Chip
Import `midas.targetutils.PerfCounter`, then use `PerfCounter.apply(UInt, LabelName, HumanReadableDescription)`. `LabelName` must be globally unique and contain no space.

```
diff --git a/src/main/scala/rocket/DCache.scala b/src/main/scala/rocket/DCache.scala
index 0467d149a..acd972f79 100644
--- a/src/main/scala/rocket/DCache.scala
+++ b/src/main/scala/rocket/DCache.scala
@@ -1050,8 +1050,17 @@ class DCacheModule(outer: DCache) extends HellaCacheModule(outer) {
 
   // performance events
   io.cpu.perf.acquire := edge.done(tl_out_a)
+  val dcache_acquire = io.cpu.perf.acquire.asUInt
+  midas.targetutils.PerfCounter.apply(dcache_acquire, "dcache_acq", "DCache Acquired")
+
   io.cpu.perf.release := edge.done(tl_out_c)
+  val dcache_release = io.cpu.perf.release.asUInt
+  midas.targetutils.PerfCounter.apply(dcache_release, "dcache_release", "DCache release")
+
   io.cpu.perf.grant := tl_out.d.valid && d_last
+  val dcache_grant = io.cpu.perf.grant.asUInt
+  midas.targetutils.PerfCounter.apply(dcache_grant, "dcache_grant", "DCache grant")
+
   io.cpu.perf.tlbMiss := io.ptw.req.fire()
   io.cpu.perf.storeBufferEmptyAfterLoad := !(
     (s1_valid && s1_write) ||
diff --git a/src/main/scala/rocket/PTW.scala b/src/main/scala/rocket/PTW.scala
index 7c36bdbf3..f814f629a 100644
--- a/src/main/scala/rocket/PTW.scala
+++ b/src/main/scala/rocket/PTW.scala
@@ -13,6 +13,7 @@ import freechips.rocketchip.tile._
 import freechips.rocketchip.tilelink._
 import freechips.rocketchip.util._
 import freechips.rocketchip.util.property
+import midas.targetutils.PerfCounter
 
 import scala.collection.mutable.ListBuffer
 
@@ -392,6 +393,9 @@ class PTW(n: Int)(implicit edge: TLEdgeOut, p: Parameters) extends CoreModule()(
     val lcount = if (s2) aux_count else count
     for (i <- 0 until pgLevels-1) {
       ccover(hit && state === s_req && lcount === i.U, s"PTE_CACHE_HIT_L$i", s"PTE cache hit, level $i")
+
+      val pte_cache_hit = (hit && state === s_req && lcount === i.U).asUInt
+      midas.targetutils.PerfCounter.apply(pte_cache_hit, s"pte_cache_hit_l${i}_${s2}", s"PTE cache hit, level $i")
     }
 
     (hit, Mux1H(hits, data))
@@ -404,6 +408,14 @@ class PTW(n: Int)(implicit edge: TLEdgeOut, p: Parameters) extends CoreModule()(
   val pte_hit = RegNext(false.B)
   io.dpath.perf.pte_miss := false.B
   io.dpath.perf.pte_hit := pte_hit && (state === s_req) && !io.dpath.perf.l2hit
+
+  val l1_pte_hit = io.dpath.perf.pte_hit.asUInt
+  midas.targetutils.PerfCounter.apply(l1_pte_hit, "l1_pte_hit", "L1 pte hit")
+
+  val l1_pte_miss = io.dpath.perf.pte_miss.asUInt
+  midas.targetutils.PerfCounter.apply(l1_pte_miss, "l1_pte_miss", "L1 pte miss")
+
+
   assert(!(io.dpath.perf.l2hit && (io.dpath.perf.pte_miss || io.dpath.perf.pte_hit)),
     "PTE Cache Hit/Miss Performance Monitor Events are lower priority than L2TLB Hit event")
   // l2_refill happens when find the leaf pte
@@ -493,6 +505,12 @@ class PTW(n: Int)(implicit edge: TLEdgeOut, p: Parameters) extends CoreModule()(
     val s2_hit = s2_valid && s2_hit_vec.orR
     io.dpath.perf.l2miss := s2_valid && !(s2_hit_vec.orR)
     io.dpath.perf.l2hit := s2_hit
+
+    val l2_pte_miss = io.dpath.perf.l2miss.asUInt
+    midas.targetutils.PerfCounter.apply(l2_pte_miss, "l2_pte_miss", "L2 pte miss")
+    val l2_pte_hit = io.dpath.perf.l2hit.asUInt
+    midas.targetutils.PerfCounter.apply(l2_pte_hit, "l2_pte_hit", "L2 pte hit")
+
     when (s2_hit) {
       l2_plru.access(r_idx, OHToUInt(s2_hit_vec))
       assert((PopCount(s2_hit_vec) === 1.U) || s2_error, "L2 TLB multi-hit")
@@ -638,6 +656,8 @@ class PTW(n: Int)(implicit edge: TLEdgeOut, p: Parameters) extends CoreModule()(
     is (s_wait2) {
       next_state := s_wait3
       io.dpath.perf.pte_miss := count < (pgLevels-1).U
+      val s_wait2_miss = io.dpath.perf.pte_miss.asUInt
+      midas.targetutils.PerfCounter.apply(s_wait2_miss, "s_wait2_pte_miss", "s_wait2 pte miss")
       when (io.mem.s2_xcpt.ae.ld) {
         resp_ae_ptw := true.B
         next_state := s_ready
diff --git a/src/main/scala/rocket/TLB.scala b/src/main/scala/rocket/TLB.scala
index 977f73819..879d02f1e 100644
--- a/src/main/scala/rocket/TLB.scala
+++ b/src/main/scala/rocket/TLB.scala
@@ -15,6 +15,7 @@ import freechips.rocketchip.util._
 import freechips.rocketchip.util.property
 import freechips.rocketchip.devices.debug.DebugModuleKey
 import chisel3.internal.sourceinfo.SourceInfo
+import midas.targetutils.PerfCounter
 
 case object PgLevels extends Field[Int](2)
 case object ASIdBits extends Field[Int](0)
@@ -721,6 +722,9 @@ class TLB(instruction: Boolean, lgMaxSize: Int, cfg: TLBConfig)(implicit edge: T
     }
 
     ccover(io.ptw.req.fire, "MISS", "TLB miss")
+    val tlb_miss_event = io.ptw.req.fire.asUInt
+    midas.targetutils.PerfCounter.apply(tlb_miss_event, "tlb_miss", "tlb miss")
+
     ccover(io.ptw.req.valid && !io.ptw.req.ready, "PTW_STALL", "TLB miss, but PTW busy")
     ccover(state === s_wait_invalidate, "SFENCE_DURING_REFILL", "flush TLB during TLB refill")
     ccover(sfence && !io.sfence.bits.rs1 && !io.sfence.bits.rs2, "SFENCE_ALL", "flush TLB")
```
