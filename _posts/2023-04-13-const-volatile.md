---
layout:     post
title:      "cv qualifier: const & volatile"
subtitle:   ""
date:       2023-04-13 16:02:00
author:     "Ethan"
catalog: false
header-style: text
tags:
    - CPP
---
> "cpp really sucks, no joke"


This is my first blog written purely in English.

# const
I've seen many people saying that a const variable is constant, even in cppreference.com, which however, I think is a misleading explanation. I prefer to regard const qualifier as **read-only**, but it might still change in the future (e.g. modified by other processes instead of you). When speaking of const, we need to distinguish const variable from pointer to const. Any direct write to const variable leads to compile-time error, any indirect write (through reference or pointer) leads to undefined behavior, so basically we can regard const variable as **real constant**. However, things are different when it comes to the pointer to const, which aims at delivering safe code, e.g.
```
void readOnly(const int *p){
    while(true){
        std::cout<<*p<<endl;
        sleep(1);
        // *p = 2;  *p is not modifiable inside function readOnly
    }
}
int main(){
    int *p = new int(1);
    std::thread t(readOnly, p);
    for(int i = 0; i < 5; ++i){
        *p = i; // *p is modifiable
        sleep(1);
    }
    t.join();
    return 0;
}
```
Pointer to const is used in such kind of situation, when passing pointer, you'd better think firstly whether I need to grant write permission to the invoked function. In the above example, readOnly function can not modify the content pointed by p, but it should be aware that the content might change.

Basically it's all for const in C, but const has additional semantics in C++, which is const member function.

A const member function promises no write to **object owned** variables i.e. **non-static** data members, **so it can modify static member variables**. Seems a bit of awkward. No worry, let's go back to think what really happens when invoking a member function in C++.

```
class A{
public:
    A(): val(1){
    }
    int getter() const{
        return val;
    }
	void foo(){

	}
private:
    int val;
};

int main(){
    const A a;
    a.getter();	//ok, A_getter(&a), pass object address to function getter
	a.foo();	//error, A_foo(&a), foo needs a pointer to non-constant
	A a2;
	a2.getter();	//ok
	a2.foo();	//ok
}
```

a.getter() actually implicilty pass a pointer (indicating the address of the object) to function getter, to be more specific, this pointer type is const A*, and member functions' signature of class A are
```
	int getter(const A* p);
	void foo(A* p);
```
So apparently for getter, you can not modify any member variable belongs to *p, but since static member variables do not belong to any object, they belong to the class, so const member function is able to modify static member variables.

And due to the fact that a pointer to constant can not convert to a pointer to non-constant, while a pointer to non-constant can convert to a pointer to constant, a constant object can only invoke const member function, a non-constant object can invoke both const and non-const member function.

# mutable
Const member functions aim at safe code: do not change what you should not change, however, sometimes we need to allow exceptions, e.g. a getter function may need to lock a mutex first before accessing data members under the situation of multithreaded programming. However, const member functions are not allowed to modify any member variables regardless of directly or indirectly (invoking another function) and mutex.lock() is a function invocation that will modify mutex itself, so contradiction happens and here mutable comes. 
By indicating a member variable as mutable, we can modify it inside a const member function, e.g.
```
class A{
public:
    A():val(1){
    }
    int getter() const{
        std::lock_guard<std::mutex> lock(m);
        return val;
    }
private:
    int val;
    mutable std::mutex m;
};
```
Attention, **declaring member variable as mutable actually make it lose the property of const when it belongs to a const object**, i.e. a mutable member variable of const object might change.
Of course, the following code does not make sense and produces compile-time error (you can not modify a constant anyway).
```
	mutable const mutex m;
```
# volatile
Volatile prohibits all possible compiler optimization and ensures every access to this variable is through every actual memory access instead of cache. Volatile is mostly used in inter-process communication through shared memory or communication with hardware through register. So volatile is mainly present in embedded system programming, take an example for explanation (from stack overflow),

```
unsigned int const volatile *status_reg; // assume these are assigned to point to the 
unsigned char const volatile *recv_reg;  //   correct hardware addresses

#define UART_CHAR_READY 0x00000001

int get_next_char()
{
    while ((*status_reg & UART_CHAR_READY) == 0) {
        // do nothing but spin
    }

    return *recv_reg;
}
```
Without volatile qualifier, compiler might regard the while loop as unnecessary because no write operation is applied to *states_reg in this program, so program might only read once after compiler optimization. However, status_reg is a pointer that points to a register exists in hardware, whose content might change from time to time depending on outside events. So, from the perspective of programmer, we need to clearly specify that these optimization is prohibited and this is what volatile qualifier does.

As you can see, for this snippet, the register pointer is defined with both const and volatile, it means the program itself is not allowed to change the value of what the pointer points to, and at the meanwhile, the content of the pointed might change from time to time.

Actually, if you wish, you can define a pointer like this
```
const volatile int* const volatile p;
```
The above indicates that both the pointer and the pointed is read-only and might change from time to time.

# Ref
- https://www.embedded.com/combining-cs-volatile-and-const-keywords/
- https://stackoverflow.com/questions/3141087/what-is-meant-with-const-at-end-of-function-declaration
- https://stackoverflow.com/questions/4592762/difference-between-const-const-volatile
- https://arne-mertz.de/2017/10/mutable/
