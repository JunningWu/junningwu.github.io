---
layout: post
title: Simulating RISC-V Program using Gem5
date: 2019-11-25
categories: 转载
tags: [转载,技术文章]
description: This guide is intended to give an extended explanation of how to use the gem5 architectural simulator in syscall emulation mode (i.e. without an operating system). It covers the basics of getting a real system ready and building gem5 from scratch, running gem5 with custom simulation settings to get the simulated system just right and run your own programs, as well as how to modify basic ISA components allowing modified or custom instructions.
---
## Intro
This guide is intended to give an extended explanation of how to use the gem5 architectural simulator in syscall emulation mode (i.e. without an operating system). It covers the basics of getting a real system ready and building gem5 from scratch, running gem5 with custom simulation settings to get the simulated system just right and run your own programs, as well as how to modify basic ISA components allowing modified or custom instructions.

I. Building

gem5 requires the following dependencies:

git (you needed this to clone the repository)

g++ (4.8+)
OR
clang (3.1+)

python (2.7) and the corresponding libpython bindings (python-dev)

scons

zlib

m4

gem5 has the following optional dependencies:

protobuf (2.1+) – used for trace capture and also provides tcmalloc() functionality, which can increase simulation performance up to 12%. Highly recommended if you plan to run larger/longer simulations.

On Unbuntu, these required and optional dependencies can be installed using the following apt command:

apt install git build-essential python-dev python scons zlib1g zlib1g-dev m4 libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev

You can then clone the gem5 repository with the following command:

git clone https://gem5.googlesource.com/public/gem5

cd into the directory and from there you can run the build commands.

To build gem5, run scons from the gem5 root directory passing in the build location of the ISA and binary type you wish to build.

Active, supported ISAs include:

ARM, RISCV, SPARC, X86

gem5 can also run ALPHA, MIPS, and POWER ISAs but they are no longer maintained and thus should be used with caution and extreme testing.

Binary types refer to the level of debug and optimization information in the compiled binary. gem5 supports 5 binary types:

debug – no optimizations and all debug symbols. Resulting binary is very slow but enables use of the debugger. Good for when you want to debug new changes you’ve made but not ideal for running longer simulations.

opt – most optimizations and all debug symbols. Results in a much faster binary than the debug type but also allows for a degree of debugging and issue tracing.

fast – all optimizations and no debug symbols. Fastest possible binary and smallest size but provides no debug information if issues occur. Good for running unmodified, vanilla gem5 runs where no issues are likely to occur or when you have the utmost faith in the changes you’ve made to the source.

prof – contains profiling information to be used with gprof (GNU profiler).

perf – contains profiling information to be used with gperftools (Google performance tools).

If you’re unsure which binary type to use, opt is usually a safe bet that mixes many of the benefits of debug and fast.

Once you’ve selected an ISA and binary type, you can build gem5 from within the gem5 root directory using scons:

scons build/isa_name/gem5.bin_type -jX

Where:

isa_name = the name of the ISA you’ve chosen, all caps, no hyphen in RISCV.

bin_type = the binary type you’ve chosen. This will be the extension of the compiled gem5 file, allowing you to have multiple binaries of different types compiled at one time for a single ISA.

X = how many threads to run the build with. Generally, you want to set this	to <number_of_cores>+1 to take advantage of your entire system and also offset I/O operations.

So to build gem5 for RISC-V using an opt binary type on a system with 10 cores, you would run:

scons build/RISCV/gem5.opt -j11

The first compile will take some time but afterwards, you can recompile by running the same function and scons will incrementally rebuild all necessary files much quicker than the initial build depending on how much you modified.

By default, the binary is built with the CPU models listed in the build_ops folder in the file for the ISA you select. If the CPU model isn’t listed there, you won’t be able to use it with the binary you compiled later on. If you want to add a different CPU model, you can modify the file in the build_opts directory or pass in a new CPU_MODELS variable like so:

scons build/RISCV/gem5.opt -j11 CPU_MODELS=O3CPU,AtomicSimpleCPU

Keep in mind that this will overwrite the CPU_MODELS value from the build_opts directory, so add all cpu models to the scons call that you want to use with this binary later.


## II. Running

To run the compiled gem5 binary, you must pass it a python configuration file that defines the features of the system you want to simulate as well as the program you want the system to run.

While these configuration files can be custom made, and you may end up doing so if you are going to use gem5 simulations for a great number of tests or for testing very application specific situations, there is a convenient, general purpose file located in configs/example called se.py. This contains a very generalized model that connects to a backbone of other files containing command-line flags you can use to customize the system.

You can see the entire list of available options by looking at the contents of the file configs/common/Options.py, but some important ones to note:

--num-cpus <x> – The number of CPUS running in the simulation. Make sure the ISA you selected has support for multi CPU simulations before changing this.
Defaults to 1.

--sys-clock <x> – Master clock speed for the simulated system. You can use units like ‘GHz’ and ‘MHz’ when specifying a speed.
Defaults to 1GHz.

--mem-type <x> – Specify the type of memory to be used in the simulation.
The possible memory types include:

SimpleMemory

DRAMCtrl

DDR3_1600_8x8

HMC_2500_1x32

DDR3_2133_8x8

DDR4_2400_16x4

DDR4_2400_8x8

DDR4_2400_4x16

LPDDR2_S4_1066_1x32

WideIO_200_1x128

LPDDR3_1600_1x32

GDDR5_4000_2x32

HBM_1000_4H_1x128

HBM_1000_4H_1x64

QoSMemSinkCtrl

Each type specifies information about the memory timings, the size, and the power usage of the devices in the memory. The details on those specifications can be found in the src/mem/DRAMCtrl.py file for all the memory types except SimpleMemory and QoSMemSinkCtrl which can be found in src/mem/SimpleMemory.py and src/mem/qos/QoSMemSinkCtrl.py respectively.
Defaults to DDR3_1600_8x8.

--mem-channels <x> – The number of memory channels to be used during the simulation.
Defaults to 1.

--mem-size <x> – Size of the memory in simulation. You can use units like GB and MB when specifying a size. You might also get a warning when running a simulation about mismatched memory size and address space if the memory type you select is larger than the memory size you specify here. This is usually only an issue if you need to use more memory for the simulation program than is specified here and can be fixed by changing this option to the size the warning specifies.
Defaults to 512MB.

--caches – Enables l1 CPU caches in the simulation. Some ISAs do not support caching on all CPU types while others require it.
Defaults to not enabled.

--l2cache – Enables l2 CPU caches in the simulation.
Defaults to not enabled.

--l1d_size <x> – Sets the size of the l1d cache used in simulation. You can use units like kB when specifying a size. Requires that caching be enabled with the --caches flag to take effect.
Defaults to 64kB.

--l1i_size <x> – Sets the size of the l1i cache used in simulation. You can use units like kB when specifying a size. Requires that caching be enabled with the --caches flag to take effect.
Defaults to 32kB.

--l2_size <x> – Sets the size of the l2 cahce used in simulation. You can use units like MB and kB when specifying a size. Requires that l2 caching be enabled with the --l2cache flag to take effect.
Defaults to 2MB.

--list-cpu-types – Displays a list of all the CPU types that the specified gem5 binary is compiled to use.

--cpu-type <x> – Sets the CPU type that the simulation will use. The possible choices for this flag depend on the ISA this gem5 binary is for or which CPU types you passed into the CPU_MODELS argument when building the binary.
Defaults to AtomicSimpleCPU.

--cpu-clock <x> – Clock speed of the CPU in simulation. Different from the system clock in that this effects only CPU-based timings. You can use units like GHz and MHz when specifying a speed.
Defaults to 2GHz.

These options are used to set information about what you want the system to do during the simulation:

--cmd <x> – The binary that the simulation CPU will run. If you are using an ISA different then the native system you are running gem5 on, you will need to cross-compile whatever code it is you want to run. While some system libraries are offered from within gem5, it is always better to compile with embedded libraries if the compiler you are using supports that functionality.

--options "<x>" – The options to pass into the binary when it runs on the simulation CPU. Make sure to enclose all of the options in a pair of double-quotes.

--input <x> – Specify a file to read stdin from. This allows you to provide input to programs that usually require user interaction and input to proceed since you won’t be able to provide that input yourself while the simulation is running.

--output <x> – Specify a file to save the stdout produced by the binary running on the simulation CPU. This will help separate the program’s output from gem5’s output and make it easier to diagnose issues if something goes wrong during the simulation.

--errout <x> – Specify a file to save the stderr produced by the binary running on the simulation CPU. As with the --output flag, this is useful for separating program and simulation output and helps with debugging issues should they arise.

There are many other options than the ones listed here that allow for much finer-grained control of the simulation settings and also allow for setting up many other features like checkpoints and peripherals.

Here is an example of a command-line run of gem5:

./build/RISCV/gem5.opt configs/example/se.py --cpu-type=DerivO3CPU --caches --mem-type=DDR4_2400_8x8 --mem-size=8GB --cmd=test_prog --options="20" --output=sim.out --errout=sim.err

Breaking this down:

./build/RISCV/gem5.opt

Run the gem5.opt simulator compiled for the RISC-V ISA.

configs/example/se.py

Use the standard syscall emulation (SE) config file with the available options listed previously.

--cpu-type=DerivO3CPU

Specify the simulation CPU model to be DerivO3CPU, a version of the Out-Of-Order CPU.

--caches

Enable l1 caches. No further options about caching means that the default cache settings are used.

--mem-type=DDR4_2400_8x8

Specifies the type of memory to be used, selected from src/mem/DRAMCtrl.py

--mem-size=8GB

Sets the size of memory for the program running in the simulation to 8GB. This is not a limit on the size of memory that gem5 will use, only a specification for the program running in the simulator.

--cmd=test_prog

Run the binary called test_prog in the simulation.

--options="20"

Pass the argument 20 into the binary. This means the execution of the binary would look something like this on the command line if run normally:

./test_prog 20

--output=sim.out

Save stdout from the binary test_prog to a file named sim.out.

--errout=sim.err

Save stderr from the binary test_prog to a file named sim.err.

It can also be helpful to tee the output of gem5 to a file so you have both a written record of the simulator’s in-progress output and any errors that occur while also still being able to keep track of how things are going on-screen:

./build/RISCV/gem5.opt configs/example/se.py --cpu-type=DerivO3CPU --caches --mem-type=DDR4_2400_8x8 --mem-size=8GB --cmd=test_prog --options="20" --output=sim.out --errout=sim.err | tee run.out

Because gem5 simulations can sometimes take a very long time (depending on the type of CPU model you use as well as the complexity of the binary the simulation is running), it is also recommended to run gem5 inside of a screen or tmux session, that way a severed SSH connection or crashed tty/terminal doesn’t result in a loss of simulation data.


## III. Modifying

The source code for gem5 (not including its included dependencies) is located in the src directory of the root gem5 folder. Most of the source code directories within are ISA agnostic in that every gem5 binary you build uses this source and will be affected if you modify it. If you want to edit the individual ISAs, you can do so in the src/arch directory.

There are many reasons you might want to edit an ISA. You might want to change the way the gem5 simulator runs a certain instruction or add an instruction it currently does not support. You may want to extend the functionality of an already-existent instruction or create an entirely new instruction from scratch for a certain architecture. Or maybe you want to make an entirely new, experimental ISA.

Regardless of the reasoning, the main way you make changes to the ISA is through the files in the isa folder in each of the architecture directories. For example, the main ISA files for RISC-V are in src/arch/riscv/isa. After this, locations will vary because each architecture organizes the ISA definitions differently, but each contain a core set of files for describing instructions.

bitfields.isa – contains the definitions for the instruction bitfields used by this ISA. This file defines all the easy-to-use names for different components of the incoming binary instruction by certain bit sequences in the instruction to names you can use in other ISA files. For example:

def bitfield RD <11:7>

allows you to use the name RD in other ISA files, like the decoder.isa file that will be discussed later, to easily reference the value of instruction bits 11:7. These bitfields are often made for easy access to register source and destination sections, immediate sections, opcode sections, and functional decision sections within the instructions for the architecture being used.
You can add your own in the following format:

def bitfield *NAME* <*HI_BIT*:*LO_BIT*>

*NAME* – Name of the bitfield that you will use in other isa files.

*HI_BIT* – High bit value of the bit range you want this bitfield to access.

*LO_BIT* – Low bit value of the bit range you want this bitfield to access.

operands.isa – defines the operands that will be used by instructions and have direct access to registers of varying types. Values defined here can be used in the instruction definitions within decoder.isa so that the system knows it is accessing a register value and not just looking and the raw value of the bitfield declared in bitfields.isa. This is also where you can specify all of the different types than can be stores in a given register. Most of the operand datatypes that a given ISA supports are already defined so modifying this section is only important if you wish to expand the types that the ISA can handle. To define an operand datatype, add a new entry to the def operand_types block with the following format:

'*NAME*' : '*C_TYPE*'

*NAME* – Name that will be used to reference the data type in the decoder ISA files.

*C_TYPE* – The C/C++ type that will be used to represent whatever operand this type is used on.

To define an operand, add a new entry to the `def operands` block with the following format:

'*NAME*': ('*OPERAND_TYPE*', '*DATA_TYPE*', '*BITFIELD*', '*FLAGS*', *SORT_INDEX*)

*NAME* – Name that will be used to reference the operand in the decoder ISA files.

*OPERAND_TYPE* – Typedef representation of what this operand will be stored as in the system. For registers, these typedefs are defined in the registers.hh file for the ISA in question. This only defines how the data will be stored in the simulation’s representation of the ISA, the actual type used when accessing the operand in the decoder ISA files will depend on the *DATA_TYPE* selected or the suffix added to when using the operand.

*DATA_TYPE* – The default type that the operand will be represented as when used in the decoder ISA files without an addition type modifier.

*BITFIELD* – The bitfield used to get the index for the operand. This will automatically alias components of incoming instructions to the correct locations in the system based on the bitfields selected, such as which destination and source registers are being used, while simultaneously making the decoder ISA files much easier to work with because register indexing is automatically handled. The possible bitfields are defined in the bitfields.isa file.

*FLAGS* – Specify the flags used to help type and classify operands. Flags are used to help classify instructions, keep instructions and operands paired together correctly, and take data on usage of different components of the ISA during simulation. The possible flags are stored in src/cpu/StaticInstFlags.py. If you want to add more than one flag, you can enclose this section in parenthesis and separate each single-quoted flag with a comma.

*SORT_INDEX* – This index doesn’t change much regarding the resulting binary, but in case you need to debug it’s good when adding entirely new operands to select an index that is not already in use. Otherwise, the index can generally be left alone.

decoder.isa – contains the definitions of the individual instructions, including how their opcode breaks down and what values are saved and loaded when the instruction runs. This is the main file to edit for modifying or adding instructions. It uses a series of switch-case-like statements and relative register accesses to generalize operations so they work with varying inputs. The file uses decode blocks to break instructions apart using a format like the following:

decode *BITFIELD* {

0x00: *OPERATION*

0x01: *OPERATION*

0x02: *OPERATION*

...

default: *OPERATION*

}

Where:

*BITFIELD* – The bitfield defined in bitfields.isa that will be used in the switch-case-like statement for incoming instruction decoding.

*OPERATION*s – The individual operations that are run whenever that particular branch of the switch-case-like decode is taken. This can be an instruction’s execution definition or the start of another decode statement to further break apart the bitfields. A default statement can optionally be used within the decode block to catch all unhanded cases just like a switch-case.

The chosen bitfield for a given decode block can be of any size and the case format can be using decimal, binary, or hex based on preference. Hex is often preferred though due to hex values often also being used in ISA definitions and documentation.

If creating this file from scratch, is it recommended to start with the most meaningful bitfield first, such as one that decides the type or major subject of the instruction, and use that as the first decode statement in the file, then define subsequent decode statements within in decreasing meaningfulness until the entire decode tree has been created and individual instructions can be defined.

In addition to decode blocks, format blocks are used to group instructions by type within a particular decode block. Instruction types provide certain registers and functionalities to the execution definitions for an instruction so they can be used in the code that defines how the instruction will behave within simulation. They also help to classify the instructions so better data can be collected during simulations. Each ISA has its own set of instruction types defined in a series of ISA format files found within the same isa directory as the other ISA files discussed so far. Make sure to check out the instruction types there if you intend to dramatically alter an instructions functionality or add an entirely new function to make sure you give the instruction the correct type so that it has all the resources it needs when you go to write the execution definition. You can also observe from other instructions in the decoder.isa file what resources different types of instructions provide by looking at what types many common instructions use within the ISA you have chosen. A format block is placed within a decode block and can contain some or all of the decode cases within that block. The format is:

format *INSTRUCTION_TYPE* {

0x0: *OPERATION*

0x1: *OPERATION*

}

Where:

*INSTRUCTION_TYPE* – The instruction type for each instruction contained within this format block. This will change what resources the instruction execution definitions will have available to them and how they are logged during simulations.

*OPERATION* – Individual operations from a decode block one layer above where the format block is defined. The format block does not do any decoding itself.

With a sufficient instruction type selected through formatting statements and enough detail added to decode to a specific instruction, the execution definition can be added to decoder.isa. These execution definitions vary widely and depend on the particular ISA in use, the instruction type being defined, and any other special characteristics of the simulation system in question, so take these outlines as a general overview. The execution definition of an instruction generally follows a format like this:

*MNEMONIC*({{

*C++_INSTRUCTION_FUNCTION*

}}, *FLAGS*);

Where:

*MNEMONIC* – The mnemonic for the instruction. This does not have to match the ISA’s actual mnemonic used for assembly programming and references, but keeping them consistent does make things easier when debugging and measuring more detailed information during simulations.

*C++_INSTRUCTION_FUNCTION* – A definition of what the instruction does using C++ programming. Register sources and destinations are pulled from by using the operands and operand types (discussed later), assuming the instruction type is correct. Ideally, unless you are testing a specific execution method or want to perform the instruction’s operation in a specific way, it is best to write the most straightforward C++ code possible to achieve the goal and the compiler will optimize things during building while the simulator retains information about execution duration and taxes on the system due to resource usage.

*FLAGS* – Flags like the ones used in operands.isa to help classify instructions and measure usage of different flagged operations during simulation. You can add more than one flag by separating them with commas.

When writing the C++ definition for an instruction, registers and other resources are accessed by using the proper operand call in the definition. Operand calls are made by combining an operand and an operand type from the operands.isa. Select an operand based on the what you want to access, such as a destination register when you want to store a result or a source register when you want to read a value, and append an operand type (separated by an underscore) to change how the operand is viewed for that particular access. An example using the operands from the RISC-V ISA:

Rs1_sd

Will access the operand Rs1, a source register, and view it as the type specified by sd, which is a signed 64-bit integer. Changing sd to uw would cause the operand to be viewed as an unsigned 32-bit integer, even though the actual size of that operand in the system is still 64-bit. By selecting the right operand types, making instructions that work with smaller sizes than what the simulated system uses internally is much easier. Once the desired operand has been made, it can be used in the C++ definition just like any other variable, whether as a destination for an assignment of a value or the source of a value used in a particular operation. An example instruction RISC-V instruction:

add({{

Rd = Rs1_sd + Rs2_sd;

}});

This is the basic add instruction in the RISC-V architecture. The C++ definition uses the operands Rs1 and Rs2, viewed as signed 64-bit integers thanks to the sd operand type, and adds them. The result is stored in the operand Rd. Rd does not have a type specified, so the default defined in operands.isa is used. This is common for destination operands as they simply hold as much as possible for their internal size regardless of the size of the source operands, but this behavior can be changed by also giving the destination operands an operand type like the source operands.

Miscellaneous registers and system resources for a particular ISA are modified in the src/arch folder for the given ISA (one directory above the isa directory that the ISA files are in). The format of these files can change depending on the ISA, but most of the files have self-explanatory names like, for example in RISC-V, registers.hh is where miscellaneous registers and their characteristics are modified.


## IV. External Libraries

If you want to include an external library and use it in the instruction execution definitions or another part of the simulator, you can do so by adding the library to a folder in the gem5 root directory and creating a SConscript file to specify the libraries you want to add to gem5. While you can put the library in any folder, it is generally easier to put libraries in their own folders within the ext directory to keep external dependencies grouped together and make sure each one has its own SConscript file properly defined. A SConscript file is a python-like file that specifies which libraries to add, which headers to include, and where to look for extra dependencies during a build. All SConscript files within the gem5 root directory and below are read during a build and the instructions for inclusions are followed during builds. A basic SConscript file for adding an external library looks like this:

Import('main')

main.append(LIBPATH=[Dir('.')])

main.Append(LIBS=['*LIBRARY_NAME*'])

main.Append(CPPPATH=[Dir('.')])

Where:

*LIBRARY_NAME* – The name of the library in the folder that you want to add to the build. Note that this is adding the library to the command line flags used by the C++ compiler so it will follow C++ library naming conventions. That is, it expects a library with the prefix lib to be present in the folder, but you should leave that prefix off when putting the name in the SConscript file.

This particular example works for adding a library and it’s headers when they are located within the same folder. If they are in different folders, the Dir('.') statements can be independently changed to point to the correct folder for the library and headers respectively, but it is good for organization to try and keep the libraries and headers in close proximity at the very least. With a SConscript file created, the headers will automatically be relocated during builds, so including the headers in files is as easy as calling:

#include <lib_header.h>

Or if you’ve included a C library:
```
extern "C" {

#include <lib_header.h>

}
```
This can be added to any file you want to use the library in and scons will automatically link the header file during the build based on the headers it needs to find in the source files and the SConscript files provided. If you want to use the library in the instruction execution definitions of the ISA files, the easiest place to put the include statement so that all instructions have access to the libraries is in the isa.hh file of the ISA you are working on. This is much cleaner than adding the include directly to the decoder.isa file itself for each instruction that needs it and keeps all the libraries being used by the execution definitions in the same place for easy management.


## V. Using the SoftFloat Library

Adding the SoftFloat library to gem5 allows instruction execution definitions to take advantage of a series of functions that accurately replicate floating-point operations in software. This can be extremely useful for adding functionality to floating-point instructions that involve accessing some or all of the information generated during an operation. Without this library, many gem5 floating-point instructions simply use the host machine’s floating point instructions, making it impossible to extract intermediate information. However, the cost of using SoftFloat comes from its speed; using entirely software to mimic floating-point operations will cost more on the host system running the simulator than using native floating-point instructions. Thus, while the results of the simulation will be the same and the total time cost of a simulated instruction within the simulator is the same, the time it takes to run simulations will increase relative to the density of floating-point operations in the program that the simulator is running.

To add SoftFloat to gem5, the same process is followed as described in IV. External Libraries. The library file is compiled from source by obtaining the repository from http://www.jhauser.us/arithmetic/SoftFloat.html and running make in the build/Linux-x86_64-GCC directory, then the resulting softfloat.a and the header files can be moved to a directory within the `gem5/ext` folder. At this point, the softfloat.a library needs to be renamed so that it is prefixed with lib. This makes adding the scons flags much easier in the long run. Once the files have been moved, a SConscript file is made so scons will pick up the new library and use it during builds. For SoftFloat, the SConscript file looks like this:

Import('main')

main.append(LIBPATH=[Dir('.')])

main.Append(LIBS=['softfloat'])

main.Append(CPPPATH=[Dir('.')])

This file must be in the same directory as the libsoftfloat.a library file and the headers for this file to work. Then, subsequent gem5 builds will include the library as appropriate. The #include statement for the SoftFloat header goes in the isa.hh which makes is available to the instruction execution definitions in decoder.isa. Because SoftFloat is a C library, it must be included using an extern "C" statement like so:
```
extern "C" {

#include <softfloat.h>

}

The SoftFloat functions can be called normally in an instruction definition like this RISC-V double-precision floating-point instruction:

fmul_d({{

if (std::isnan(Fs1) || std::isnan(Fs2)) {

if (issignalingnan(Fs1) || issignalingnan(Fs2)) {

FFLAGS |= FloatInvalid;

}

Fd = numeric_limits::quiet_NaN();

} else {

float64_t fs1_sf;

float64_t fs2_sf;

float64_t fd_sf;

std::memcpy(&fs1_sf, &Fs1, sizeof Fs1);

std::memcpy(&fs2_sf, &Fs2, sizeof Fs2);

fd_sf = f64_mul(fs1_sf, fs2_sf);

std::memcpy(&Fd, &fd_sf, sizeof fd_sf);

}

}}, FloatMultOp);
```
This will use type-punning to cast the double operands to SoftFloat types, use the SoftFloat library call to perform the operation, and then type-pun the double result back to a value usable in the simulator. Before this modification, the entire else block of the code snippet shown above was simply one line:

Fd = Fs1*Fs2;

That makes it impossible to access intermediate values created during the operation. With SoftFloat, the library source can easily be modified to access intermediate values to extend the functionality of floating-point operations of all types.


## VI. Using SPEC Benchmarks

Even though many of the SPEC benchmarks require the use of operating system libraries, some can be run using newlib and other embedded system libraries making them perfect candidates for running on gem5 in syscall emulation mode. You will need to follow the instructions on “avoiding runspec/runcpu” in the SPEC documentation as you will not be able to use the runspec/runcpu tool directly in gem5, as well as allow cross-compiling if the ISA you will be simulating is not the one on which you are running gem5 (most likely x86).

Create a new cfg file in the config folder by copying an example cfg that best matches the system you intend to model. Edit the file and the CC, CXX, and FC compiler variables to point to the location of the compiler for the ISA you intend to use. If your compiler does not support one of those (some ISAs do not have well-supported Fortran compilers) then you will need to avoid benchmarks that make use of those compilers. Once set up, follow through the build process for the particular version of SPEC you are using, making sure when it has generated a Makefile.spec file to check the contents and to ensure all the proper locations for the compilers have been set. You may also need to add flags to the compiler through the flags variable in the Makefile.spec file if the ISA you are using requires additional settings, such as RISC-V which in some benchmarks requires flags that specify which architectural extensions are included. Finish building the benchmark and use the resulting binary and provided input arguments as the input --cmd and --options flags respectively in a gem5 run. Additional flags can be used to add stdin, stdout, and stderr as outlined previously in II. Running.

Remember that not every benchmark will work with embedded libraries and some will attempt to access operating system libraries, usually from Linux. Those benchmark will require specific tailoring for both the situation you’re in as well as the kind of results you want to get out of those simulations to get them working in gem5 using syscall emulation mode. Thus, most of those benchmarks are usually skipped in favor of the ones that support embedded libraries. Also note that some of the SPEC benchmarks will take a long time to run on gem5, especially if you are using some of the more expensive simulation features like multi-layer caching simulation and memory system modeling. With an Out-Of-Order CPU model, DDR3 memory model, and caching enabled, a RISC-V gem5 simulation that would normally take 75 seconds on hardware could take over 60 hours to simulate depending on the host hardware used. Under the same settings and features, a simulation that would take 25 minutes on hardware could take over a month to run. You can help alleviate some of this in the SPEC benchmarks by selecting the smallest test size when generating build files for particular benchmarks. Not all benchmarks have different sizes, but some support using smaller data sets for testing the benchmarks before running them full-scale for longer periods of time. The smaller test sizes can help make the gem5 simulation runs much more realistic at the expense of less test data, a trade-off often worth making when the simulation could take months to run otherwise.

https://vlsiarch.ecen.okstate.edu/gem5/