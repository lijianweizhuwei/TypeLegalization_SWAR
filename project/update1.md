# SIMD Legalization and SWAR

## Project Flowcharts
![Alt text](../image//flowchart.png)

The picture above is the flowchart of our updated model. In our project, firstly need analyss the input instruction set, then determine a appropriate processing method according to the pre-processed cost table. Finally, output the comparsion result.

We devide our project into three main part. 

## Type Legalization

### Brief description
Based on the LLVM source code, we find that the type legalization part can't support all legalizing type and has a few problems need to be solve ,even for the lastest version (LLVM 6.0.0).

### Current Progress
We have done part of Legalization.cpp and the related files based on LLVM source code. We are trying to override the logic of legalization to improve the efficiency and support more arbitrary vector types, however, we also encounter some problems. Following is a key part of determine the correct element number after legalization and choose the correct legalization action.
The basic action has already been saved in a *legalization table*, such as i3 need to be promoted and i64 need to be splitted.


``` 
LegalizerInfo::legalizationAction(const SizeAndActionsVec &Vec, const uint32_t size) {
  
  ...
  
  //VecIdx stands for the current element size
  //action stands for the legalization operation need to be done to this element type
  
  int VecIdx = VecIt - Vec.begin();
  LegalizeAction action = Vec[VecIdx].action;
  
  switch (action) {
  
  case alreadyLegal:
  // Legal means this type can be supported directly or with some libraries.
  return {size, action};
  
  case Promotion:
  case Widen: {
    for (std::size_t i = VecIdx + 1; i < Vec.size(); ++i)
      if (!needsLegalizingToDifferentSize(Vec[i].action) && Vec[i].action != Unsupported)
        return {Vec[i].size, action};
    llvm_unreachable("");
  }
  
  case Narrow:
  // Only the scalar type can be supported by now
    if (Vec == SizeAndActionsVec({{1, Narrow}}))
      return {1, Narrow};
    llvm_unreachable("");
  
  case Demotion: {
    for (int i = VecIdx - 1; i >= 0; --i)
      if (!needsLegalizingToDifferentSize(Vec[i].action) && Vec[i].action != Unsupported)
        return {Vec[i].size, Demotion};
    llvm_unreachable("");
  }
  
  case Unsupported:
    return {Size, Unsupported};
  
  }
  llvm_unreachable("Action has an unknown enum value");
}

``` 
### Next to do
The current implementation has the following problems:
1. Can't support oddball types like <5 x i3>, <25 x i5> ... The reason is that the Promotion action and the Widen action  have the same implementation to get the legal elements number, which means the Promotion action is only part done. If we want to legalize a vecter type like <5 x i3>, we need a combination of Widen and Promotion.
2. The process is time-consuming. For example, if we need to transfer a illegal element type *n* to a nearby legal element type *m*, we need to interate all |n-m|-1 types between *n* and *m*. We once plan to use a *legal mask* to improve the efficiency, but if the legalization is working on a single vector of certain type, the mask loss it's advantage.
3. Can't support operations which might have more than one type, such as *icmp*.

## SWAR
### Brief description


2 WAYS:
* Optimize the add instruction through pass (take)
* Generate a new function 

Process:
* Recognize the "add" instruction
```
if (!inst.isBinaryOp() || inst.getOpcode() != Instruction::Add) {
    continue;
}
```
* Extract operands (i.e. 2 vectors) from the instruction
* Change vectors into a single vector
* Generate a mask for operating instruction
```
uint64_t generateMask(Value * elemSize, Value * numElems);
uint64_t mask = generateMask(elemSize, numElems);
```
* Operate AND, XOR, ADD instructions
```
llvm::BinaryOperator::CreateAnd(vector1, llvm::ConstantInt::get(Int32Ty, mask), "", inst);
```
* Reverse the single vector to the original vector type

Note:
If the vector size is over 128, then give up the SWAR optimization.

For the overflow, there is a way extending one bit used for the overflow. We choose to give up the overlfow and return the same data type as it was first input.

### Current Progress

### Next to do

## Reference
1. [Sierra: A SIMD Extension for C++](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/Sierra.pdf)
2. [Auto-vectorization for Image Processing DSLs](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/DSLs.pdf)
3. [Systematic Support of Parallel Bit Streams in LLVM](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/Systematic.pdf)
4. [MoviCompile : An LLVM based compiler for heterogeneous SIMD code generation](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/MoviCompile.pdf)
5. [Compiling For SIMD Within A Register](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/Compiling.pdf)
6. [SIMD Within A Register (e.g., using MMX)](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/MoviCompile.pdf)
7. [LLVM SIMDtoSWAR Pass](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/LLVM.pdf)
8. [SWAR Support for LLVM](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/SWAR_S.pdf)
9. [General-Purpose SIMD Within A Register: Parallel Processing on Consumer Microprocessors](https://github.com/lijianweizhuwei/CMPT886/blob/master/file/swar.pdf)
10. [LLVM Code Generator](http://llvm.org/docs/CodeGenerator.html#selectiondag-legalizetypes-phase)


