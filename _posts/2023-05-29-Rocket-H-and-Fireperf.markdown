---
layout: post
title: "Use Fireperf to Profile Rocket Chip with Hypervisor Extension"
categories: RISC-V Virtualization Fireperf
---

## Compile `kvm.ko` and Guest Kernel Image
Remember to comment out `csr_write(henvcfg)`.

## Compile `lkvm-static`
Follow the other post.

## Setup Rocket Chip with Hypervisor Extension and AutoCounters
### Build Single Core Rocket Chip with H-Extension and AutoCounters
Put basic AutoCounters at `RockectCore.scala`, add PWC AutoCounters at `PTW.scala`.

See [appendix](#appendix-one-click-patches) for git patches for these files.

## Inline Assembly for Fireperf Triggers
The expected compiled bytes are 00008013 and 00010013. Use `riscv64-unknown-linux-gnu-objdump -S` to check.
```c
#define FIRESIM_START_TRIGGER asm volatile ("addi x0, x1, 0":::"cc")
#define FIRESIM_END_TRIGGER asm volatile ("addi x0, x2, 0":::"cc")
```

## Rocket Chip Default Achitectural Parameters
|Component|Size|Explaination|Source|Experiments|
|---------|----|------------|------|-----------|
|L1I|16KB|4 way set associative * 64 sets * 64B cache line size|`src/main/scala/rocket/HellaCache.scala`||
|L1D|16KB|4 way set associative * 64 sets * 64B cache line size|`src/main/scala/rocket/HellaCache.scala`|4096 * 4B(uint_32) would fill up the cache|
|L2Shared|512KB|8 way set associative * 1024 sets * 64B cache line size|[SiFive Rocket Chip Inclusive Cache](https://github.com/chipsalliance/rocket-chip-inclusive-cache/tree/02e002b324c0e6316234045fa739fdb9d716170d)||
|L1ITLB|128 entries|32 way fully associative * 1 set * 4 sectors|`src/main/scala/rocket/HellaCache.scala`||
|L1DTLB|128 entries|32 way fully associative * 1 set * 4 sectors|`src/main/scala/rocket/HellaCache.scala`||
|L2TLB||Not instantiated|`src/main/scala/rocket/RocketCore.scala`||
|s1 s2 pte cache level 0 1|8 entries each|Two level PWC at each stage, two stage address translation|`src/main/scala/rocket/RocketCore.scala` and `src/main/scala/rocket/PTW.scala:makePTECache`||


## Four Files Under `firesim/deploy`
- `config_build_recipes.yaml`
  ```yaml
  firesim_rocket_h_singlecore_no_nic_l2_llc4mb_ddr3_with_dcache_tlb_counter_ptx:
    DESIGN: FireSim
    TARGET_CONFIG: FireSimRocketHypervisorConfig
    PLATFORM_CONFIG: WithAutoCounter_BaseF1Config
    deploy_triplet: null
    platform_config_args:
        fpga_frequency: 90
        build_strategy: TIMING
    post_build_hook: null
    metasim_customruntimeconfig: null
    bit_builder_recipe: bit-builder-recipes/f1.yaml
  ```
- `config_hwdb.yaml`
  ```yaml
  firesim_rocket_h_singlecore_no_nic_l2_llc4mb_ddr3_with_dcache_tlb_counter_ptx:
    agfi: agfi-09046647cef0cb3d7
    deploy_triplet_override: null
    custom_runtime_config: null
  ```
- `config_runtime.yaml`
  ```yaml
  target_config:
    default_hw_config: firesim_rocket_h_singlecore_no_nic_l2_llc4mb_ddr3_with_dcache_tlb_counter_ptx
  tracing:
    enable: yes
    output_format: 0
    selector: 3
    start: ffffffff00008013
    end: ffffffff00010013
  autocounter:
    read_rate: 100
  workload:
    workload_name: linuxkvm.json
  ```

# References
1. [Chipyard Rocket Chip Generator Microachitectual Parameters](https://chipyard.readthedocs.io/en/stable/Customization/Memory-Hierarchy.html#memory-hierarchy)

# Appendix 

## One Click Patches
To apply a patch, simply copy the content of the patch into a file, say `patch1.patch`, then `git apply patch1.patch`.

### Single Core Rocket Chip with Hypervisor Extension
Go to `firesim/target-design/chipyard/` and apply this patch.

```
diff --git a/generators/chipyard/src/main/scala/config/RocketConfigs.scala b/generators/chipyard/src/main/scala/config/RocketConfigs.scala
index b6677cb1..98b7b92a 100644
--- a/generators/chipyard/src/main/scala/config/RocketConfigs.scala
+++ b/generators/chipyard/src/main/scala/config/RocketConfigs.scala
@@ -7,6 +7,10 @@ import freechips.rocketchip.diplomacy.{AsynchronousCrossing}
 // Rocket Configs
 // --------------
 
+class RocketHypervisorConfig extends Config(
+  new freechips.rocketchip.subsystem.WithHypervisor ++
+  new RocketConfig)
+
 class RocketConfig extends Config(
   new freechips.rocketchip.subsystem.WithNBigCores(1) ++         // single rocket-core
   new chipyard.config.AbstractConfig)
diff --git a/generators/firechip/src/main/scala/TargetConfigs.scala b/generators/firechip/src/main/scala/TargetConfigs.scala
index 2ea848df..6475ba49 100644
--- a/generators/firechip/src/main/scala/TargetConfigs.scala
+++ b/generators/firechip/src/main/scala/TargetConfigs.scala
@@ -306,3 +306,9 @@ class FireSimLeanGemminiRocketMMIOOnlyConfig extends Config(
   new WithDefaultMemModel ++
   new WithFireSimConfigTweaks ++
   new chipyard.LeanGemminiRocketConfig)
+
+class FireSimRocketHypervisorConfig extends Config(
+  new WithDefaultFireSimBridges ++
+  new WithDefaultMemModel ++
+  new WithFireSimConfigTweaks ++
+  new chipyard.RocketHypervisorConfig)
```

### Insert AutoCounters into Rocket Chip
Go to `firesim/target-design/chipyard/generators/rocket-chip` and apply this patch

```
diff --git a/src/main/scala/rocket/DCache.scala b/src/main/scala/rocket/DCache.scala
index 8c83d2cc9..084ce5281 100644
--- a/src/main/scala/rocket/DCache.scala
+++ b/src/main/scala/rocket/DCache.scala
@@ -15,6 +15,8 @@ import chisel3.{DontCare, WireInit, dontTouch, withClock}
 import chisel3.internal.sourceinfo.SourceInfo
 import TLMessages._
 
+import midas.targetutils.PerfCounter
+
 // TODO: delete this trait once deduplication is smart enough to avoid globally inlining matching circuits
 trait InlineInstance { self: chisel3.experimental.BaseModule =>
   chisel3.experimental.annotate(
@@ -1052,6 +1054,14 @@ class DCacheModule(outer: DCache) extends HellaCacheModule(outer) {
   io.cpu.perf.acquire := edge.done(tl_out_a)
   io.cpu.perf.release := edge.done(tl_out_c)
   io.cpu.perf.grant := tl_out.d.valid && d_last
+
+  // val dcache_acquire = io.cpu.perf.acquire.asUInt
+  // midas.targetutils.PerfCounter.apply(dcache_acquire, "dcache_acq", "DCache Acquired")
+  // val dcache_release = io.cpu.perf.release.asUInt
+  // midas.targetutils.PerfCounter.apply(dcache_release, "dcache_release", "DCache release")
+  // val dcache_grant = io.cpu.perf.grant.asUInt
+  // midas.targetutils.PerfCounter.apply(dcache_grant, "dcache_grant", "DCache grant")
+
   io.cpu.perf.tlbMiss := io.ptw.req.fire()
   io.cpu.perf.storeBufferEmptyAfterLoad := !(
     (s1_valid && s1_write) ||
diff --git a/src/main/scala/rocket/NBDcache.scala b/src/main/scala/rocket/NBDcache.scala
index 796fb6872..65e74a733 100644
--- a/src/main/scala/rocket/NBDcache.scala
+++ b/src/main/scala/rocket/NBDcache.scala
@@ -10,6 +10,7 @@ import chisel3.experimental.dataview._
 import freechips.rocketchip.config.Parameters
 import freechips.rocketchip.tilelink._
 import freechips.rocketchip.util._
+import midas.targetutils.PerfCounter
 
 trait HasMissInfo extends Bundle with HasL1HellaCacheParameters {
   val tag_match = Bool()
@@ -1048,6 +1049,13 @@ class NonBlockingDCacheModule(outer: NonBlockingDCache) extends HellaCacheModule
   io.cpu.perf.release := edge.done(tl_out.c)
   io.cpu.perf.tlbMiss := io.ptw.req.fire
 
+  val nbdcache_acquire = io.cpu.perf.acquire.asUInt
+  PerfCounter.apply(nbdcache_acquire, "nbdcache_acquire", "NBDcache acquire")
+  val nbdcache_release = io.cpu.perf.release.asUInt
+  PerfCounter.apply(nbdcache_release, "nbdcache_release", "NBDcache release")
+  val nbdcache_tlbmiss = io.cpu.perf.tlbMiss.asUInt
+  PerfCounter.apply(nbdcache_tlbmiss, "nbdcache_tlbmiss", "NBDcache tlb miss")
+
   // no clock-gating support
   io.cpu.clock_enabled := true
 }
diff --git a/src/main/scala/rocket/PTW.scala b/src/main/scala/rocket/PTW.scala
index 717ef33a4..aa1bffdf7 100644
--- a/src/main/scala/rocket/PTW.scala
+++ b/src/main/scala/rocket/PTW.scala
@@ -13,6 +13,7 @@ import freechips.rocketchip.tile._
 import freechips.rocketchip.tilelink._
 import freechips.rocketchip.util._
 import freechips.rocketchip.util.property
+import midas.targetutils.PerfCounter
 
 import scala.collection.mutable.ListBuffer
 
@@ -392,6 +393,10 @@ class PTW(n: Int)(implicit edge: TLEdgeOut, p: Parameters) extends CoreModule()(
     val lcount = if (s2) aux_count else count
     for (i <- 0 until pgLevels-1) {
       ccover(hit && state === s_req && lcount === i.U, s"PTE_CACHE_HIT_L$i", s"PTE cache hit, level $i")
+
+      val pte_cache_hit = (hit && state === s_req && lcount === i.U).asUInt
+      val stage_name = if (s2) "stage 2" else "stage 1"
+      midas.targetutils.PerfCounter.apply(pte_cache_hit, s"pte_cache_hit_l${i}_${s2}", s"PTE cache hit, level ${i}, ${stage_name}")
     }
 
     (hit, Mux1H(hits, data))
@@ -404,6 +409,13 @@ class PTW(n: Int)(implicit edge: TLEdgeOut, p: Parameters) extends CoreModule()(
   val pte_hit = RegNext(false.B)
   io.dpath.perf.pte_miss := false.B
   io.dpath.perf.pte_hit := pte_hit && (state === s_req) && !io.dpath.perf.l2hit
+
+  val l1_pte_hit = io.dpath.perf.pte_hit.asUInt
+  midas.targetutils.PerfCounter.apply(l1_pte_hit, "l1_pte_hit", "L1 pte hit (TLB hit)")
+
+  val l1_pte_miss = io.dpath.perf.pte_miss.asUInt
+  midas.targetutils.PerfCounter.apply(l1_pte_miss, "l1_pte_miss", "L1 pte miss (TLB miss)")
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
+    midas.targetutils.PerfCounter.apply(l2_pte_miss, "l2_pte_miss", "L2 pte miss (TLB miss)")
+    val l2_pte_hit = io.dpath.perf.l2hit.asUInt
+    midas.targetutils.PerfCounter.apply(l2_pte_hit, "l2_pte_hit", "L2 pte hit (TLB hit)")
+
     when (s2_hit) {
       l2_plru.access(r_idx, OHToUInt(s2_hit_vec))
       assert((PopCount(s2_hit_vec) === 1.U) || s2_error, "L2 TLB multi-hit")
@@ -638,6 +656,8 @@ class PTW(n: Int)(implicit edge: TLEdgeOut, p: Parameters) extends CoreModule()(
     is (s_wait2) {
       next_state := s_wait3
       io.dpath.perf.pte_miss := count < (pgLevels-1).U
+      // val s_wait2_miss = io.dpath.perf.pte_miss.asUInt
+      // midas.targetutils.PerfCounter.apply(s_wait2_miss, "s_wait2_pte_miss", "s_wait2 pte miss")
       when (io.mem.s2_xcpt.ae.ld) {
         resp_ae_ptw := true.B
         next_state := s_ready
diff --git a/src/main/scala/rocket/RocketCore.scala b/src/main/scala/rocket/RocketCore.scala
index 1351a8086..8698acdfc 100644
--- a/src/main/scala/rocket/RocketCore.scala
+++ b/src/main/scala/rocket/RocketCore.scala
@@ -13,6 +13,8 @@ import freechips.rocketchip.util.property
 import freechips.rocketchip.scie._
 import scala.collection.mutable.ArrayBuffer
 
+import midas.targetutils.PerfCounter
+
 case class RocketCoreParams(
   bootFreqHz: BigInt = 0,
   useVM: Boolean = true,
@@ -987,6 +989,21 @@ class Rocket(tile: RocketTile)(implicit p: Parameters) extends CoreModule()(p)
   val icache_blocked = !(io.imem.resp.valid || RegNext(io.imem.resp.valid))
   csr.io.counters foreach { c => c.inc := RegNext(perfEvents.evaluate(c.eventSel)) }
 
+  // Autocounter perf events
+  val cpu_mem_load = (id_ctrl.mem && id_ctrl.mem_cmd === M_XRD && !id_ctrl.fp).asUInt
+  val cpu_mem_store = (id_ctrl.mem && id_ctrl.mem_cmd === M_XWR && !id_ctrl.fp).asUInt
+  val cpu_dcache_miss = (io.dmem.perf.acquire).asUInt
+  val cpu_dcache_release = (io.dmem.perf.release).asUInt
+  val cpu_dtlb_miss = (io.dmem.perf.tlbMiss).asUInt
+  val cpu_l2_tlb_miss = (io.ptw.perf.l2miss).asUInt
+
+  PerfCounter.apply(cpu_mem_load, "cpu_mem_load", "CPU Memory load")
+  PerfCounter.apply(cpu_mem_store, "cpu_mem_store", "CPU Memory store")
+  PerfCounter.apply(cpu_dcache_miss, "cpu_dcache_miss", "CPU dcache miss")
+  PerfCounter.apply(cpu_dcache_release, "cpu_dcache_release", "CPU dcache release")
+  PerfCounter.apply(cpu_dtlb_miss, "cpu_dtlb_miss", "CPU dtlb miss")
+  PerfCounter.apply(cpu_l2_tlb_miss, "cpu_l2_tlb_miss", "CPU shared l2 tlb miss")
+
   val coreMonitorBundle = Wire(new CoreMonitorBundle(xLen, fLen))
 
   coreMonitorBundle.clock := clock
diff --git a/src/main/scala/rocket/TLB.scala b/src/main/scala/rocket/TLB.scala
index 798fa98b0..9909207ab 100644
--- a/src/main/scala/rocket/TLB.scala
+++ b/src/main/scala/rocket/TLB.scala
@@ -15,6 +15,7 @@ import freechips.rocketchip.util._
 import freechips.rocketchip.util.property
 import freechips.rocketchip.devices.debug.DebugModuleKey
 import chisel3.internal.sourceinfo.SourceInfo
+import midas.targetutils.PerfCounter
 
 case object PgLevels extends Field[Int](2)
 case object ASIdBits extends Field[Int](0)
@@ -723,6 +724,9 @@ class TLB(instruction: Boolean, lgMaxSize: Int, cfg: TLBConfig)(implicit edge: T
     }
 
     ccover(io.ptw.req.fire, "MISS", "TLB miss")
+    // val tlb_miss_event = io.ptw.req.fire.asUInt
+    // midas.targetutils.PerfCounter.apply(tlb_miss_event, "tlb_miss", "tlb miss")
+
     ccover(io.ptw.req.valid && !io.ptw.req.ready, "PTW_STALL", "TLB miss, but PTW busy")
     ccover(state === s_wait_invalidate, "SFENCE_DURING_REFILL", "flush TLB during TLB refill")
     ccover(sfence && !io.sfence.bits.rs1 && !io.sfence.bits.rs2, "SFENCE_ALL", "flush TLB")

```