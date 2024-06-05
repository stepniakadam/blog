---
date: "2023-11-04T18:13:39+02:00"
title: Implicit conversions - sneaky cases
---

### Implicit conversions - back to basics

Compiler produces assembly code and assembly does not like when 2 operands have different types (and sizes as a result). Types need conversions and C++ allow them to be implicit. As result of compiler seeing type mismatch  


What is output of following program?
```c++
int foo(bool) { return 1; }
int foo(const char*) { return 2; }

int main() {
    std::cout << foo("abc") << std::endl;
    return 0;
}
```

Answer: `2`

Then what is expected output of following program?
```c++
int foo(bool) { return 1; }
int foo(const std::string&) { return 2; }

int main() {
    std::cout << foo("Test") << std::endl;
    return 0;
}
```
Answer: 1

This is because when it comes to : "A standard conversion sequence is always better than a user-defined conversion sequence or an ellipsis conversion sequence."

```c++
std::cout << (sizeof(int) > -1)  << std::endl;
```
Answer: `0`

This is because 
