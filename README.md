# Try new awesome fnshift FunctionPass in Action
Function pass proposed to use with ARM arch.  

## Prerequisites
* Linux platform
* Ability to build llvm tree
* Updated llvm repository

### Apply git patch
From **{llvm-src-folder}** folder:  

    git apply {patch-file-name}
    
### Building
Create folder **{llvm-build-folder}** and move there: 

    mkdir {llvm-build-folder}; cd {llvm-build-folder}
    
Configure project:  

    cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD="ARM;X86" {llvm-src-folder}/llvm/
    
Build the tree (N - number of cores claimed for building):  

    make -jN

### Test C snippet

```c
    #include <stdio.h>
    #include <stdint.h>
    
    int fnshift(unsigned x, unsigned shift) {
        __builtin_assume(shift < 33);
        return (shift == 32) ? 0 : (x >> shift);
    }
    
    int main() {
        int toprint = fnshift(512,4);
        printf("Shifted: %d\n", toprint);
        return 0;
    }
```
### Generate llvm IR
Make folder for experiments and create **main.c** file with the content above.  
Target triple name given just for example and don't make any sence excluding **arm** arch name.   
Optimization level also given for example and don't influence on the process.  

    clang -emit-llvm -O0 -S main.c -o fnshift_O0.ll --target=armv7-linux-gnueabihf
    
After the command you will have **fnshift_O0.ll** file with generated llvm IR.  

### Apply pass   

    opt -load {llvm-build-folder}/lib/LLVMFnshift.so -fnshift -S < fnshift_O0.ll > fnshift_O0_opt.ll
    
Now you can take a look to changes in IR; between **fnshift_O0.ll** and **fnshift_O0_opt.ll** files

### Generate ARM asm
You can also take a look on difference between generated ARM asm:
Generate asm for IR with no optimizations:  

    llc --mtriple=armv7-linux-gnueabihf fnshift_O0.ll -o fnshift.s
    
Generate asm for IR with fnshift optimization:  

    llc --mtriple=armv7-linux-gnueabihf fnshift_O0_opt.ll -o fnshift_opt.s
    
### Runing tests  
Submitted patch, also, contains three tests for checking corecteness of the pass's  
working:   

**llvm/test/Transforms/InstSimplify/fnshift.ll**  
Checks if the pass works how expected with function compiled with -O0 flag  

**llvm/test/Transforms/InstSimplify/fnshiftOpt.ll**  
Checks if the pass works how expected with function compiled with -O!{0} flag  

**llvm/test/CodeGen/ARM/fnshift.ll**  
Checks if correct arm instruction is generating

**Next steps assumes that you already have builded llvm tree.**  

To run tests firstly you should add path to the shared liblrary with the pass  
to **lit.cfg.py** file by replacing *{llvm-build-folder}* part of the string  
*fnshiftsopath* onto your specific path:  

```python
    fnshiftsopath = "{llvm-build-folder}/lib/LLVMFnshift.so"        
    config.substitutions.append(('%fnshiftpath', fnshiftsopath))
```

**%fnshiftpath** substitutes are using in test's RUN commands.

Now, you can run tests for example by following command:  
    
    llvm-lit {llvm-src-folder}/llvm/test/Transforms/InstSimplify/fnshiftOpt.ll
