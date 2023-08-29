---
date: "2022-10-02T18:13:39+02:00"
title: Compilation - programmer perspective
---

Maintaining consistent and well organized physical code structure is challanging and underrestimated part of C++ systems development/maintance. Inproper management of interfaces and depenenies will pose maintainablitity and productivity challanges over time. The article aims to summarize simple solutions and attempts to define enforcement rules for best practices.


### Case study
Let's assume we are are about to develop small library consumed by many components of the system. Library will communicate with database containing workers entries. Implementation assumes caching for performace reasons.

**Solution 1 -** Quick reasearch reveals exising implementation of database connector. Implementing proxy with caching mechanism seems to fullfll all requirements. 

Possible hpp file:

```cpp
#include <vector>
#include <string>
#include "WorkersDBConnection.hpp"

class Workers {
public:
Workers(WorkersDBConnection* connection);

void add(const std::string& name);
void remove(const std::string& name);

private:
  std::vector<std::string> names;
  WorkersDBConnection* connection;
};
```

All non-functional requirements would be fulliled by such code but it could be written in a better way.

**Challanges:**
* Library users will indirectly include other headers. Every change in any of depended headers will tigger build and testing pipeline.
* Library is consumed by many translation units. Each indirectly included header will take part in compilation process slowing it down.
* Each translation unit will have more declared, but not defined symbols. This will have impact on linkage time.   
* There is limited control over what's included by depended header file (it may be owned by other team)
* 'Workers' class exposes private implementation details in header file. If implementation changes then all depended translation units have to be recompiled and tested.
* Unnecessary header dependencies results code growing in each library client 

### First improvement:

Dependency to ``WorkersDBConnection`` can be removed by technique called: forward-declation. At compilation time only size of declared object is required. As long as there are no direct method calls in header file there is no need for translation units to know any other symbol from that class. As result the pointer to declared type is sufficient. :

```cpp
#include <vector>
#include <string>

class WorkersDBConnection;

class Workers {
public:
Workers(WorkersDBConnection* connection);

void add(const std::string& name);
void remove(const std::string& name);

private:
  std::vector<std::string> names;
  WorkersDBConnection* connection;
};
```

GCC has flag ``-E`` allowing stoping compilation after preprocessing phase. Together with ``wc`` tool can roughtly measure impact on number of lines produced by compiler at first phase of compilation. 

Executing command **before change**:
```
gcc -E workers.cpp | wc
```
... produces output:
```
  36217   78064  892068
```

while executing command ***after change*** produces output:
```
26681   57656  658516
```

This **one-line change** excluded from translation unit roughtly **1/3 lines of code** to process. This may seem as not important, but remember we are developing a library consumed by houndreds/thoudands number of translation units in the system. This may be left in other library in the system that would be also popular and transitively adds **10000 lines** of code **to each translation unit.

What imact it has on comilation time?  

Before:
```
real    0m0.482s
user    0m0.427s
sys     0m0.055s
```

After:
```
real    0m0.376s
user    0m0.318s
sys     0m0.058s
```

There is **100 [ms] speed up** on compilation on **each translation unit!**. This is huge considering thousands of such unit compiled in the whole system! Imagine having 1000x0.1 [s] = 100 [s] speadup on each recompilation by forward declaring dependent type.

### Second improvement:

A header file can be considered as inteface of translation unit - it provides declarations and introduces symbols. Unfortunately declaration of a class in C++ cannot be split into chunks containing public and private members. There is no need for a library class user to know implementation details of prive section of the class. There should be no need for all library clients to need to recompile each translation unit that depends on such header. There is no need for all translation units to compile longer due to header included due to implementation details.

In order to mimic C++ inability to understand partialled class declartion we can modify the code in two ways:
* Extract pure virtual interface from public methods 
* Use PImpl idiom (pointer to implementation)

Both method solevs all described problems but there is no such thing as free lunch. Extracting pure virtual interface means that vtable will be created for Workers object. Creating anoter indirection is sometimes not an option due performance reasons. Such class cannot be constucted without a factory method returing pointer. This way concrete, inheriting class instance is constructed on heap rather than on stack. And again this approach means performance penaly. Additional indirection means more complicated code as well. 


Workers.hpp for virtual interface:
```cpp
#pragma once

#include <string>

class Workers {
public:
virtual ~Workers(){}

virtual void add(const std::string& name) = 0;
virtual void remove(const std::string& name) = 0;
};
```

Alternative for this approach is 'pointer to implementation' idiom. It requires creating implementation class called by interface class. This means performance penaly due to indirect calls to implementation and allocation on heap. 

Worker.hpp for PImpl
```cpp
#pragma once

#include <string>

class WorkersImpl;
class WorkersDBConnection;

class Workers {
public:
Workers(WorkersDBConnection* connection);

void add(const std::string& name);
void remove(const std::string& name);

private:
  WorkersImpl* pImpl;
  WorkersDBConnection* connection;
};
```

What impact such change has on compilation time? Let's put pImpl solution as example (there should be no significant difference between theese two as resulting header is almost the same)
```
real    0m0.324s
user    0m0.273s
sys     0m0.051s
```

This is not significant, but there was only std::vector dependency removed. Second step (one way or another) in real-life scenario can give incredible improvements in terms of compilation time!. 

### Final thoughts:

A header file is tranlation unit interface and should not be cluttered with unnecessary dependencies. This is extremely important when designing header file that is a library public interface. The article described techniques aiming to resolve related problems, but it's not recommented to apply then everwhere in the code. Gains should allways overweight the cost.

