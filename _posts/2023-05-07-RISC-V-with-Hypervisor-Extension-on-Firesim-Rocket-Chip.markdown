---
layout: post
title: "Running RISC-V with Hypervisor Extension on Firesim Using Rocket Chip"
categories: RISC-V Virtualization Firesim
---

# Environment
Firesim 1.16.0

# Recap
In the previous post, we boot a linux hypervisor + guest on QEMU. In this post, we will do the same thing with firesim and rocket chip, which allows for FPGA-accelearted cycle accurate simulation.

# Firesim, Chipyard, Firemarshal and Rocket Chip.
- Firesim: the framework for agile achitecture development, testing and evaluation.
- Chipyard: contains generators for different chips, such as rocket chip and boom.
- Firemarshal: helps to setup linux images and rootfs for simulation.
- Rocket Chip: A generator that can create hardware descriptions of different flavours of an in-order CPU. More importantly, it implements the hypervisor extension. 

# Running Linux Hypervisor and Linux Guest on Firesim

## Setup Firesim on AWS EC2
Follow the [firesim initial setup tutorial](https://docs.fires.im/en/stable/Initial-Setup/index.html) to setup AWS-based infrastructure. `mosh` is preferred to `ssh` when connecting to the EC2 manager instance. `mosh` tolerates network changes, so that we can take a walk in the park when long running tasks such as `firesim buildbitstream` are executing, bringing our laptop. When using EC2 management console, always check if you are in the correct region.

## Build Rocket Chip with Hypervisor Extention FPGA image
Once firesim manager instance is setup, we can first build the FPGA image that allows us to simulate rocket chip with hypervisor extension. The following is mainly based on the [firesim ASPLOS Tutorial 2023](https://fires.im/asplos23-slides-pdf/07_building_hw_firesim.pdf), the [kvm_howto wiki](https://github.com/kvm-riscv/howto/wiki/KVM-RISC-V-64bit-on-FireSim) and the [firesim documentation](https://docs.fires.im/en/stable/Building-a-FireSim-AFI.html#amazon-s3-setup).

To build the FPGA image for Rocket Chip, we first need to specify a RocketConfig that has Hypervisor Extension enabled. Then we specify a FPGA config that wraps around the RocketConfig. After that, we specify how the build would look like and ask firesim to build for us this image. Finally, we add the built image to our hardware database so that we can use it. The actual image will be stored in an auto created AWS S3 bucket.

First, specify a rocket chip config that has h-extension
Append to `firesim/target-design/chipyard/generators/chipyard/src/main/scala/config/RocketConfigs.scala`:
```scala
class QuadRocketHypConfig extends Config(
new freechips.rocketchip.subsystem.WithHypervisor ++
new QuadRocketConfig)
```

> Note: To check what subsystems are available, see `firesim/target-design/chipyard/generators/rocket-chip/src/main/scala/subsystem/Configs.scala`

Then, define FPGA config at `firesim/target-design/chipyard/generators/firechip/src/main/scala/TargetConfigs.scala`:
```scala
class FireSimQuadRocketHypConfig extends Config(
new WithDefaultFireSimBridges ++
new WithDefaultMemModel ++
new WithFireSimConfigTweaks ++
new chipyard.QuadRocketHypConfig)
```

Then add build receipe to `firesim/deploy/config_build_recipes.yaml`:
```yaml
firesim_rocket_h_quadcore_no_nic_l2_llc4mb_ddr3_ours:
    DESIGN: FireSim
    TARGET_CONFIG: FireSimQuadRocketHypConfig
    PLATFORM_CONFIG: BaseF1Config
    deploy_triplet: null
    platform_config_args:
        fpga_frequency: 90
        build_strategy: TIMING
    post_build_hook: null
    metasim_customruntimeconfig: null
    bit_builder_recipe: bit-builder-recipes/f1.yaml
```

> Note: The `TARGET_CONFIG` is parsed following the syntatic rules described [here](https://docs.fires.im/en/1.16.0/Advanced-Usage/Generating-Different-Targets.html#specifying-a-target-instance).

Then turn on build in `firesim/deploy/config_build.yaml`.

Start the image building process:
```
$ firesim buildbitstream
```

When the build process is complete, add the `afgi` to `firesim/deploy/config_hwdb.yaml`:
```yaml
firesim_rocket_h_quadcore_no_nic_l2_llc4mb_ddr3_ours:
    agfi: agfi-?
    deploy_triplet_override: null
    custom_runtime_config: null
```

Finally, specify that we want to use our built image in `firesim/deploy/config_runtime.yaml`:
```yaml
...
target_config:
    ...
    default_hw_config: firesim_rocket_h_quadcore_no_nic_l2_llc4mb_ddr3_ours
    ...
...
```

## Use Firemarshal to Build Linux Workload and Boot Linux

Firemarshal is a buildroot-like system specifically designed for generating workloads for firesim FPGA accelerated simulation. Using firemarshal, we can easily create and deploy workloads on firesim simulation.

We first start with a simple linux boot to demonstrate the firemarshal workflow. This example is from the [firesim documentation](https://docs.fires.im/en/stable/Running-Simulations-Tutorial/Running-a-Single-Node-Simulation.html).

First, initialize firemarshal:
```
$ cd firesim/sw/firesim-software
$ ./init-submodules.sh
```
Then build the workload, which is a pre-defined workload:

```
$ marshal build br-base.json
```

Since this workload is pre-defined, we can refer to it directly. Change the `firesim/deploy/config_runtime.yaml` accordingly:

```yaml
...
workload:
        workload_name: linux-uniform.json
        ...
...
```

To prepare F1 instances to run simulation, we first start them:
```
$ firesim launchrunfarm
```

Then, before each run of simulation, we run:
```
$ firesim infrasetup
```
Note: This may take sometime.

When the machine is ready, we can run the workload:
```
$ firesim runworkload
```

Create another `ssh` connection into the EC2 manager instance, then `ssh` into the F1 instance running the simulation, then attach to the screen of that simulation using `screen -r fsim0`.

We should now be able to login with username `root`. Use `poweroff` to finish simulation.

When simulation is finished, return to the `mosh` connection to the EC2 manager instance. We should see that simulation is complete and logs and outputs are properly saved.

To start another run of simulation, we need to `firesim infrasetup && firesim runworkload` again, even if the workload and hardware config have not changed, as adviced by the firesim documentation.

When everything is finished, we should stop the costly F1 instances:
```
$ firesim terminaterunfarm
``` 
and input `yes`.

> Check from EC2 management console that the instance has terminated, and then stop (Note: not terminate) the manager instance if you decide to finish today's work.


## Boot Linux Hypervisor and Linux Guest
Now we can run our custom workload, which will be based on `br-base.json`, with additional files `Image`, `lkvm-static` and `kvm.ko`.

First, set firemarshal workload search path by editing ``:
```
```


# References

1. [Firesim ASPLOS Tutorial 2023](https://fires.im/asplos-2023-tutorial/)
2. [Firemarshal](https://firemarshal.readthedocs.io/en/latest/)
3. [Screen](https://www.gnu.org/software/screen/manual/screen.html)

# Appendix

## Firesim Conda environment
In the firesim conda environment, we are using a specific set of compiler toolchains. Specifically, the GCC in the environment uses its own sysroot, which can be located with `gcc --print-sysroot`. This means that `apt install` will not install packages to this sysroot.

## Firemarshal and QEMU
We can test the firemarshal created workload with QEMU on the manager instance and verify its functional correctness. However, The pre-built qemu5.0 that ships with firesim conda env does not support hypervisor extension. I tried to compile a QEMU on the manager instance, eventually succeeded but still unable to run the workload. 

The qemu command to start the workload can be found with `marshal -v launch workload_name`.

## Screen
- If you press `ctrl-a` then `ctrl-x` on screen, then you will lock the screen session. You need to unlock it with the password of the current user `centos`, which you don't know. In this situation we usually sigh and use `firesim kill` to stop the simulation. (Note: `ctrl-a` then `ctrl-d` to detach from screen, leaving the program running.) (`ctrl-a` then `ctrl-x` on QEMU is to terminate QEMU.) 