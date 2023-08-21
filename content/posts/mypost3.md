---
title: "What are symbols and why they are needed?"
date: 2022-11-16T18:38:01+01:00
draft: false
---


In order to fully undestand usage symbols you will need to undestand basics of C++ compilation. This article will explain what are symbols in context of compilation by walking you step by step through compilation phases. 

<!--more-->

### Compilation steps

Compilation will be explained on following 'Hello World' example:
```cpp
#include <iostream>

int main(){
   std::cout << "Hello world" << std::endl;
   return 0;
}
```  

Then the following compilation steps are performed:
  
#### 1. Preprocessing

This is mostly about **copy-pasting text files** where they were included. Also macro expansion is done and some logic can be applied there, but you should avoid playing with preprocessor as it does not know types. Command used for stopping compilation after preprocessing in gcc is:
```bash
g++ -E main.cpp -o main.i
```
If executed it outputs **32098 lines of iostream + couple of lines of main.cpp** file.

At this point your main.cpp code can be really messed up and you will not see any compiler complaints. No syntax and types checks are performed. You will only know if you mess up syntax around preprocessor macros. 

#### 2. Translating into assembly

This step will translate main.i into main.s ASCII assembly file. It is still not undestandable by an operation system, so it cannot be loaded into memory and started. It's just a human-readable **textual representation of assembly code**. Command used for perform this step is:
```bash
g++ main.i -S -o main.s
```  

And it ouputs 85 lines of assembly file. There is bunch of code divided by sections and I will not dive deep into it. At this point you have to know that there will be many such files in larger project, so from your perspective is's just a **file containing some chunks of code in ASCII form**. I'm putting bellow couple of lines of the output:
```asm
.LFB2201:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        movl    $65535, %esi
        movl    $1, %edi
        call    _Z41__static_initialization_and_destruction_0ii
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc

```

#### 3. Translating into an object file

This step result with creating and object file in format allowing to combine many of such files into one executable object. The file is divided into sections with code, data, read-only data, symbol table, debbuging symbols and some others mainly required for relocations during linkage.

```bash
g++ main.s -c -o main.o
``` 

The most interesting part for us at this point is section with symbol table. It can be seen by executing:
```bash
nm -u main.o
```

The command will show undefined symbols only and it this case contains
```bash
_atexit
                 U __dso_handle
                 U _ZNSolsEPFRSoS_E
                 U _ZNSt8ios_base4InitC1Ev
                 U _ZNSt8ios_base4InitD1Ev
                 U _ZSt4cout
                 U _ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
                 U _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
```

It is basically a list of symbols that the object knows about (were declared) but has no definiton of. During the linkage it will look for them and properly reallocate code and data. Each symbol outputed above is mangled which means it decorates used name with some more information allowing to uniquely indentify function, structure, class, variable ect. As function can be overloaded then parameters it should contains parameter types. Manging is reversable and can be done in this page: http://demangler.com/

#### 4. Linking phase 

This step will result with an executable file that can be loaded into memory and executed. At this compilation stage there is a set of object files with plenty of undefined symbols, hopefully defined by other objects. But in our case there is only one object file (main.o), right? So why we still need this phase?

In fact there are more object existing in the system that compiler already know about. They don't have to be given to it explilicty in linkage command. But we can stil know what linker does for us by using additional flag -v (verbose):
```bash
g++ -v main.o -o program
```

Among other things it outputs gcc options used by compiler and linker:
```cfg
COLLECT_GCC_OPTIONS='-v' '-o' 'program' '-shared-libgcc' '-mtune=generic' '-march=x86-64' '-dumpdir' 'program.'
 /opt/rh/devtoolset-11/root/usr/libexec/gcc/x86_64-redhat-linux/11/collect2 -plugin /opt/rh/devtoolset-11/root/usr/libexec/gcc/x86_64-redhat-linux/11/liblto_plugin.so -plugin-opt=/opt/rh/devtoolset-11/root/usr/libexec/gcc/x86_64-redhat-linux/11/lto-wrapper -plugin-opt=-fresolution=/tmp/cc8Fkmdb.res -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lgcc --build-id --no-add-needed --eh-frame-hdr --hash-style=gnu -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o program /lib/../lib64/crt1.o /lib/../lib64/crti.o /opt/rh/devtoolset-11/root/usr/lib/gcc/x86_64-redhat-linux/11/crtbegin.o -L/opt/rh/devtoolset-11/root/usr/lib/gcc/x86_64-redhat-linux/11 -L/opt/rh/devtoolset-11/root/usr/lib/gcc/x86_64-redhat-linux/11/../../../../lib64 -L/lib/../lib64 -L/usr/lib/../lib64 -L/opt/rh/devtoolset-11/root/usr/lib/gcc/x86_64-redhat-linux/11/../../.. main.o -lstdc++ -lm -lgcc_s -lgcc -lc -lgcc_s -lgcc /opt/rh/devtoolset-11/root/usr/lib/gcc/x86_64-redhat-linux/11/crtend.o /lib/../lib64/crtn.o
```

There is list of libraries given as arguments with '-l' suffix and `-lstdc++` among them containing standard library functions definitions. In our example main function uses `std::cout` (mangled to `_ZSt4cout`) that is not defined in that object file. From that reason 'nm main.o' will returns:
```bash
U _ZSt4cout
```

When the program is linked then definition is provided and symbol is no longer unresolved:
```bash
0000000000404060 B _ZSt4cout@GLIBCXX_3.4
```


### Why symbols are needed for linkage?

Before linkage there is group of obj files containing 'references' to other objects (Object A may call function defined in Object B, refer to a variable, ect). When code is compiled to assembly 

#### Example:
`main.cpp:`

```cpp
void foo();

int main() {
   foo();
   return 0;
}

```

`Foo.cpp:`

```cpp
void foo(void) {
    static int i = 0;
    i++;
}
```

Each file is compiled into a relocatable object file. Main function (from translation unit main.o) calls function `foo()` that is defined in other translation unit (Foo.o). When we look at dissassembly of each object the we could see that call to function `foo()` does not refer to definition in `Foo.o`. Address of the function definition is unknown at compilation time of `Main.o`. Starting from address 0x05 there is address of the function `foo()` 0x00000000. 

```diff
0000000000000000 <main>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   e8 00 00 00 00          call   9 <main+0x9>
   9:   b8 00 00 00 00          mov    $0x0,%eax
   e:   5d                      pop    %rbp
   f:   c3                      ret
```

At this point compiler produces entry in symbol table describing undefined reference by symbol name, type and offset from beggining of the section:

```asm
RELOCATION RECORDS FOR [.text]:
OFFSET           TYPE              VALUE 
0000000000000005 R_X86_64_PLT32    _Z3foov-0x0000000000000004
```

As it starts 5 bytes from the beggining of section text then offset equals 0x05. Undefined symbol will be found by linker in next stage in Foo.o containing full defintion of the function:

```diff
0000000000000000 <_Z3foov>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   8b 05 00 00 00 00       mov    0x0(%rip),%eax        # a <_Z3foov+0xa>
   a:   83 c0 01                add    $0x1,%eax
   d:   89 05 00 00 00 00       mov    %eax,0x0(%rip)        # 13 <_Z3foov+0x13>
  13:   90                      nop
  14:   5d                      pop    %rbp
  15:   c3                      ret
```

While combining sections together linker will update addresses to their final values (both intruction addresses and reference values). These will correspond to memory where program will be actuall placed in memory after loading. 
Going back to our example: when binary is linked then call to `foo()` references to relative address `0x07` containing defintion of the function. `.text` sections of 2 different translation units were merged together and their addresses where updated accordingly.


```diff
0000000000401166 <main>:                                                                            
  401166:       55                      push   %rbp                                                 
  401167:       48 89 e5                mov    %rsp,%rbp                                            
  40116a:       e8 07 00 00 00          call   401176 <_Z3foov>                                     
  40116f:       b8 00 00 00 00          mov    $0x0,%eax                                            
  401174:       5d                      pop    %rbp                                                 
  401175:       c3                      ret                                                         
                                                                                                    
0000000000401176 <_Z3foov>:                                                                         
  401176:       55                      push   %rbp                         
  401177:       48 89 e5                mov    %rsp,%rbp                                      
  40117a:       48 8d 05 8f 0e 00 00    lea    0xe8f(%rip),%rax 
  401181:       48 89 c6                mov    %rax,%rsi                                            
  401184:       48 8b 05 65 2e 00 00    mov    0x2e65(%rip),%rax                                    
  40118b:       48 89 c7                mov    %rax,%rdi                                            
  40118e:       e8 dd fe ff ff          call   
  401193:       48 8b 15 5e 2e 00 00    mov    0x2e5e(%rip),%rdx 
```

### Shared libraries

If multiple executables are linked against the same static library then copy of the code is present in each of file. It increases space used by executables by the system, but also increases memory memory consuption of loaded programs. 

Shared libraries address these disadvantages by performing linkage at runtime of the program. Library is loaded into memory and shared among many executables. When executable references symbol defined in such libary the address is unknown until loader does it's job. 



