---
title: "Modules - solution for compilation challanges"
date: 2022-11-02T18:43:19+01:00
draft: true
---

As described in previous post C++ compilation suffers from many challanges and requres from developer to understand compilation and linking in order to maintain well structured code.  

### Why we use header files?

Header files are used to increase code maintaintability by sharing declaration of names used accross translation units (cpp files). Each cpp file is compiled separately hence compiler has to know types used. It is possible to build software without using header files, but it would require duplication of declarations and would be maintainability nightmare. 

worker.hpp
```cpp
#pragma once

void doWork();
```
job1.cpp
```cpp
#include "worker.hpp"

...
doWork();
...
```
job2.cpp
```cpp
#include "worker.hpp"

...
doWork();
...
```

After preprocessing preprocessing the code in ``worker.hpp`` file is copy pased into each translation unit that has ``#include "worker.hpp"``. After that step declarion of function ``doWork()`` is present in each translation unit. The same could be achieved by declaring function in ``job1.cpp`` and in ``job2.cpp``. Header file would be no longer necessary:

job1.cpp:
```cpp
void doWork();
...
doWork();
...
```

job2.cpp
```cpp
void doWork();
...
doWork();
...
```

Consequences of such approach:
* If signature of function `doWork` changes, or function is no longer used then manual update of all cpp files has to be done.
* More difficult to find where `doWork` is accually defined. 
* Less readable code

### What's wrong with header files?

If there is no reason to duplicate declaration of names accross many cpp files then why header file are not flowless solutions for that?

Headers having only shared names would solve the problem, but in reality there are also names put in headers that are not shared, but 'private' to its tranlation unit. This is result of developers not paying attention to such separation, but also as result of language limitations. 

For example it is not possible in C++ to split class declarations into ``public`` and ``private`` part:

```cpp
#pramga once

#include "ClassWithLargeHeader.hpp"

#include <string>
#include <vector>

class Worker {
public:
void doWork();

private:
 std::vector<std::string> tasks;
 ClassWithLargeHeader obj;
};
```

As a result name as ```ClassWithLargeHeader``` had to be introduced into header file, despite beeing implementation detail of ```Worker``` class. None of cpp that uses ``Worker`` class has to know about ``obj`` and ``tasks`` type.  

``ClassWithLargeHeader`` happens to have many dependencies itself and it's inclusion have large impact on compilation. In real life scenario ``Worker`` be included without clients knowing about ``ClassWithLargeHeader`` heavieness.

Answering to the main question: **Headers tent to introduce more declarations that are needed!**

### What is more than necessary?

Case 1:

Header file intruduces name used only in corresponding to header translation unit.

Implemenation proposition:

Getting list of 'unnecessary names declarations/definitions' by keeping:
* List of names used in each translation unit 
* List of names declared by each header file
* List of names defined by each header file

Challanges:
* There is not difference between header filed named after cpp file and any other header included in that cpp. There is no header to implementation correposndence other than convention. This means that there is no way to tell without compilation which header declares symbols for given source file. 

* Header files usually contain declartions of symbols but definitions are also allowed. Symbol defined in a header file will have it's own definition in translation units including that header. 



Case 2:

Header file introduces symbols that are not used by corresponding to header translation unit.






### How moules solve this problem?
