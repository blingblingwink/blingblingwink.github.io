---
layout:     post
title:      "多线程原子操作"
subtitle:   "Atomic in multithreading"
date:       2022-06-29 14:35:00
author:     "Ethan"
catalog: false
header-style: text
tags:
    - 编程基础
    - CPP
---
众所周知，多线程同时操作一个变量存在线程安全的问题，举个例子：

```
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
#include <atomic>
#include <chrono>
using namespace std;

int main(){
	constexpr unsigned num_threads = 4;
	constexpr unsigned add_times = 100000;
	vector<thread> threads(num_threads);
	unsigned sum = 0;
	auto f = [&sum]{
		for(size_t i = 0; i < add_times; ++i){
			++sum;
		}
	};
	
	auto start=chrono::high_resolution_clock::now();
	for(size_t i = 0; i < num_threads; ++i){
		threads[i] = thread(f);	
	}
	for(auto &t:threads){
		t.join();
	}
	auto end=chrono::high_resolution_clock::now();
	auto duration=chrono::duration_cast<chrono::milliseconds>(end-start);
	cout<<"final sum: "<<sum<<" duration: "<<duration.count()<<"ms"<<endl;
	return 0;
}

```

运行结果如下：

`final sum: 120604 duration: 2ms`

预期值应该是400000，有很多修改丢失了。我们可以选择使用互斥量来访问，如下所示：

# 互斥量
```
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
#include <atomic>
#include <chrono>
using namespace std;

int main(){
	constexpr unsigned num_threads = 4;
	constexpr unsigned add_times = 100000;
	vector<thread> threads(num_threads);
	unsigned sum = 0;
	mutex sum_mutex;
	auto f = [&sum,&sum_mutex]{
		for(size_t i = 0; i < add_times; ++i){
			lock_guard<mutex> sum_lock(sum_mutex);
			++sum;
		}
	};
	
	auto start=chrono::high_resolution_clock::now();
	for(size_t i = 0; i < num_threads; ++i){
		threads[i] = thread(f);	
	}
	for(auto &t:threads){
		t.join();
	}
	auto end=chrono::high_resolution_clock::now();
	auto duration=chrono::duration_cast<chrono::milliseconds>(end-start);
	cout<<"final sum: "<<sum<<" duration: "<<duration.count()<<"ms"<<endl;
	return 0;
}
```

结果如下所示：

`final sum: 400000 duration: 61ms`

可发现，值是正确的，但耗时大大增加，由原来的2ms增长为61ms，现在我们把sum改成C++的原子类型试一下(不用原子类型，把++sum换成__sync_fetch_and_add(&sum,1)也是一样的)：

# 原子类型
```
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
#include <atomic>
#include <chrono>
using namespace std;

int main(){
	constexpr unsigned num_threads = 4;
	constexpr unsigned add_times = 100000;
	vector<thread> threads(num_threads);
	atomic<unsigned> sum={0};
	auto f = [&sum]{
		for(size_t i = 0; i < add_times; ++i){
			++sum;
		}
	};
	
	auto start=chrono::high_resolution_clock::now();
	for(size_t i = 0; i < num_threads; ++i){
		threads[i] = thread(f);	
	}
	for(auto &t:threads){
		t.join();
	}
	auto end=chrono::high_resolution_clock::now();
	auto duration=chrono::duration_cast<chrono::milliseconds>(end-start);
	cout<<"final sum: "<<sum<<" duration: "<<duration.count()<<"ms"<<endl;
	return 0;
}
```

结果如下所示：

`final sum: 400000 duration: 11ms`

可发现值是正确的，耗时是11ms，远低于使用互斥量的耗时。

但我们回想一下原子操作的定义：不会被中断的操作，如果是单核CPU，上述代码确实没问题，但如果是多核CPU同时执行原子操作呢？其实仍然是没有问题的，因为上述这个众所周知的原子操作的定义其实不准确，原子操作不仅仅不会被中断，并且一个核在执行原子操作时，别的核都不能再对同一内存数据项进行原子操作。我们可以做个实验，把线程绑定到不同的核上运行，看下结果如何。

```
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
#include <atomic>
#include <chrono>
using namespace std;

int main(){
	constexpr unsigned num_threads=4;
	constexpr unsigned add_times=100000;
	vector<thread> threads(num_threads);
	atomic<unsigned> sum={0};
	mutex io_mutex;
	auto f = [&io_mutex, &sum](int i){
		{
			lock_guard<mutex> io_lock(io_mutex);
			cout<<"thread "<<i<<" running on CPU "<<sched_getcpu()<<endl;
		}
		for(size_t i = 0; i < add_times; ++i){
			++sum;
		}
	};
	
	auto start=chrono::high_resolution_clock::now();
	for(size_t i = 0; i < num_threads; ++i){
		threads[i] = thread(f,i);
		cpu_set_t cpuset;
		CPU_ZERO(&cpuset);
		CPU_SET(i,&cpuset);
		int rc = pthread_setaffinity_np(threads[i].native_handle(),sizeof(cpu_set_t),&cpuset);
		if(rc != 0){
			cerr<<"error calling pthread_setaffinity_cp"<<rc<<endl;
		}
	}
	for(auto &t:threads){
		t.join();
	}
	auto end=chrono::high_resolution_clock::now();
	auto duration=chrono::duration_cast<chrono::milliseconds>(end-start);
	cout<<"final sum: "<<sum<<" duration: "<<duration.count()<<"ms"<<endl;
	return 0;
}
```

结果如下所示：

```
thread 1 running on CPU 1
thread 3 running on CPU 3
thread 2 running on CPU 2
thread 0 running on CPU 0
final sum: 400000 duration: 11ms
```

可知，将线程绑定到不同的CPU上原子操作仍然是正确的，那么原子操作是如何在多核CPU的情况下实现线程安全的呢，我们用objdump命令看一下原子操作的反汇编的结果（部分）：

` 401f73:       f0 0f c1 02             lock xadd %eax,(%rdx)`

代码中sum是原子类型，++sum调用了原子类型的重载函数，其中对应的最重要的汇编代码就是这句lock xadd，lock是x86平台下汇编指令的一个前缀，使用了这个前缀的指令在执行时，会对总线进行加锁，使得其他CPU无法再访问同一内存数据项，也就无法执行原子操作了，这就回答了刚才的疑问。

# 参考文献
- https://wiki.osdev.org/Atomic_operation
- https://eli.thegreenplace.net/2016/c11-threads-affinity-and-hyperthreading/
