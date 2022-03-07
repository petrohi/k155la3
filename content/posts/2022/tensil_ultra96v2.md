---
title: Tensil tutorial for Ultra96 V2
date: 2022-03-06T00:00:00
categories: [projects, tensil, fpga]
tags: [tensil, fpga, verilog, chisel]
language: en
slug: tensil-tutorial-for-ultra96-v2
---

## Introduction

This tutorial will use [Avnet Ultra96 V2](https://www.avnet.com/wps/portal/us/products/avnet-boards/avnet-board-families/ultra96-v2/) development board and [Tensil open-source inference accelerator](https://www.tensil.ai/) to show how to run ML models on FPGA. We will be using ResNet20 trained on the CIFAR dataset. Still, the steps also apply to your model--you can try it! (We currently support most key ML operations commonly used in state-of-the-art convolutional neural networks.) We'll give detailed end-to-end coverage that is easy to follow. In addition, we include in-depth explanations to get a good understanding of the Tensil and [Xilinx Vivado](https://www.xilinx.com/products/design-tools/vivado.html) toolchains and [PYNQ framework](http://www.pynq.io).

![board](/media/2022/tensil_ultra96v2/board.webp)

Before we start, let's look at the Tensil toolchain flow to get a bird's eye view of what we want to accomplish. 

![flow](/media/2022/tensil_ultra96v2/flow.png)

## Tensil architecture

The flow starts with the Tensil architecture definition file (tarch) that contains the parameters of the architecture we plan to implement. These parameters give great flexibility and are the cornerstone of Tensil. Our example will select parameters that will provide the highest utilization of the ZU3EG FPGA part, a centerpiece of the Ultra96 development board.

Before we start the flow, we need to get the Tensil toolchain. The easiest way is to pull the Tensil docker container from Docker Hub. The following command will pull the image and then run the container.

```bash
docker pull tensilai/tensil:latest
docker run -v $(pwd):/work -w /work -it tensilai/tensil:latest bash
```

The first branch of Tensil toolchain flow is to create RTL for a specified Tensil architecture. The container image conveniently includes the architecture file for the Ultra96 development board at `/demo/arch/ultra96v2.tarch`. Let's take a look at what's inside.

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

The file contains a JSON object with several parameters. It defines the data type used throughout Tensil compute unit (TCU), including systolic array, SIMD ALUs, accumulators, and local memory. We will use a 16-bit fixed-point with an 8-bit base point (`FP16BP8`), which in most cases allows simple rounding of 32-bit floating-point models without the need for quantization. Next, we define a systolic array size of 16, which results in 256 parallel multiply-accumulate (MAC) units. This number is picked to maximize the utilization of DSP units available on ZU3EG. 

Then, we define the size of DRAM0 and DRAM1 pools on the host side to feed the TCU with the model's weights and the input and store intermediate results and outputs. Note that memory sizes are in vectors, which means array size (16) multiplied by data type size (16-bits). Next, we define the size of local and accumulators memories used as cache on the FPGA.

The difference between accumulators and local memory is that accumulators can perform SIMD operations on stored vectors used for ML operations like activation. The total size of accumulators plus local memory is again selected to maximize the utilization of BRAM resources on ZU3EG. Finally, the architecture specifies the number of registers included in SIMD ALU and available stride bits for enabling "striped" memory reads and writes.

## RTL

Run the Tensil RTL tool with the following command inside the Tensil toolchain docker.

```bash
tensil rtl -a /demo/arch/ultra96v2.tarch -d 128
```

Note the `-d 128` parameter, which defines that RTL will be compatible with 128-bit AXI interfaces supported by the ZU3EG part. This command will produce several Verilog files listed in the ARTIFACTS table. It also prints the RTL SUMMARY table with some essential parameters of the resulting RTL, particularly the Instruction Size in bytes we'll use when designing in Vivado.

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

## Vivado

It is now time to start Xilinx Vivado. I will be using version 2021.2, which you can download free of charge (for prototyping) at the [Xilinx website](https://www.xilinx.com/support/download.html).

First, create a new RTL project named `tensil-ultra96v2` and add Verilog files generated by the Tensil RTL tool.

![new_project_rtl](/media/2022/tensil_ultra96v2/new_project_rtl.png)

Choose boards and search for Ultra96. Select Ultra96-V2 Single Board Computer with file version 1.2. You may need to click the Install icon in the Status column. (If you don't find the board, click on the Refresh button below.)

![new_project_board](/media/2022/tensil_ultra96v2/new_project_board.png)

Under IP INTEGRATOR, click Create Block Design.

![create_design](/media/2022/tensil_ultra96v2/create_design.png)

Drag `top_ultra96v2` from the Sources tab onto the block design diagram. You should see the Tensil RTL block with its interfaces.

![design_tensil_rtl](/media/2022/tensil_ultra96v2/design_tensil_rtl.png)

Next, click plus button in the Diagram toolbar and select Zynq UltraScale+ MPSoC. Do the same for Processor System Reset. The Zynq block represents the "hard" part of the Xilinx platform, which includes ARM processors, DDR interfaces, and much more. The Processor System Reset is a utility box that provides the design with correctly synchronized reset signals.

Click Run Block Automation and Run Connection Automation. Check All Automation.

Double-click Zynq UltraScale+ MPSoC. First, go to Clock Configuration and ensure PL Fabric Clocks have PL0 checked and set to 100MHz.

![zynq_clocks](/media/2022/tensil_ultra96v2/zynq_clocks.png)

Then, go to PS-PL Configuration. Uncheck AXI HPM1 FPD and check AXI HP1 FPD, AXI HP2 FPD, and AXI HP3 FPD. These changes will configure all necessary interfaces between Processing System (PS) and Programmable Logic (PL) necessary for our design.

![zynq_ps_pl](/media/2022/tensil_ultra96v2/zynq_ps_pl.png)

Now, connect `m_axi_dram0` and `m_axi_dram1 ports` on Tensil block to `S_AXI_HP1_FPD` and `S_AXI_HP2_FPD` on Zynq block correspondingly. The TCU has two DRAM banks to enable their parallel operation by utilizing separate PS ports.

Next, click plus button in the Diagram toolbar and select AXI Direct Memory Access. The DMA is used to organize the feeding of the Tensil program to the TCU without keeping the PS ARM processor busy.

Double-click it. Disable Scatter Gather Engine, and Write Channel. Change Width of Buffer Length Register to be 26 bits. Select Memory Map Data Width and Stream Data Width to be 128 bits. Change Max Burst Size to 256.

![dma](/media/2022/tensil_ultra96v2/dma.png)

Again, click plus button in the Diagram toolbar and select AXI4-Stream Data Width Converter. A width converter is necessary because the Tensil architecture variants may differ instruction sizes depending on memory sizes and strides. Instruction operands contain base addresses and sizes.

Double-click it and change Master Interface TDATA Width to 9 bytes as printed in RTL SUMMARY when running the Tensil RTL tool.

![width_converter](/media/2022/tensil_ultra96v2/width_converter.png)

Connect `instruction` port on Tensil block to `M_AXIS` on AXI4-Stream Data Width Converter. Then connect `S_AXIS` on the AXI4-Stream Data Width Converter block to `M_AXIS_MM2S` on the AXI DMA block. Finally, connect `M_AXI_MM2S` on the AXI DMA block on `S_AXI_HP1_FPD` on Zynq.

Once again, click plus button in the Diagram toolbar and select AXI SmartConnect. The SmartConnect is necessary to expose DMA control registers to the PS ARM, which will enable software to control the DMA. Double-click it and choose 1 for Number of Slave and Master Interfaces. 

![smartconnect](/media/2022/tensil_ultra96v2/smartconnect.png)

Connect `M00_AXI` on the AXI SmartConnect block to `S_AXI_LITE` on the AXI DMA block. Connect `S00_AXI` on the AXI SmartConnect to `M_AXI_HPM0_FPD` on the Zynq block.

Finally, click Run Connection Automation and check All Automation. By doing this, we connect all the clocks and resets. Click the Regenerate Layout button in the Diagram toolbar to make the diagram look nice.

![design_final](/media/2022/tensil_ultra96v2/design_final.png)

Next, switch to the Address Editor tab. Click the Assign All button in the toolbar. By doing this, we assign address spaces to various AXI interfaces. For example, `m_axi_dram0` and `m_axi_dram1` gain access to the entire address space on the Ultra96 board, including DDR memory and control register spaces. (We only need access to DDR, so you can exclude register address space if you'd like.)

![design_address](/media/2022/tensil_ultra96v2/design_address.png)

Back in the Diagram tab, click the Validate Design (or F6) button. You should see the message informing you of successful validation! You can now close the Block Design by clicking x in the right upper corner.

The final step is to create the HDL wrapper for our design, which will tie everything together and enable synthesis and implementation—right-click `tensil_ultra96v2` item in the Sources tab and choose Create HDL Wrapper. Keep Let Vivado manage wrapper and auto-update selected. Wait for the Sources tree to be fully updated and right-click on `tensil_ultra96v2_wrapper`. Choose Set as Top.

Now it is time to let Vivado perform synthesis and implementation and write the resulting bitstream. In the Flow Navigator sidebar, click on Generate Bitstream and OK. Now Vivado will take time to work on our Tensil design. When done, you can observe some vital stats in the Project Summary. First is utilization, which shows what percentage of different FPGA resources our design is using. Note how we pushed BRAM and DSP resources to high utilization.

![utilization](/media/2022/tensil_ultra96v2/utilization.png)

The second is timing, confirming that our design meets signal propagation constraints at the clock speed specified for programmed logic (PL). The Worst Negative Slack being a positive number is good news, which means our design meets propagation constraints for all nets!

![timing](/media/2022/tensil_ultra96v2/timing.png)

## Model

The second branch of Tensil toolchain flow is to compile ML model to Tensil program consisting of TCU instructions, which are executed by the TCU hardware directly. For this tutorial, we will use ResNet20 trained on the CIFAR dataset. The model is conveniently placed on Tensil docker image at `/demo/models/resnet20v2_cifar.onnx`. We will be using the ONNX model. The Tensil compiler also supports TensorFlow, and you can try compiling the same model as the frozen TensorFlow graph at `/demo/models/resnet20v2_cifar.pb`. From within the Tensil docker container, run the following command.

```bash
tensil compile -a /demo/arch/ultra96v2.tarch -m /demo/models/resnet20v2_cifar.onnx -o "Identity:0" -s true
```

Or in case you decide to use TensorFlow model.

```bash
tensil compile -a /demo/arch/ultra96v2.tarch -m /demo/models/resnet20v2_cifar.pb -o "Identity" -s true
```

The result will be files listed in the ARTIFACTS table. The manifest (`tmodel`) is a JSON description of the compiled model. The Tensil program (`tprog`) and weights data (`tdata`) are the other two files. The Tensil compiler also prints COMPILER SUMMARY table with interesting stats for both the TCU architecture and the model.

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

## PYNQ

Now it's time to put everything together on our development board. For this, we first need to set up the PYNQ environment. This process starts with downloading the [SD card image for our development board](http://www.pynq.io/board.html). There's the [detailed instruction](https://ultra96-pynq.readthedocs.io/en/latest/getting_started.html) for setting board connectivity on the PYNQ documentation website. You should be able to open Jupyter notebooks and run some examples.

There is one caveat that needs addressing once PYNQ is installed. On the default PYNQ image, the setting for Linux kernel [CMA (Contiguous Memory Allocator)](https://elinux.org/images/2/23/LinuxCMA-cewg43.pdf) area size is 128MB. The CMA is used for all host-side memory with which the TCU will interact. This memory includes the instruction buffer, DRAM0, and DRAM1 pools. Given our Tensil architecture, the default CMA size is too small. To address this, you'll need to download our patched kernel, copy it to `/boot`, and reboot your board. Note that the patched kernel is built for PYNQ 2.7 and will not work with other versions.

```bash
wget https://s3.us-west-1.amazonaws.com/downloads.tensil.ai/pynq/2.7/image.ub
scp image.ub xilinx@192.168.3.1:
ssh xilinx@192.168.3.1
sudo cp image.ub /boot/
rm image.ub
logout
```

Now that PYNQ is up and running, the next step is to `scp` Tensil PYNQ driver. Start by cloning the [Tensil GitHub repository](https://github.com/tensil-ai/tensil) and then copy `drivers/tcu_pynq` to `/home/xilinx/tcu_pynq` on your board.

```bash
git clone git@github.com:tensil-ai/tensil.git
scp -r tensil/drivers/tcu_pynq xilinx@192.168.3.1:
```

We also need to `scp` the bitstream and compiler artifacts.

The bitstream contains the FPGA configuration resulting from Vivado synthesis and implementation. PYNQ also needs a hardware handoff file that describes FPGA components accessible to the host, such as DMA. Place both files in `/home/xilinx` on the development board. Assuming you are in the Vivado project directory, run the following commands to copy files over.

```bash
scp tensil-ultra96v2.runs/impl_1/tensil_ultra96v2_wrapper.bit xilinx@192.168.3.1:tensil_ultra96v2.bit
scp tensil-ultra96v2.gen/sources_1/bd/tensil_ultra96v2/hw_handoff/tensil_ultra96v2.hwh xilinx@192.168.3.1:
```

Note that we renamed bitstream to match the hardware handoff file name.

Now copy `tmodel`, `tprog` and `tdata` aftifacts produced by the compiler in `/home/xilinx`.

```bash
scp resnet20v2_cifar_onnx_ultra96v2.t* xilinx@192.168.3.1:
```

One more thing necessary to run ResNet model is CIFAR dataset. You can get it at [Kaggle](https://www.kaggle.com/janzenliu/cifar-10-batches-py). Put these files in `/home/xilinx/cifar-10-batches-py/` on your development board.

We are finally ready to fire up Jupyter notebook and run ResNet model on TCU.

## Jupyter

First, we import Tensil PYNQ driver and other required utilities.

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

Next, let's load CIFAR images from `test_batch`.

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
    array_width = 16
    img = np.transpose(data_norm[n].reshape((3, 32, 32)), axes=[1, 2, 0])
    img = np.pad(img, [(0, 0), (0, 0), (0, array_width - 3)], 'constant', constant_values=0)
    return img.reshape((-1, array_width))

def get_label(labels, label_names, n):
    label_idx = labels[n]
    name = label_names[label_idx]
    return (label_idx, name)
```

And extract one of the images we will be using to test the model.

```python
n = 9
img = get_img(data, n)
label_idx, label = get_label(labels, label_names, n)
show_img(data, n)
```

You should see the image.

![frog](/media/2022/tensil_ultra96v2/frog.png)

Then, specify the location of the bitstream and hardware handoff file.

```python
bitstream = '/home/xilinx/tensil_ultra96v2.bit'
```

Now, initialize PYNQ overlay from the bitstream and instantiate Tensil driver based on TCU architecture and DMA configuration. Note that we are passing `axi_dma_0` object from the overlay, which name matches with the DMA block in the Vivado design.

```python
overlay = Overlay(bitstream)
tcu = Driver(ultra96, overlay.axi_dma_0)
```

The Tensil PYNQ driver includes the Ultra96 architecture definition. Following is the excerpt from `architecture.py`.

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

Next, load `tmodel` manifest into the driver. The manifest contains references to the other two files (program and weights data).

```python
resnet = '/home/xilinx/resnet20v2_cifar_onnx_ultra96v2.tmodel'
tcu.load_model(resnet)
```

Finally, run the model and print ResNet classification results converted to the CIFAR labels. Note that if you are using ONNX model, the input and output are named `x:0` and `Identity:0` correspondingly. For TensorFlow model they are named `x` and `Identity`

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

Here is the expected printout.

```
Ran inference in 0.03043s

Output activations:
[-13.59375    -12.25        -7.90625     -6.21484375  -8.25
 -12.24609375  15.0390625  -15.10546875 -10.71875     -9.1796875 ]

Result: frog (idx = 6)
Actual: frog (idx = 6)
```
