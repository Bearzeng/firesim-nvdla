# FireSim-NVDLA: NVDLA Integrated with Rocket Chip SoC on FireSim

FireSim-NVDLA is a fork of the [FireSim](https://github.com/firesim/firesim) FPGA-accelerated full-system simulator which we have integrated [NVIDIA Deep Learning Accelerator (NVDLA)](http://nvdla.org) into. FireSim-NVDLA runs on the Amazon FPGA cloud (EC2 F1 instance). The figure below shows the overview of FireSim-NVDLA:

<p align="center">
<img src="http://ittc.ku.edu/~farshchi/firesim-nvdla/overview.png" width="450">
</p>

## Contents

1. [Using FireSim](#using-firesim)
2. [Running YOLOv3 on NVDLA](#running-yolov3-on-nvdla)
3. [Building Your Own Hardware](#building-your-own-hardware)
4. [Questions and Reporting Bugs](#questions-and-reporting-bugs)
5. [EMC<sup>2</sup> Workshop Paper](#emc2-workshop-paper)

## Using FireSim

To work with FireSim-NVDLA, first, you need to learn how to use FireSim. Once you learned that, simulating NVDLA is going to be very easy. We recommend following the steps in the [FireSim documentation (v1.5.0)](http://docs.fires.im/en/1.5.0) to set up the simulator and run a single-node simulation. The only difference is you use the URL of this repository when cloning FireSim in ["Setting up the FireSim Repo"](http://docs.fires.im/en/1.5.0/Initial-Setup/Setting-up-your-Manager-Instance.html#setting-up-the-firesim-repo):

```
git clone https://github.com/CSL-KU/firesim-nvdla
cd firesim-nvdla
./build-setup.sh fast
```

Once you successfully run a single-node simulation, come back to this guide and follow the rest of instructions.

## Running YOLOv3 on NVDLA
In this section, we guide you through configuring FireSim to run [YOLOv3](https://pjreddie.com/darknet/yolo) object detection algorithm on NVDLA. YOLOv3 runs on a modified version of the [Darknet](https://github.com/CSL-KU/darknet-nvdla) neural network framework that supports NVDLA acceleration. First, download Darknet and rebuild the target software:

```
cd firesim-nvdla/sw/firesim-software
./get-darknet.sh
./marshal -v build workloads/darknet-nvdla.json
./marshal install workloads/darknet-nvdla.json
```

Then, configure FireSim to simulate the target which has the NVDLA model. In order to do that, in `firesim-nvdla/deploy/config_runtime.ini`, change the parameter `defaulthwconfig` to `firesim-quadcore-no-nic-nvdla-ddr3-llc4mb`. Additionally change `workloadname` to `darknet-nvdla.json`. Your final `config_runtime.ini` should look like this:

```
# RUNTIME configuration for the FireSim Simulation Manager
# See docs/Advanced-Usage/Manager/Manager-Configuration-Files.rst for documentation of all of these params.

[runfarm]
runfarmtag=mainrunfarm

f1_16xlarges=0
m4_16xlarges=0
f1_4xlarges=0
f1_2xlarges=1

runinstancemarket=ondemand
spotinterruptionbehavior=terminate
spotmaxprice=ondemand

[targetconfig]
topology=no_net_config
no_net_num_nodes=1
linklatency=6405
switchinglatency=10
netbandwidth=200
profileinterval=-1

# This references a section from config_hwconfigs.ini
# In homogeneous configurations, use this to set the hardware config deployed
# for all simulators
defaulthwconfig=firesim-quadcore-no-nic-nvdla-ddr3-llc4mb

[tracing]
enable=no
startcycle=0
endcycle=-1

[workload]
workloadname=darknet-nvdla.json
terminateoncompletion=no
```

Follow the instructions on the FireSim documentation to launch the simulation then, log into the simulated machine and run:

```
cd darknet-nvdla
./solo.sh
```

The command above launches Darknet and runs YOLOv3 on the image `darknet-nvdla/data/person.jpg`. Once detection is done, the time that it takes to run the algorithm and the probabilities of objects detected in the image appear on the screen:

```
...
data/person.jpg: Predicted in 0.129548 seconds.
horse: 100%
dog: 99%
person: 100%
```

Darknet saves the image with bounding boxes around the detected objects in `darknet-nvdla/predictions.png`:

<p align="center">
<img src="http://www.ittc.ku.edu/~farshchi/firesim-nvdla/person-detected.png" width="550">
</p>

## Building Your Own Hardware
The pre-built target we provided above is for a quad-core processor with no network interface, a last-level cache with the maximum size of 4 MiB, and a DDR3 memory model with FR-FCFS controller. It is simple and easy to add NVDLA to any other configuration and build your own FPGA image. First, learn how to build a FireSim FPGA image by reading ["Building Your Own Hardware Designs (FireSim FPGA Images)"](http://docs.fires.im/en/1.5.0/Building-a-FireSim-AFI.html) and learn about the meaning and use of parameters in [`config_build_recipes.ini`](http://docs.fires.im/en/1.5.0/Advanced-Usage/Manager/Manager-Configuration-Files.html#config-build-recipes-ini).

Once you know how to build a FireSim FPGA image, building your own custom configuration with NVDLA is easy. Simply, add a new build definition in `config_build_recipes.ini` and add `_WithNVDLALarge` to the end of `TARGET_CONFIG` parameter. For example, use the build definition below to build an image for a single-core processor with a network interface and a latency-bandwidth pipe memory model:

```
[name-of-your-configuration]
DESIGN=FireSim
TARGET_CONFIG=FireSimRocketChipSingleCoreConfig_WithNVDLALarge
PLATFORM_CONFIG=FireSimConfig
instancetype=c5.9xlarge
deploytriplet=None
```

Replace `name-of-your-configuration` with the desired name for your new configuration. Follow the instructions on the FireSim documentation to build the AGFI and add it to `config_hwdb.ini`. NVDLA is a large design therefore, it takes about 8 hours to finish the build on a c5.9xlarge instance. To simulate the target you have built, replace `defaulthwconfig` in `config_runtime.ini` with `name-of-your-configuration`.

You can do all sort of cool things by experimenting with different configurations. For example, you can measure the performance of NVDLA with respect to the memory latency when you choose the latency-bandwidth pipe memory model. The latency of this memory model can be configured at the runtime without having to rebuild the FPGA image. In addition, the Rocket Chip can be further customized by modifying the Chisel code. For example, you can change the memory bus width and see how the NVDLA performance changes.

## Questions and Reporting Bugs
If you have a question about using FireSim-NVDLA or you want to report a bug, please file an issue on this repository.

## EMC<sup>2</sup> Workshop Paper
You can read our EMC<sup>2</sup> workshop paper to learn more about the integration of NVDLA into FireSim and find out how we used this platform to evaluate the perforamnce of NVDLA:

Farzad Farshchi, Qijing Huang, and Heechul Yun, **"Integrating NVIDIA Deep Learning Accelerator (NVDLA) with RISC-V SoC on FireSim"**, 2nd Workshop on Energy Efficient Machine Learning and Cognitive Computing for Embedded Applications (EMC<sup>2</sup> 2019), Washington, DC, February 2019. [Paper PDF](http://www.ittc.ku.edu/~farshchi/papers/nvdla-firesim-emc2-paper.pdf) | [Slides](http://www.ittc.ku.edu/~farshchi/papers/nvdla-firesim-emc2-slides.pdf)
