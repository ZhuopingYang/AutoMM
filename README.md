# AutoMM

## Overview
In this repo, we use general-purpose Matrix-Matrix Multiplication (GEMM) applications as an example and provide a detailed description of how to build a system-level design on AMD Versal VCK190 Platform. By going through this repo, users can get knowledge on:

+ How to design a high efficient single AIE kernel by leveraging the 7-way very long instruction words (VLIW)?
+ How to sustain 400 AIEs with the limited I/O interfaces between AIE and PL by using broadcast-packet mechanism?
+ How to transfer data from PL/AIE to AIE/PL by using bubble-free pipeline strategy?

We provide an automatic code generation and compilation flow that users can build the system on Versal step by step by changing the configuration files.

## Dependencies 
To play with the CHARM MM Accelerators, following software and hardware dependencies are required:
+ AMD/Xilinx Vitis 2021.1
+ AMD/Xilinx XRT Library
+ AMD/Xilinx VCK190 Evaluation Kit
+ Linux System with "tar" installed

## Environment Setup
1. To quickly boost and run experiments on the board instead of building the platform and Linux from scratch, users can download the platform package (VCK190 Base 2021.1) and petalinux common image(Versal common image) from the following link:<br>

https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-platforms/2021-1.html<br>

2. VCK190 Base 2021.1: It contains the pre-built Versal extensible embedded platform. During compilation users need to specify the platofrm path in the following format.<br/> 
```
PLATFORM = ${PATH}/xilinx_vck190_base_202110_1/xilinx_vck190_base_202110_1.xpfm
```

3. Versal common image: It includes the petalinux system boot files and the cross compilation environment needed for ARM CPU. Users can install the petalinux by running ``./sdk.sh``. During compilation, users need to point the path to SYSROOT and EDGE_COMMON_SW.<br/>
```
SYSROOT = ${PATH}/sysroots/cortexa72-cortexa53-xilinx-linux
EDGE_COMMON_SW=${PATH}/xilinx-versal-common-v2021.1
```

4. Vitis and Cross-compilation Environment Setup<br/>
```
source /opt/tools/xilinx/Vitis/2021.1/settings64.sh
source /opt/xilinx/xrt/setup.sh
unset LD_LIBRARY_PATH (If needed)
source ${PATH}/environment-setup-cortexa72-cortexa53-xilinx-linux
```

5. Project Setup and Compilation
Users can generate the customized project by setting up the configuration file and directly running the following command:
```
./project_setup.sh ./config_files/input.cfg ${Project_DIR}
cd ${Project_DIR}
make all EDGE_COMMON_SW_PATH=${PATH} SYSROOT_PATH={PATH}
```
5. On Board Execution for MM with Arbitrary Sizes
After copy the sd card image to micro sd card and boot up the system run the following commands to get the execution results. {M}, {K}, {N} refers to the size of MM. In order to reduce the effect of overhead of calling API when runnning the kernel, users can specify the number of {iteration} of running the MM then it provides the average throughput. To verify the correctness of the MM kernel, {verify} should be assigned to 1, otherwise 0. One example of running MM with 1024\*1024*\1024 for 100 iterations without verify the result can be: **./hostexe mm_hw.xclbin 1024 1024 1024 100 0**
```
cd /mnt/sd-mmcblk0p1
./hostexe mm_hw.xclbin {M} {K} {N} {iteration} {verify}
```


## Step by Step Tutorial
In this part, we first introduce the overall MM tiling strategy including four levels of tilings. Then in the later parts, we illustrate the methodology of how we handle each of these level of tilings.<br>
### Overall MM Tiling Strategy:<br>
Given a large Matrix Multiplication(MM) with size (M\*K) * (K\*N) refer as M\*K\*N, the listing bellow shows four level of tilings to handle this MM (from innermost to outermost):<br>
+ Line 16-20: MM calculated on a **single AIE core**. 
+ Line 12-14: The spatial distribution unrolled across different AIE cores in **AIE Array**.
+ Line 7-9: The sequential processing of data stored in **PL on-chip memories**. 
+ Line 2-4: The temporal processing of data stored in off-chip memory.<br>

We visualize the on-chip buffer level tiling in the right figure. We refer the MM calculated in single AIE as  **"Tile"**  level and refer the MM unrolled on AIE array level as  **"Batch"**  level. The strtegy of mapping the tiled MM on AIE array will be illustrated later. 

<img src="https://user-images.githubusercontent.com/77606152/198852392-beb5d876-56c7-4486-8b14-f3ea0021ab72.png" width="400" height="300"> <img src="https://user-images.githubusercontent.com/77606152/198853940-ebdd1006-4807-42eb-9595-ce4f6fe2cc18.png" width="500" height="300">

### Single AIE Programming:<br>
In this part, we demonstrate the coding style of calculating MM with size **TI\*TK\*TJ** in a single AIE which corresponds to the **first level of tiling**.<br>

AIE is a very-long instruction word (VLIW) processor which can issue upto seven operations in parallel using one VLIW word:<br>
+ **Two Load** Operations: Load upto (the width of data could change) two 256bits data from AIE local memory to AIE register.<br>
+ **One Store** Operation: Store upto one 256bits data from AIE register to AIE local memory.<br>
+ **One Vector** Operation : Calculating a vector matrix multiplt or add.<br>
+ **One Scalar** Operation<br>
+ **Two Move** Operations: Two move operations that move the data from scalar/vector to scalar/vector.<br> $~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~$

<img src="https://user-images.githubusercontent.com/77606152/198855601-6c7e307b-d78a-4c98-bd05-b3b6eb3f8902.png" width="600" height="400"><br>

**The key challenge of programming single AIE is how to make back-to-back issued instructions by utilizing the 32KB local memory and 2KB local registers of a single AIE (for integer data type there are additional 3KB accumulator registers).**<br>

We provide our source code that achieves 95% efficiency when calculating 32\*32\*32 MM in src/aie/mm_kernel0.cc. The visualization of the algorithm is shown below:<br>

**The insights of programming AIE are:**
+ **Mannually unroll the innermost loop to make it fully pipelined**: View the report under "Work/aie/xx_xx/xx_xx.log" (here xx_xx is the position of AIE) after compilation and make sure 1) "Critical cycle of length" equals the number of vector operations. 2) The cycle after folding achieves the cycle due to"Critical cycle of length". The following figure is reported from a cycle accurate simulator (Vitis Analyzer) provided by AMD. In our innermost loop (line 43), we mannually unrolled 16 vector mac operations and after checking we fully pipelined them.<br>
<img src="https://user-images.githubusercontent.com/77606152/198856298-b6f5b92b-9a5a-4eb1-8e43-bca0ce992e69.png" width="1000" height="200"><br>
+ **Avoid of using judgement statement in the inner most loop**: The judgement statement will prevent the compiler to acheive fully pipelined mac instructions. As can be seen in line 98, since we need to store the data from register back to the local memory when finishing the last iteraion of reduction dimension (K loop), to avoid of using the judgement in the innermost loop we mannually write the last iteration of the innermost loop out of the "for loop" region. <br>
+ **Using __restrict and chess_prepare_for_pipelining pragma to improve the efficiency**: [Software pipelining](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2022_1/ug1079-ai-engine-kernel-coding.pdf)**<br>

## Automatic Code Generation (ACG):<br>
The tools for automatically generate the source code is under ""src_gen"" folder

**ACG** takes platform information and user-specified design point as input, and automatically generated the systen-level design by launching the following 4 template based components sequentially:<br>

**Kerne_lGen:** Kernel_Gen is launched to generate both the single AI Engine(AIE) C code and adaptive data flow (ADF) graph code in C++ for verfying the correctness of single kernel design. MM kernels with fp32 data type in different shape that can be fit in single kernel are supported in current version.<br>

**AIE_ArrGen:** AIE_ArrGen is launched to generate new ADF graph code that defines how packet-switch streams are connected to AIE array which contains 400 AIEs. Single kernel calculating 32x32x32 MM with fp32 data type is supported to scale out to the AIE array. <br>

**PL_Gen:** Based on the AIE array created by AIE_ArrGen, PL_Gen is launched to generate PL streams, scheduling controller C/C++ HLS modules to communicate with AIE array and PL on-chip buffers, off-chip AXI data transfer modules to communicate with DDR. Differnet system level designs varying in on-chip buffer size and its implementation option (BRAM or URAM) fp32 data type are supported.<br>

**Host_Gen:** Host_Gen is launched to generate the system control logic running on the ARM CPU by using AMD XRT APIs.<br>

**Compilation**
After code generation, the vendor tools AIE compiler and V++ compiler take ADF gragh and HLS C/C++ as input respectively. Their output object file libadf.a and kernel.xo will be linked into xclbin file which includes the hardware information of the design for the target platform. C++ compiler compiles XRT API based host code to executable file runs on CPU.<br>

### Configuration File 
We provide a configuration file template under "./config_files/input.cfg", users can specify platform, data type, kernel type and mapping strategy of each level in this file. The feasible option of each parameter are illustrated in **( )** The rules of using this configuration file are listed below:
- **Platform** refers to the hardware platform used in the project. VCK5000 and VCK190 are supported in the current framework.
- **DATA_TYPE** the framework currently support fp32, int32 and int16 data types.
- **KernelGen, AIEArrGen, SysGen** decide if the corresponding ACG should be launched (1 refers to launch). 
- **KRL_TYPE** refers two types of MM kernels provided in our source file.
- **I, K, J** refers to the MM size stored and calculated in a single AIE.
- **A, B, C** refers to the BATCH level parameter.
- **X, Y, Z** refers to the on-chip level parameter.
- **LHS_BUFF, RHS_BUFF, OUT_BUFF** dicide the implmentation option for LHS, RHS and output buffers. 1 refers to URAM and 0 refers to BRAM. For example, LHS_BUFF=1 means LHS buffer is implemented by URAM.
```sh
Platform:VCK190;
DATA_TYPE:fp32;
KernelGen:1;
	KRL_TYPE:0;
	I:32;
	K:32;
	J:32;
AIEArrGen:1;
	NUM_PACK:4;
	A:6;
	B:4;
	C:16;
	A_BRO:4;
	C_BRO:3;
SysGen:1;
	X:8;
	Y:1;
	Z:2;
	LHS_BUFF:0;
	RHS_BUFF:0;
	OUT_BUFF:1;
```


**References:**<br>
[1] **[AIE Architecture](https://docs.xilinx.com/r/en-US/am009-versal-ai-engine)**<br>
[2] **[AIE Instructions and APIs](https://www.xilinx.com/htmldocs/aiengine_intrinsics_start.html)**<br>
[3] **[AIE Coding Example](https://www.xilinx.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2022_1/ug1079-ai-engine-kernel-coding.pdf)**<br>
[4] **[Introduction to FP32 programming of AIE](https://github.com/Xilinx/Vitis-Tutorials/blob/2022.1/AI_Engine_Development/Feature_Tutorials/07-AI-Engine-Floating-Point/README.md)**<br>
