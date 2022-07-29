---
title: Getting ResNet to 300 fps on ZCU104
date: 2022-07-29T00:00:00
categories: [projects, tensil, fpga]
tags: [tensil, fpga]
language: en
slug: getting-resnet-to-300-fps-on-zcu104
---

- [Introduction](#introduction)
- [Baseline solution](#baseline-solution)
    - [Tensil RTL and Vivado implementation](#tensil-rtl-and-vivado-implementation)
    - [ResNet compiled for Tensil](#resnet-compiled-for-tensil)
    - [Tensil for Vitis embedded applications](#tensil-for-vitis-embedded-applications)
- [Dual clock solution](#dual-clock-solution)
- [Ultra RAM solution](#ultra-ram-solution)
- [Solutions with large local memory](#solutions-with-large-local-memory)


## Introduction

Sometimes the application requires pushing the performance to its limits. In this tutorial we will show how to optimize Tensil to run ResNet20 trained on CIFAR to reach the maximum performance. To do this we will use the powerful ZCU104 board and implement an embedded application to remove the overhead of running Linux OS and PYNQ. Importantly, we still won't quantize the model and use Tensil with 16-bit fixed points. We will demonstrate that running the CIFAR test data set shows very little accuracy drop when rounding down from the original 32-bit floating point of the [ResNet20 model](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/notebooks/resnet20v2_cifar.ipynb).

![board](/media/2022/tensil_zcu104_resnet/board.jpg)

We will be using Vivado 2021.1, which you can [download](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools/2021-2.html) and use for free for educational projects. Make sure to install Vitis, which will include Vivado and Vitis. We will use Vitis for building the embedded application.

Tensil tools are packaged in the form of Docker container, so you’ll need to have Docker installed and then pull [Tensil Docker image](https://hub.docker.com/r/tensilai/tensil) by running the following command.

```
docker pull tensilai/tensil
```

Use the following command to launch Tensil container.

```
docker run \
    -u $(id -u ${USER}):$(id -g ${USER}) \
    -v $(pwd):/work \
    -w /work \
    -it tensilai/tensil \
    bash
```

You will also need to clone the tutorial [GitHub repository](https://github.com/petrohi/tensil-zcu104-tutorial). It contains necessary source files as well as all of the intermediate artifacts in case you would like to skip running Tensil tools or Vivado. If you choose to run Tensil Docker image, make sure to launch Tensil container in the GitHub repository directory.


## Baseline solution

We start with the baseline solution, in which we will create a working Tensil application with default design choices. Once we have a stable working solution we will look at opportunities to improve its performance. For each such opportunity we will measure actual performance and hopefully see the improvement materialize.


### Tensil RTL and Vivado implementation

Let’s start with generating Tensil RTL. Run the following command to generate Verilog artifacts.

```
tensil rtl -a /demo/arch/zcu104.tarch -d 128 -s true -t vivado/baseline
```

You can skip running the Tensil RTL tool and grab the baseline Verilog artifacts [here](https://github.com/petrohi/tensil-zcu104-tutorial/tree/main/vivado/baseline).

Note that we need to specify the data width for AXI interfaces to be 128 bits, so that they fit directly with AXI ports on the ZYNQ UltraScale+ device at the heart of the ZCU104 board.

Next, create the new Vivado RTL project, select the ZCU104 board and import the three Verilog files produced by the Tensil RTL command.

We provide [scripted block design](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/vivado/baseline/tensil_zcu104.tcl), so that you won’t need to connect blocks manually. You can use _Tools -> Run Tcl Script_ to run it. Once you have the design ready, right-click on `tensil_zcu104` in the _Design Sources_ pane and select _Create HDL Wrapper_. Let Vivado manage the wrapper. Once the wrapper is ready, right-click on `tensil_zcu104_wrapper` in the same _Design Sources_ pane and select _Set as Top_.

Following is how the baseline block design looks like. Note the direct connections between Tensil block AXI masters and ZYNQ AXI slaves. The instruction AXI stream port is connected via AXI DMA block.


[![baseline_design](/media/2022/tensil_zcu104_resnet/baseline_design.png)](/media/2022/tensil_zcu104_resnet/baseline_design.png)

Next, click on _Generate Bitstream_ in the _Flow Navigator_ pane. Once bitstream is ready click on _File -> Export -> Export Hardware_. Make sure that _Include bitstream_ choice is selected. Now you have the XSA file containing everything necessary for Vitis to create the embedded application project.

You can skip the Vivado implementation and grab the baseline XSA file [here](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/vivado/baseline/tensil_zcu104_wrapper.xsa).

Another important result of running Vivado implementation is FPGA utilization. It is shown as one of the panes in the project summary once implementation is completed. The utilization is a direct function of our choice of [Tensil architecture](https://github.com/tensil-ai/tensil/blob/main/arch/zcu104.tarch). Specifying 32 by 32 systolic array size contributed to the high utilization of multiply-accumulate units (DSP). Note how we pushed Block RAM (BRAM) utilization almost to its limit by specifying 16 KV local memory and 4 KV accumulators (KV = 1024 vectors = 1024 * 32 * 16 bits). 


![baseline_util](/media/2022/tensil_zcu104_resnet/baseline_util.png)

### ResNet compiled for Tensil

The ZCU104 board supports an SD card interface. This allows us to use Tensil embedded driver file system functionality to read the ResNet model and a set of images to test it with. The set we will be using is the test set for the original [CIFAR-10 data set](https://www.cs.toronto.edu/~kriz/cifar.html). The ResNet model is trained with the separate training and validation sets from CIFAR-10. The test set is what the model hasn’t seen in training and therefore gives an objective estimate of its performance. The CIFAR-10 provides the test set of 10,000 images in several formats. We use [the binary format](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/sdcard/test_batch.bin) that is more suitable for the embedded application.

Let’s start with compiling the ResNet ONNX file to Tensil model artifacts. The following command will produce `*.tmodel`, `*.tdata` and `*.tprog` files under the `sdcard/baseline/` directory. The `*.tmodel` file is a JSON-formatted description of the Tensil model, which references `*.tdata` (model weights) and `*.tprog` (model program for the Tensil processor.)

```
tensil compile \
    -a /demo/arch/zcu104.tarch \
    -m /demo/models/resnet20v2_cifar.onnx \
    -o "Identity:0" \
    -s true \
    -t sdcard/baseline/
```

Once compiled, copy the content of the `sdcard` directory, which should also include the CIFAR-10 test data set (`test_batch.bin`) to the FAT-formatted SD card. Insert the card into the ZCU104 SD card slot. 

You can skip the model compilation step and use the [`sdcard` directory](https://github.com/petrohi/tensil-zcu104-tutorial/tree/main/sdcard) in our GitHub repository.


### Tensil for Vitis embedded applications

Launch Vitis IDE and create a new workspace for our project. Once greeted by the welcome screen click on _Create Application Project_. On the subsequent screen select _Create a new platform from hardware (XSA)_ and select the XSA file produced in the previous section. Name the application project `tensil_zcu104`. Keep default choices in the next few screens. On the last screen select _Empty Application (C)_.

Now let’s make one important adjustment to the platform. Right-click on the `tensil_zcu104 [Application]` project in the _Assistant_ pane and select _Navigate to BSP settings_. You will see a tree containing `psu_cortex53_0` (the ARM cores we will be running on) containing `zynqmp_fsbl` (first stage bootloader) and `standalone_psu_cortex53_0` (our application) sub-branches. _Board Support Package_ under standalone should be selected.

![bsp](/media/2022/tensil_zcu104_resnet/bsp.png)

In the _Board Support Package_ pane click on _Modify BSP settings_. In _Overview_ click the checkbox for `xilff`. This will enable the FAT filesystem driver for the SD card slot supported by the ZCU104 board. Click on `xilff` that should have appeared under _Overview_ in the left-side pane. Change `use_lfn` from 0 to 1. This will enable long file name support for the FAT filesystem driver.

![xilff](/media/2022/tensil_zcu104_resnet/xilff.png)

Before we copy all necessary source files, let's adjust application C/C++ build settings to link our application with the standard math library. In the _Explorer_ pane right-click the `tensil_zcu104` application project. Select _C/C++ Build Settings_ from the menu.

![explorer](/media/2022/tensil_zcu104_resnet/explorer.png)

Click on _Configuration_ dropdown and choose _All configurations_. Then, under _ARM v8 gcc linker_ click on _Libraries_. In the _Libraries_ pane click on the add button and enter `m`.

![m](/media/2022/tensil_zcu104_resnet/m.png)

Now, let’s copy all necessary source files. For this you will need to clone the tutorial GitHub repository as well as the Tensil GitHub repository for the embedded driver sources. But first, let's copy architecture definitions for the embedded driver from the output artifacts of Tensil RTL tool.

```
cp \
    <tutorial repository root>/vivado/baseline/architecture_params.h \
    <Vitis workspace root>/tensil_zcu104/src/
```

Next, copy the embedded driver.

```
cp -r \
    <Tensil repository root>/drivers/embedded/tensil \
    <Vitis workspace root>/tensil_zcu104/src/
```

Lastly, copy the ZCU104 embedded application. Note that `platform.h` copied in the previous step gets overwritten.

```
cp -r \
    <tutorial repository root>/vitis/* \
    <Vitis workspace root>/tensil_zcu104/src/
```

Finally, let's build and run the application. First, make sure the ZCU104 board has boot DIP switches (SW6) all in ON position (towards the center of the board) to enable booting from JTAG. Then, right-click on the _Debug_ entry under the _tensil_zcu104 [Application]_ project in the _Assistant_ pane and select _Debug -> Launch Hardware_.

With the SD card inserted and containing the CIFAR-10 test data set and the ResNet model compiled for Tensil you should see the inference printing every 100’s image and the corresponding prediction along with measured inferences (frames) per second.

![truck](/media/2022/tensil_zcu104_resnet/truck.png)

After running the inference on the entire test data set the program will print the final average frames per second and the accuracy of the inference. For the baseline solution we are getting an average of 133.54 frames per second with 90% accuracy. Note that the accuracy [we are seeing when testing](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/notebooks/resnet20v2_cifar.ipynb) the same ResNet model with TensorFlow is 92%. The 2% drop is due to changing the data type from 32-bit floating point in TensorFlow to 16-bit fixed point in Tensil.

## Dual clock solution

The first optimization is based on the following observation. The Tensil RTL block is clocked at 100MHz. (We could clock it higher, but for the purpose of this tutorial let’s assume this is our maximum clock.) The Tensil block DRAM0 and DRAM1 ports are connected to AXI interfaces on the ZYNQ block. The instruction port is indirectly connected to the AXI on the ZYNQ block via AXI DMA. ZYNQ UltraScal+ AXI ports support up to 333MHz and a maximum of 128 bits of width. This gives us the opportunity to introduce a second clock domain for 333MHz while at the same time making the Tensil AXI ports wider. 

The following diagram shows how this may work in a simpler 100MHz to 400MHz, 512- to 128-bit conversion. Each clock in the Tensil clock domain would pump one 512-bit word in or out. This would match 4 clocks in the ZYNQ clock domain with 512-bit word split to or composed from 4 128-bit words.

![dual_clock](/media/2022/tensil_zcu104_resnet/dual_clock.svg)

First let’s use the `-d` argument in Tensil RTL command to generate the RTL with 512-bit interfaces.

```
tensil rtl -a /demo/arch/zcu104.tarch -d 512 -s true -t vivado/dual_clock
```

The AXI SmartConnect block allows for both AXI width adjustment and separate clock domains. We change our block design by inserting these blocks in all three connections between Tensil RTL and the ZYNQ AXI ports. Again, we provide [scripted block design](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/vivado/dual_clock/tensil_zcu104.tcl), so that you won’t need to connect blocks manually. We suggest following the [steps above](#tensil-rtl-and-vivado-implementation) to create a new Vivado project for the dual clock design. Following is how the dual clock block design looks like.

[![dual_clock_design](/media/2022/tensil_zcu104_resnet/dual_clock_design.png)](/media/2022/tensil_zcu104_resnet/dual_clock_design.png)

You can skip the Vivado implementation and grab the dual clock XSA file [here](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/vivado/dual_clock/tensil_zcu104_wrapper.xsa).

Let’s take a look at FPGA utilization for dual clock design. Note that the increase is in LUT, LUTRAM and FF and not in BRAM or DSP. The reason is that we did not change the Tensil architecture, which would affect BRAM and DSP.

![dual_clock_util](/media/2022/tensil_zcu104_resnet/dual_clock_util.png)

Now, we suggest you also create a new Vitis workspace for the dual clock design and follow the [steps above](#tensil-for-vitis-embedded-applications) to get the inference running. The model remains unchanged since we did not change the architecture for the Tensil.

For the dual clock solution we are getting an average of 152.04 frames per second, a meaningful improvement over the baseline. This improvement is roughly proportional to the ratio of time spent in moving data to and from the FPGA to the time spent in internal data movement and computation.

## Ultra RAM solution

The second optimization is based on the higher-end ZYNQ UltraScale+ devices support for another type of on-chip memory called Ultra RAM. By default Vivado maps dual-port memory to Block RAM. In order for it to map to the Ultra RAM it needs hints in the Verilog code. The amount of Ultra-RAM available on ZCU104 allows us to add around 48 KV memory in addition to 20 KV available through Block RAM.

We start by creating a new [Tensil architecture](https://github.com/tensil-ai/tensil/blob/main/arch/zcu104_uram.tarch) for ZCU104 in which we allocate all of the Block RAM (20 KV) to accumulators and all of the Ultra RAM (48 KV) to local memory.

Run the following command to generate Verilog artifacts.

```
tensil rtl \
    -a /demo/arch/zcu104_uram.tarch \
    -d 512 \
    --use-xilinx-ultra-ram true \
    -s true \
    -t vivado/ultra_ram
```

You can also skip running the Tensil RTL tool and grab the Ultra RAM Verilog artifacts [here](https://github.com/petrohi/tensil-zcu104-tutorial/tree/main/vivado/ultra_ram).

Follow the [steps above](#tensil-rtl-and-vivado-implementation) to create a new Vivado project for Ultra RAM solution. We provide [scripted block design](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/vivado/ultra_ram/tensil_zcu104.tcl), so that you won’t need to connect blocks manually. Following is how the Ultra RAM block design looks like. Note, that we based it on the dual clock design and the only difference is the Tensil RTL block name.

[![ultra_ram_design](/media/2022/tensil_zcu104_resnet/ultra_ram_design.png)](/media/2022/tensil_zcu104_resnet/ultra_ram_design.png)

You can skip the Vivado implementation and grab the Ultra RAM XSA file [here](https://github.com/petrohi/tensil-zcu104-tutorial/blob/main/vivado/ultra_ram/tensil_zcu104_wrapper.xsa).

Now, let’s take a look at FPGA utilization for Ultra RAM design. Note that the utilization is mostly unchanged from the one of the dual clock design. The exception is the new line item for Ultra RAM (URAM), which we pushed to its full limit.

![ultra_ram_util](/media/2022/tensil_zcu104_resnet/ultra_ram_util.png)

Because we changed the Tensil architecture the new model needs to be compiled and copied to the SD card.

```
tensil compile -a /demo/arch/zcu104_uram.tarch -m /demo/models/resnet20v2_cifar.onnx -o "Identity:0" -s true -t sdcard/ultra_ram/
```

You can skip the model compilation step and use the [`sdcard` directory](https://github.com/petrohi/tensil-zcu104-tutorial/tree/main/sdcard) in our GitHub repository.

We again, suggest you create a new Vitis workspace for the Ultra RAM design and follow the [steps above](#tensil-for-vitis-embedded-applications) to get the inference running. Make sure to uncomment the correct `MODEL_FILE_PATH` definition for the newly created `*.tmodel` file.

For the Ultra RAM solution we are getting an average of 170.16 frames per second, another meaningful improvement. This improvement is purely based on having larger on-chip memory. With a small on-chip memory the Tensil compiler is forced to partition ResNet convolution layers into multiple load-compute-save blocks. This, in turn, requires that the activations are loaded multiple times since weights are loaded only once. (This is called weight-stationery ordering. In the future we plan to add an option for input-stationery ordering.)

The following diagram shows such partitioned compilation. Layer N has 2 stages. In each stage a unique subset of weights is loaded. Then, each stage is further split into 2 partitions. Partition is defined by the largest amount of weights, input and output activations, and intermediate results that fit local memory and accumulators.

![multi_stage](/media/2022/tensil_zcu104_resnet/multi_stage.svg)

Having larger on-chip memory reduces this partitioning and, by extension, the need to load the same data multiple times. The following diagram shows how layer N now has 1 stage and 1 partition that fits larger local memory and accumulators allowing weights and activations to be loaded only once.

![single_stage](/media/2022/tensil_zcu104_resnet/single_stage.svg)

## Solutions with large local memory

The final optimization is based on the hardware design and Tensil architecture definition we created to support the Ultra RAM. We will only change the Tensil compiler strategy.

As we mentioned previously, the Tensil compiler, by default, assumes that model layers will be much larger in terms of its weights and activations size than the local memory available on the FPGA. This is definitely true for large models and for low-end FPGA devices. For small and medium size models running on large FPGA devices there is a distinct possibility that local memory is large enough to contain the entire weights for each layer as well as activations for each pair of the previous and the next layer in the model. To see if this strategy is worth trying we first look at the output of Tensil compiler for the ZCU104 architecture with the Ultra RAM.

![compiler_summary](/media/2022/tensil_zcu104_resnet/compiler_summary.svg)

The maximum number for stages and partitions being both 1 inform us that none of the layers were partitioned, or, in other words each layer’s weights and activations did fit in local memory. Another way to guide this decision is to use the `--layers-summary true` option with the Tensil compiler, which will report the summary per each layer with local and accumulator utilization.

![layers_summary](/media/2022/tensil_zcu104_resnet/layers_summary.svg)

Thus, the first strategy will be to try keeping activations in local memory between layers by specifying `--strategy local-vars`. The following diagram shows this strategy.

![shared_local_vars](/media/2022/tensil_zcu104_resnet/shared_local_vars.svg)

Run the following command and then copy the newly created model to the SD card.

```
tensil compile \
    -a /demo/arch/zcu104_uram.tarch \
    -m /demo/models/resnet20v2_cifar.onnx \
    -o "Identity:0" \
    -s true \
    --strategy local-vars \
    -t sdcard/ultra_ram_local_vars/
```

You can skip all of the model compilation steps in the section and use the [`sdcard` directory](https://github.com/petrohi/tensil-zcu104-tutorial/tree/main/sdcard) in our GitHub repository.

This time, you can reuse the Vitis workspace for Ultra RAM solution and simply uncomment the correct `MODEL_FILE_PATH` definition for the newly created `*.tmodel` file.

With the `local-vars` strategy we are getting an average of 214.66 frames per second.

Now that we have seen the improvement allowed by large on-chip memory, let’s see if any other load and save operations can be avoided. With `local-vars` strategy we load model input image and weights and then save its output predictions. What if there would be enough on-chip memory to keep weights loaded? There is a strategy for this!

With the `local-consts` strategy the inference expects all of the model weights to be preloaded to the local memory before the inference. This is the job of the driver. If it loads the model compiled with `local-consts` strategy it also preloads its weights from `*.tdata` file into Tensil local memory. The following diagram shows this strategy.

![shared_local_consts](/media/2022/tensil_zcu104_resnet/shared_local_consts.svg)

```
tensil compile \
    -a /demo/arch/zcu104_uram.tarch \
    -m /demo/models/resnet20v2_cifar.onnx \
    -o "Identity:0" \
    -s true \
    --strategy local-consts \
    -t sdcard/ultra_ram_local_consts/
```

With the `local-consts` strategy we are getting an average of 231.57 frames per second.

Finally, the `local-vars-and-consts` strategy combines `local-vars` and `local-consts`. With this strategy the inference will only load the input image and save the output predictions. The following diagram shows this strategy.

![shared_local_vars_and_consts](/media/2022/tensil_zcu104_resnet/shared_local_vars_and_consts.svg)

```
tensil compile \
    -a /demo/arch/zcu104_uram.tarch \
    -m /demo/models/resnet20v2_cifar.onnx \
    -o "Identity:0" \
    -s true \
    --strategy local-vars-and-consts \
    -t sdcard/ultra_ram_local_vars_and_consts/
```

With the `local-vars-and-consts` strategy we are getting an average of 293.58 frames per second.

![resnet_300fps](/media/2022/tensil_zcu104_resnet/resnet_300fps.gif)

# Conclusion

In this tutorial we demonstrated how improvements in Vivado hardware design, leveraging Xilinx Ultra RAM with corresponding changes in Tensil architecture, and advanced strategies in Tensil compiler can improve the performance of inference for the ResNet20 model running on the ZCU104 board.

The following chart summarizes presented solutions and their frame per second performance.

![fps](/media/2022/tensil_zcu104_resnet/fps.svg)