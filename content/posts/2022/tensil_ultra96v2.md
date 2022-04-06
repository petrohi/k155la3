---
title: Tensil tutorial for Ultra96 V2
date: 2022-03-06T00:00:00
categories: [projects, tensil, fpga]
tags: [tensil, fpga, verilog, chisel]
language: en
slug: tensil-tutorial-for-ultra96-v2
---

## Introduction

This tutorial will use the [Avnet Ultra96 V2](https://www.avnet.com/wps/portal/us/products/avnet-boards/avnet-board-families/ultra96-v2/) development board and [Tensil's open-source inference accelerator](https://www.tensil.ai/) to show how to run machine learning (ML) models on FPGA. We will be using ResNet-20 trained on the CIFAR dataset. These steps should work for any supported ML model -- currently all the common state-of-the-art convolutional neural networks are supported. Try it with your model!

We'll give detailed end-to-end coverage that is easy to follow. In addition, we include in-depth explanations to get a good understanding of the technology behind it all, including the Tensil and [Xilinx Vivado](https://www.xilinx.com/products/design-tools/vivado.html) toolchains and [PYNQ framework](http://www.pynq.io).

If you get stuck or find an error, you can ask a question on our [Discord](https://discord.gg/TSw34H3PXr) or send an email to [support@tensil.ai](mailto:support@tensil.ai).

![board](/media/2022/tensil_ultra96v2/board.webp)

## Overview

Before we start, let's look at the Tensil toolchain flow to get a bird's eye view of what we want to accomplish. We'll follow these steps:

1. [Get Tensil](#1-get-tensil)
2. [Choose architecture](#2-choose-architecture)
3. [Generate TCU accelerator design (RTL code)](#3-generate-tcu-accelerator-design-rtl-code)
4. [Synthesize for Ultra96](#4-synthesize-for-ultra96)
5. [Compile ML model for TCU](#5-compile-ml-model-for-tcu)
6. [Execute using PYNQ](#6-execute-using-pynq)


![flow](/media/2022/tensil_ultra96v2/flow.png)

## 1. Get Tensil

[Back to top](#overview)

First, we need to get the Tensil toolchain. The easiest way is to pull the Tensil docker container from Docker Hub. The following command will pull the image and then run the container.

```bash
docker pull tensilai/tensil
docker run -v $(pwd):/work -w /work -it tensilai/tensil bash
```

## 2. Choose architecture

[Back to top](#overview)

Tensil's strength is customizability, making it suitable for a very wide range of applications. The Tensil architecture definition file (.tarch) specifies the parameters of the architecture to be implemented. These parameters are what make Tensil flexible enough to work for small embedded FPGAs as well as large data-center FPGAs. Our example will select parameters that provide the highest utilization of resources on the ZU3EG FPGA part at the core of the Ultra96 board. The container image conveniently includes the architecture file for the Ultra96 development board at `/demo/arch/ultra96v2.tarch`. Let's take a look at what's inside.

```json
{
    "data_type": "FP16BP8",
    "array_size": 16,
    "dram0_depth": 2097152,
    "dram1_depth": 2097152,
    "local_depth": 20480,
    "accumulator_depth": 4096,
    "simd_registers_depth": 1,
    "stride0_depth": 8,
    "stride1_depth": 8
}
```

The file contains a JSON object with several parameters. The first, `data_type`, defines the data type used throughout the Tensor Compute Unit (TCU), including in the systolic array, SIMD ALUs, accumulators, and local memory. We will use 16-bit fixed-point with an 8-bit base point (`FP16BP8`), which in most cases allows simple rounding of 32-bit floating-point models without the need for quantization. Next, `array_size` defines a systolic array size of 16x16, which results in 256 parallel multiply-accumulate (MAC) units. This number was chosen to maximize the utilization of DSP units available on ZU3EG, but if you needed to use some DSPs for another application in parallel, you could decrease it to free some up.

With `dram0_depth` and `dram1_depth`, we define the size of DRAM0 and DRAM1 memory buffers on the host side. These buffers feed the TCU with the model's weights and inputs, and also store intermediate results and outputs. Note that these memory sizes are in number of vectors, which means array size (16) multiplied by data type size (16-bits) for a total of 256 bits per vector.

Next, we define the size of the `local` and `accumulator` memories which will be implemented on the FPGA fabric itself. The difference between the accumulators and the local memory is that accumulators can perform a write-accumulate operation in which the input is added to the data already stored, as opposed to simply overwriting it. The total size of accumulators plus local memory is again selected to maximize the utilization of BRAM resources on ZU3EG, but if necessary you could reduce these to free up resources needed elsewhere.

With `simd_registers_depth`, we specify the number of registers included in each SIMD ALU, which can perform SIMD operations on stored vectors used for ML operations like ReLU activation. Increasing this number is only needed rarely, to help compute special activation functions. Finally, `stride0_depth` and `stride1_depth` specify the number of bits to use for enabling "strided" memory reads and writes. It's unlikely you'll ever need to change this parameter.

## 3. Generate TCU accelerator design (RTL code)

[Back to top](#overview)

Now that we've selected our architecture, it's time to run the Tensil RTL generator. RTL stands for "Register Transfer Level" -- it's a type of code that specifies digital logic stuff like wires, registers and low-level logic. Special tools like Xilinx Vivado or [yosys](https://yosyshq.net/yosys/) can synthesize RTL for FPGAs and even ASICs.

To generate a design using our chosen architecture, run the following command inside the Tensil toolchain docker container:

```bash
tensil rtl -a /demo/arch/ultra96v2.tarch -s true -d 128
```

Note the `-d 128` parameter, which specifies that the generated RTL will be compatible with 128-bit AXI interfaces supported by the ZU3EG part. This command will produce several Verilog files listed in the `ARTIFACTS` table printed out at the end. It also prints the `RTL SUMMARY` table with some of the essential parameters of the resulting RTL.

```
-----------------------------------------------------------------------
RTL SUMMARY
-----------------------------------------------------------------------
Data type:                                      FP16BP8   
Array size:                                     16        
Consts memory size (vectors/scalars/bits):      2,097,152 33,554,432 21
Vars memory size (vectors/scalars/bits):        2,097,152 33,554,432 21
Local memory size (vectors/scalars/bits):       20,480    327,680    15
Accumulator memory size (vectors/scalars/bits): 4,096     65,536     12
Stride #0 size (bits):                          3         
Stride #1 size (bits):                          3         
Operand #0 size (bits):                         24        
Operand #1 size (bits):                         24        
Operand #2 size (bits):                         16        
Instruction size (bytes):                       9         
-----------------------------------------------------------------------
```

## 4. Synthesize for Ultra96

[Back to top](#overview)

It is now time to start Xilinx Vivado. I will be using version 2021.2, which you can download free of charge (for prototyping) at the [Xilinx website](https://www.xilinx.com/support/download.html).

First, create a new RTL project named `tensil-ultra96v2` and add Verilog files generated by the Tensil RTL tool.

![new_project_rtl](/media/2022/tensil_ultra96v2/new_project_rtl.png)

Choose boards and search for Ultra96. Select Ultra96-V2 Single Board Computer with file version 1.2. You may need to click the Install icon in the Status column. (If you don't find the board, click on the Refresh button below.)

![new_project_board](/media/2022/tensil_ultra96v2/new_project_board.png)

Under IP INTEGRATOR, click Create Block Design.

![create_design](/media/2022/tensil_ultra96v2/create_design.png)

Drag `top_ultra96v2` from the Sources tab onto the block design diagram. You should see the Tensil RTL block with its interfaces.

![design_tensil_rtl](/media/2022/tensil_ultra96v2/design_tensil_rtl.png)

Next, click the plus `+` button in the Block Diagram toolbar (upper left) and select "Zynq UltraScale+ MPSoC" (you may need to use the search box). Do the same for "Processor System Reset". The Zynq block represents the "hard" part of the Xilinx platform, which includes ARM processors, DDR interfaces, and much more. The Processor System Reset is a utility box that provides the design with correctly synchronized reset signals.

Click "Run Block Automation" and "Run Connection Automation". Check "All Automation".

Double-click Zynq UltraScale+ MPSoC. First, go to Clock Configuration and ensure PL Fabric Clocks have PL0 checked and set to 100MHz.

![zynq_clocks](/media/2022/tensil_ultra96v2/zynq_clocks.png)

Then, go to PS-PL Configuration. Uncheck AXI HPM1 FPD and check AXI HP1 FPD, AXI HP2 FPD, and AXI HP3 FPD. These changes will configure all the necessary interfaces between Processing System (PS) and Programmable Logic (PL) necessary for our design.

![zynq_ps_pl](/media/2022/tensil_ultra96v2/zynq_ps_pl.png)

Now, connect `m_axi_dram0` and `m_axi_dram1 ports` on Tensil block to `S_AXI_HP1_FPD` and `S_AXI_HP2_FPD` on Zynq block correspondingly. The TCU has two DRAM banks to enable their parallel operation by utilizing separate PS ports.

Next, click the plus `+` button in the Block Diagram toolbar and select "AXI Direct Memory Access" (DMA). The DMA block is used to organize the feeding of the Tensil program to the TCU without keeping the PS ARM processor busy.

Double-click it. Disable "Scatter Gather Engine" and "Write Channel". Change "Width of Buffer Length Register" to be 26 bits. Select "Memory Map Data Width" and "Stream Data Width" to be 128 bits. Change "Max Burst Size" to 256.

![dma](/media/2022/tensil_ultra96v2/dma.png)

Connect the `instruction` port on the Tensil `top` block to `M_AXIS_MM2S` on the AXI DMA block. Then, connect `M_AXI_MM2S` on the AXI DMA block to `S_AXI_HP1_FPD` on Zynq.

Once again, click the plus `+` button in the Block Diagram toolbar and select "AXI SmartConnect. The SmartConnect is necessary to expose DMA control registers to the Zynq CPU, which will enable software to control the DMA transactions. Double-click it and set "Number of Slave and Master Interfaces" to 1. 

![smartconnect](/media/2022/tensil_ultra96v2/smartconnect.png)

Connect `M00_AXI` on the AXI SmartConnect block to `S_AXI_LITE` on the AXI DMA block. Connect `S00_AXI` on the AXI SmartConnect to `M_AXI_HPM0_FPD` on the Zynq block.

Finally, click "Run Connection Automation" and check "All Automation". By doing this, we connect all the clocks and resets. Click the "Regenerate Layout" button in the Block Diagram toolbar to make the diagram look nice.

![design_final](/media/2022/tensil_ultra96v2/design_final.png)

Next, switch to the "Address Editor" tab. Click the "Assign All" button in the toolbar. By doing this, we assign address spaces to various AXI interfaces. For example, `m_axi_dram0` and `m_axi_dram1` gain access to the entire address space on the Ultra96 board, including DDR memory and control register spaces. We only need access to DDR, so you can manually exclude the register address space if you know what you're doing.

![design_address](/media/2022/tensil_ultra96v2/design_address.png)

Back in the Block Diagram tab, click the "Validate Design" (or F6) button. You should see the message informing you of successful validation! You can now close the Block Design by clicking `x` in the right upper corner.

The final step is to create the HDL wrapper for our design, which will tie everything together and enable synthesis and implementation. Right-click the `tensil_ultra96v2` item in the Sources tab and choose "Create HDL Wrapper". Keep "Let Vivado manage wrapper and auto-update" selected. Wait for the Sources tree to be fully updated and right-click on `tensil_ultra96v2_wrapper`. Choose Set as Top.

Now it's time to let Vivado perform synthesis and implementation and write the resulting bitstream. In the Flow Navigator sidebar, click on "Generate Bitstream" and hit OK. Vivado will start synthesizing our Tensil design -- this may take around 15 minutes. When done, you can observe some vital stats in the Project Summary. First, look at utilization, which shows what percentage of each FPGA resource our design is using. Note how we pushed BRAM and DSP resources to high utilization.

![utilization](/media/2022/tensil_ultra96v2/utilization.png)

The second is timing, which tells us about how long it takes for signals to propagate in our programmable logic (PL). The "Worst Negative Slack" being a positive number is good news -- our design meets propagation constraints for all nets at the specified clock speed!

![timing](/media/2022/tensil_ultra96v2/timing.png)

## 5. Compile ML model for TCU

[Back to top](#overview)

The second branch of the Tensil toolchain flow is to compile the ML model to a Tensil binary consisting of TCU instructions, which are executed by the TCU hardware directly. For this tutorial, we will use ResNet20 trained on the CIFAR dataset. The model is included in the Tensil docker image at `/demo/models/resnet20v2_cifar.onnx`. From within the Tensil docker container, run the following command.

```bash
tensil compile -a /demo/arch/ultra96v2.tarch -m /demo/models/resnet20v2_cifar.onnx -o "Identity:0" -s true
```
We're using the ONNX version of the model, but the Tensil compiler also supports TensorFlow, which you can try by compiling the same model in TensorFlow frozen graph form at `/demo/models/resnet20v2_cifar.pb`. 

```bash
tensil compile -a /demo/arch/ultra96v2.tarch -m /demo/models/resnet20v2_cifar.pb -o "Identity" -s true
```

The resulting compiled files are listed in the `ARTIFACTS` table. The manifest (`tmodel`) is a plain text JSON description of the compiled model. The Tensil program (`tprog`) and weights data (`tdata`) are both binaries to be used by the TCU during execution. The Tensil compiler also prints a `COMPILER SUMMARY` table with interesting stats for both the TCU architecture and the model.

```
----------------------------------------------------------------------------------------------
COMPILER SUMMARY
----------------------------------------------------------------------------------------------
Model:                                           resnet20v2_cifar_onnx_ultra96v2 
Data type:                                       FP16BP8                         
Array size:                                      16                              
Consts memory size (vectors/scalars/bits):       2,097,152                       33,554,432 21
Vars memory size (vectors/scalars/bits):         2,097,152                       33,554,432 21
Local memory size (vectors/scalars/bits):        20,480                          327,680    15
Accumulator memory size (vectors/scalars/bits):  4,096                           65,536     12
Stride #0 size (bits):                           3                               
Stride #1 size (bits):                           3                               
Operand #0 size (bits):                          24                              
Operand #1 size (bits):                          24                              
Operand #2 size (bits):                          16                              
Instruction size (bytes):                        9                               
Consts memory maximum usage (vectors/scalars):   35,743                          571,888    
Vars memory maximum usage (vectors/scalars):     13,312                          212,992    
Consts memory aggregate usage (vectors/scalars): 35,743                          571,888    
Vars memory aggregate usage (vectors/scalars):   46,097                          737,552    
Number of layers:                                23                              
Total number of instructions:                    102,741                         
Compilation time (seconds):                      30.066                          
True consts scalar size:                         568,474                         
Consts utilization (%):                          97.210                          
True MACs (M):                                   61.476                          
MAC efficiency (%):                              0.000                           
----------------------------------------------------------------------------------------------
```

## 6. Execute using PYNQ

[Back to top](#overview)

Now it's time to put everything together on our development board. For this, we first need to set up the PYNQ environment. This process starts with downloading the [SD card image for our development board](http://www.pynq.io/board.html). There's the [detailed instruction](https://ultra96-pynq.readthedocs.io/en/latest/getting_started.html) for setting board connectivity on the PYNQ documentation website. You should be able to open Jupyter notebooks and run some examples.

There is one caveat that needs addressing once PYNQ is installed. On the default PYNQ image, the setting for the Linux kernel [CMA (Contiguous Memory Allocator)](https://elinux.org/images/2/23/LinuxCMA-cewg43.pdf) area size is 128MB. Given our Tensil architecture, the default CMA size is too small. To address this, you'll need to download our patched kernel, copy it to `/boot`, and reboot your board. Note that the patched kernel is built for PYNQ 2.7 and will not work with other versions. To patch the kernel, run these commands:

```bash
wget https://s3.us-west-1.amazonaws.com/downloads.tensil.ai/pynq/2.7/ultra96v2/image.ub
scp image.ub xilinx@192.168.3.1:
ssh xilinx@192.168.3.1
sudo cp /boot/image.ub /boot/image.ub.backup
sudo cp image.ub /boot/
rm image.ub
sudo reboot
```

Now that PYNQ is up and running, the next step is to `scp` the Tensil driver for PYNQ. Start by cloning the [Tensil GitHub repository](https://github.com/tensil-ai/tensil) to your work station and then copy `drivers/tcu_pynq` to `/home/xilinx/tcu_pynq` onto your board.

```bash
git clone git@github.com:tensil-ai/tensil.git
scp -r tensil/drivers/tcu_pynq xilinx@192.168.3.1:
```

We also need to `scp` the bitstream and compiler artifacts.

Next we'll copy over the bitstream, which contains the FPGA configuration resulting from Vivado synthesis and implementation. PYNQ also needs a hardware handoff file that describes FPGA components accessible to the host, such as DMA. Place both files in `/home/xilinx` on the development board. Assuming you are in the Vivado project directory, run the following commands to copy files over.

```bash
scp tensil-ultra96v2.runs/impl_1/tensil_ultra96v2_wrapper.bit xilinx@192.168.3.1:tensil_ultra96v2.bit
scp tensil-ultra96v2.gen/sources_1/bd/tensil_ultra96v2/hw_handoff/tensil_ultra96v2.hwh xilinx@192.168.3.1:
```

Note that we renamed bitstream to match the hardware handoff file name.

Now, copy the `.tmodel`, `.tprog` and `.tdata` artifacts produced by the compiler to `/home/xilinx` on the board.

```bash
scp resnet20v2_cifar_onnx_ultra96v2.t* xilinx@192.168.3.1:
```

The last thing needed to run our ResNet model is the CIFAR dataset. You can get it from [Kaggle](https://www.kaggle.com/janzenliu/cifar-10-batches-py) or run the commands below (since we only need the test batch, we remove the training batches to reduce the file size). Put these files in `/home/xilinx/cifar-10-batches-py/` on your development board.

```bash
wget http://www.cs.toronto.edu/~kriz/cifar-10-python.tar.gz
tar xfvz cifar-10-python.tar.gz
rm cifar-10-batches-py/data_batch_*
scp -r cifar-10-batches-py xilinx@192.168.3.1:
```

We are finally ready to fire up the PYNQ Jupyter notebook and run the ResNet model on TCU.

### Jupyter notebook

First, we import the Tensil PYNQ driver and other required utilities.

```python
import sys
sys.path.append('/home/xilinx')

# Needed to run inference on TCU
import time
import numpy as np
import pynq
from pynq import Overlay
from tcu_pynq.driver import Driver
from tcu_pynq.architecture import ultra96

# Needed for unpacking and displaying image data
%matplotlib inline
import matplotlib.pyplot as plt
import pickle
```

Now, initialize the PYNQ overlay from the bitstream and instantiate the Tensil driver using the TCU architecture and the overlay's DMA configuration. Note that we are passing `axi_dma_0` object from the overlay -- the name matches the DMA block in the Vivado design.

```python
overlay = Overlay('/home/xilinx/tensil_ultra96v2.bit')
tcu = Driver(ultra96, overlay.axi_dma_0)
```

The Tensil PYNQ driver includes the Ultra96 architecture definition. Here it is in an excerpt from `architecture.py`: you can see that it matches the architecture we used previously.

```python
ultra96 = Architecture(
    data_type=DataType.FP16BP8,
    array_size=16,
    dram0_depth=2097152,
    dram1_depth=2097152,
    local_depth=20480,
    accumulator_depth=4096,
    simd_registers_depth=1,
    stride0_depth=8,
    stride1_depth=8,
)
```

Next, let's load CIFAR images from the `test_batch`.

```python
def unpickle(file):
    with open(file, 'rb') as fo:
        d = pickle.load(fo, encoding='bytes')
    return d

cifar = unpickle('/home/xilinx/cifar-10-batches-py/test_batch')
data = cifar[b'data']
labels = cifar[b'labels']

data = data[10:20]
labels = labels[10:20]

data_norm = data.astype('float32') / 255
data_mean = np.mean(data_norm, axis=0)
data_norm -= data_mean

cifar_meta = unpickle('/home/xilinx/cifar-10-batches-py/batches.meta')
label_names = [b.decode() for b in cifar_meta[b'label_names']]

def show_img(data, n):
    plt.imshow(np.transpose(data[n].reshape((3, 32, 32)), axes=[1, 2, 0]))

def get_img(data, n):
    img = np.transpose(data_norm[n].reshape((3, 32, 32)), axes=[1, 2, 0])
    img = np.pad(img, [(0, 0), (0, 0), (0, tcu.arch.array_size - 3)], 'constant', constant_values=0)
    return img.reshape((-1, tcu.arch.array_size))

def get_label(labels, label_names, n):
    label_idx = labels[n]
    name = label_names[label_idx]
    return (label_idx, name)
```

To test, extract one of the images.

```python
n = 9
img = get_img(data, n)
label_idx, label = get_label(labels, label_names, n)
show_img(data, n)
```

You should see the image.

![frog](/media/2022/tensil_ultra96v2/frog.png)

Next, load the `tmodel` manifest for the model into the driver. The manifest tells the driver where to find the other two binary files (program and weights data).

```python
tcu.load_model('/home/xilinx/resnet20v2_cifar_onnx_ultra96v2.tmodel')
```

Finally, run the model and print the results! The call to `tcu.run(inputs)` is where the magic happens. We'll convert the ResNet classification result vector into CIFAR labels. Note that if you are using the ONNX model, the input and output are named `x:0` and `Identity:0` respectively. For the TensorFlow model they are named `x` and `Identity`.

```python
inputs = {'x:0': img}

start = time.time()
outputs = tcu.run(inputs)
end = time.time()
print("Ran inference in {:.4}s".format(end - start))
print()

classes = outputs['Identity:0'][:10]
result_idx = np.argmax(classes)
result = label_names[result_idx]
print("Output activations:")
print(classes)
print()
print("Result: {} (idx = {})".format(result, result_idx))
print("Actual: {} (idx = {})".format(label, label_idx))
```

Here is the expected result:

```
Ran inference in 0.03043s

Output activations:
[-13.59375    -12.25        -7.90625     -6.21484375  -8.25
 -12.24609375  15.0390625  -15.10546875 -10.71875     -9.1796875 ]

Result: frog (idx = 6)
Actual: frog (idx = 6)
```

Congratulations! You ran a machine learning model a custom ML accelerator that you built on your own work station! Just imagine the things you could do with it...


## Wrap-up

[Back to top](#overview)

In this tutorial we used Tensil to show how to run machine learning (ML) models on FPGA. We went through a number of steps to get here, including installing Tensil, choosing an architecture, generating an RTL design, synthesizing the desing, compiling the ML model and finally executing the model using PYNQ.

If you made it all the way through, big congrats! You're ready to take things to the next level by trying out your own model and architecture. Join us on [Discord](https://discord.gg/TSw34H3PXr) to say hello and ask questions, or send an email to [support@tensil.ai](mailto:support@tensil.ai).