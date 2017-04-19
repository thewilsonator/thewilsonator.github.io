So you want to write an LLVM backend Part 1: Important Files
===========================================================

So you want to write an LLVM backend. You take a look at the source tree and go "Where do I start?".
This is a guide based off of my own experience of trying to get Khrono's SPIRV up to date and in 
shape to be accepted upstream and make some of my own adjustments (namely using real intrinsics rather than 
"Itanium with weird extension" mangled C++ and the ability to more easily add support for GLSL 
for [dcompute](https://github.com/libmir/dcompute)).

You want to start by cloning LLVM and placing is somewhere (I'm going to call this $ROOT), and 
all the modifications will take place within $ROOT.

Firstly we need to create a directory under $ROOT/lib/Target/ called $ARCH.
Now that we have a place for you code to live we need to make LLVM's bukld system aware of it. LLVM uses a 
combination of CMake (bog standard for a large C++ project) and llvmbuild.

In $ARCH  add a `CMakeLists.txt`, and add `set(LLVM_TARGET_DEFINITIONS $ARCH.td)`. 
Next add any subdirectories `TargetInfo` and `MCTargetDesc` under $ARCH and to the `CMakeLists.txt`
Then add `add_llvm_target(${ARCH}Codegen` followed by the list of sources (dont forget to close the `)`.

Next we set up the `LLVMBuild.txt`:

add
```
[common]
subdirectories = MCTargetDesc TargetInfo

[component_0] 
type = TargetGroup
name = $ARCH
parent = Target

[component_1]
type = Library
name = ${ARCH}Codegen
parent = $ARCH
add_to_library_groups = $ARCH
required_libraries = Core ; plus anything else
```

We also need to inform $ROOT/libTarget/ about us. Under it's `[common] subdirectories add $ARCH`
and finally under $ROOT/CMakeLists.txt find `set(LLVM_ALL_TARGETS` add $ARCH to that list.

Great now we're all set to start writing code.

A note to reduce turnaround time: when building set the cmake variable `LLVM_TARGETS_TO_BUILD` to `$ARCH` or
`$ARCH;X86` (assuming you're on an x86 machine) so you don't wate time building other archs. 
