---
layout: post
title: "Use FirePerf to Profile Rocket Chip with Hypervisor Extension"
categories: RISC-V Virtualization FirePerf
---

## Rocket Chip Default Architectural Parameters
|Component|Size|Explaination|Source|Coverage|
|---------|----|------------|------|--------|
|L1I|16KB|4 way set associative * 64 sets * 64B cache line size|`src/main/scala/rocket/HellaCache.scala`| 16KB |
|L1D|16KB|4 way set associative * 64 sets * 64B cache line size|`src/main/scala/rocket/HellaCache.scala`| 16KB |
|L2Shared|512KB|8 way set associative * 1024 sets * 64B cache line size|[SiFive Rocket Chip Inclusive Cache](https://github.com/chipsalliance/rocket-chip-inclusive-cache/tree/02e002b324c0e6316234045fa739fdb9d716170d)| 512KB |
|L1ITLB|32 entries|32 way fully associative * 1 set |`src/main/scala/rocket/HellaCache.scala`| 32 * 4KB = 128KB |
|L1DTLB|32 entries|32 way fully associative * 1 set |`src/main/scala/rocket/HellaCache.scala`| 32 * 4KB = 128KB |
|L2TLB||Not instantiated|`src/main/scala/rocket/RocketCore.scala`| - |
|s1 s2 pte cache level 0 1|8 entries each|Two level PWC at each stage, two stage address translation|`src/main/scala/rocket/RocketCore.scala` and `src/main/scala/rocket/PTW.scala:makePTECache`| level 1: 12 + 9 bits -> 2^21 = 2MB * 8 entries = 16MB </br> level 0: 12 + 9 + 9 bits -> 2^30 = 1GB * 8 = 8GB |

## Experiment Design
### ccbench/caches[[4]](#references)
$(num_elem) uint32_t in a contiguous array.
When runType = 0, this will access a fixed sized array randomly, at cache boundary. Otherwise, the array will be accessed in a strided way, with stride = runType.

| num_element | array size | remark                                                                               | how to verify                                                         |
|-------------|------------|--------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| 4K          | 16KB       |     L1D cache size = 16KB                                                                | expecting dcache misses to be low for <4K and high for >4K (3e5)      |
| 32K         | 128KB      |     L1D TLB coverage = 32 entries * 4KB each = 128KB                                    | expecting DTLB misses to be low for <32K and high for >32K            |
| 128K        | 512KB      |     L2 Shared cache size = 512KB                                                        | expecting execution time to increase for 128K                         |
| 4M          | 16MB       |     gPWC_1 coverage = 8 entries * 2^21 = 8 * 2MB = 16MB<br/> 2 more stage 2 translations | expecting gPWC_1 hit to be low for >4M (However this is not observed) |
| -           | -          |     gPWC_0 coverage = 8 entries * 2^30 = 8 * 1GB = 8GB<br/> 3 more stage 2 translations | this is greater than the maximum size of problems we investigate      |

## Automated Result Collection
First, change in `kvmtool/include/kvm/kvm-config.h` the `DEFAULT_SANDBOX_FILENAME` to `sandbox.sh`. Recompile `lkvm-static`.
```
diff --git a/include/kvm/kvm-config.h b/include/kvm/kvm-config.h
index 368e6c7..05199dd 100644
--- a/include/kvm/kvm-config.h
+++ b/include/kvm/kvm-config.h
@@ -15,7 +15,7 @@
 #define DEFAULT_GUEST_MAC      "02:15:15:15:15:15"
 #define DEFAULT_HOST_MAC       "02:01:01:01:01:01"
 #define DEFAULT_SCRIPT         "none"
-#define DEFAULT_SANDBOX_FILENAME "guest/sandbox.sh"
+#define DEFAULT_SANDBOX_FILENAME "sandbox.sh"
 
 #define MIN_RAM_SIZE           SZ_64M
```

Following the firemarshal documentation[[2]](#references), we make use of the `run` option to automatically run the workload for us.
```json
{
  "name" : "linuxkvm",
  "base" : "br-base.json",
  "files": [ ["./Image", "/root/"],
             ["./kvm.ko", "/root/"],
             ["./linuxkvmriscv64/kvmtool/lkvm-static", "/root/"]
           ],
  "overlay": "./linuxkvmriscv64/overlay",
  "run": "./linuxkvmriscv64/overlay/root/run_guest_sandbox.sh"
}

```

The `run_guest_sandbox.sh` is as follows:
```
echo "Inserting kvm.ko"
insmod /root/kvm.ko

cat /root/$1

echo "Starting sandbox"

/root/lkvm-static sandbox -m 1G -c1 --console serial -p "console=ttyS0 earlycon" -k /root/Image -- sh /host/root/$1
poweroff
# Example input for $1:
# mem_benches/ccbench/run_caches_guest.sh

# The normal lkvm-static run instruction is 
# /root/lkvm-static run -m 1G -c1 --console serial -p "console=ttyS0 earlycon" -k /root/Image
```

A sample `run_caches.sh` may look something like this:
```
BASEDIR=$(dirname "$0")

numIterations=300000
runType=0

for appSizeArg in 67108864 268435456 1073741824 2147483648
do 
  $BASEDIR/caches $appSizeArg $numIterations $runType
  sleep 0.1
done

runType=16
for appSizeArg in 67108864 268435456 1073741824 2147483648
do 
  $BASEDIR/caches $appSizeArg $numIterations $runType
  sleep 0.1
done

```

Now `marshal build` and `marshal install` the workload to `firesim/deploy/workloads`.

Edit `firesim/deploy/workloads/linuxkvm.json` to include `TRACEFILE*` and `AUTOCOUNTER*` as output.
```json
{
  "common_simulation_outputs": [
    "uart_log",
    "AUTOCOUNTER*"
  ]
}
```

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
    enable: no
    output_format: 0
    selector: 3
    start: ffffffff00008013
    end: ffffffff00010013
  autocounter:
    read_rate: 100
  workload:
    workload_name: linuxkvm.json
    terminate_on_completion: yes # Choose yes to automatically shutdown the f1 instances.
  ```
- `config_build.yaml`
  ```yaml
  agfi_to_build:
    - firesim_rocket_h_singlecore_no_nic_l2_llc4mb_ddr3_with_dcache_tlb_counter_ptx
  ```

# References
1. [Chipyard Rocket Chip generator micro-architectural parameters](https://chipyard.readthedocs.io/en/stable/Customization/Memory-Hierarchy.html#memory-hierarchy)
2. [Firemarshal workload configuration options](https://firemarshal.readthedocs.io/en/latest/workloadConfig.html#configuration-options)
3. [kvmtool man page](https://github.com/kvmtool/kvmtool/blob/master/Documentation/kvmtool.1)
4. [ccbench](https://github.com/ucb-bar/ccbench)

# Appendix 

## One Click Patches
To apply a patch, simply copy the content of the patch into a file, say `patch1.patch`, then `git apply patch1.patch`.

### Rocket Core AutoCounters

```diff
diff --git a/src/main/scala/rocket/PTW.scala b/src/main/scala/rocket/PTW.scala
index 717ef33a4..103875cfb 100644
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
+      val stage_name = if (s2) "guest" else "host"
+      PerfCounter.apply(pte_cache_hit, s"pte_cache_hit_l${i}_${s2}", s"PTE cache hit, level ${i}, on ${stage_name} translation")
     }
 
     (hit, Mux1H(hits, data))
diff --git a/src/main/scala/rocket/RocketCore.scala b/src/main/scala/rocket/RocketCore.scala
index 1351a8086..75acf9c50 100644
--- a/src/main/scala/rocket/RocketCore.scala
+++ b/src/main/scala/rocket/RocketCore.scala
@@ -12,6 +12,7 @@ import freechips.rocketchip.util._
 import freechips.rocketchip.util.property
 import freechips.rocketchip.scie._
 import scala.collection.mutable.ArrayBuffer
+import midas.targetutils.PerfCounter
 
 case class RocketCoreParams(
   bootFreqHz: BigInt = 0,
@@ -987,6 +988,25 @@ class Rocket(tile: RocketTile)(implicit p: Parameters) extends CoreModule()(p)
   val icache_blocked = !(io.imem.resp.valid || RegNext(io.imem.resp.valid))
   csr.io.counters foreach { c => c.inc := RegNext(perfEvents.evaluate(c.eventSel)) }
 
+  // Autocounter perf events
+  val cpu_mem_load = (id_ctrl.mem && id_ctrl.mem_cmd === M_XRD && !id_ctrl.fp).asUInt
+  val cpu_mem_store = (id_ctrl.mem && id_ctrl.mem_cmd === M_XWR && !id_ctrl.fp).asUInt
+  val cpu_dcache_miss = (io.dmem.perf.acquire).asUInt
+  val cpu_dcache_release = (io.dmem.perf.release).asUInt
+  val cpu_dtlb_miss = (io.dmem.perf.tlbMiss).asUInt
+  val cpu_l2_tlb_miss = (io.ptw.perf.l2miss).asUInt
+  val cpu_ptw_pte_hit = (io.ptw.perf.pte_hit).asUInt
+  val cpu_ptw_pte_miss = (io.ptw.perf.pte_miss).asUInt
+
+  PerfCounter.apply(cpu_mem_load, "cpu_mem_load", "CPU main memory load")
+  PerfCounter.apply(cpu_mem_store, "cpu_mem_store", "CPU main memory store")
+  PerfCounter.apply(cpu_dcache_miss, "cpu_dcache_miss", "CPU dcache miss")
+  PerfCounter.apply(cpu_dcache_release, "cpu_dcache_release", "CPU dcache release")
+  PerfCounter.apply(cpu_dtlb_miss, "cpu_dtlb_miss", "CPU dtlb miss")
+  PerfCounter.apply(cpu_l2_tlb_miss, "cpu_l2_tlb_miss", "CPU shared l2 tlb miss")
+  PerfCounter.apply(cpu_ptw_pte_hit, "cpu_ptw_pte_hit", "CPU PTW any pte hit")
+  PerfCounter.apply(cpu_ptw_pte_miss, "cpu_ptw_pte_miss", "CPU PTW all pte miss")
+
   val coreMonitorBundle = Wire(new CoreMonitorBundle(xLen, fLen))
 
   coreMonitorBundle.clock := clock
```
