---
layout: post
title: Process memory 구조
tags: [dev, cs, process, memory, linux]
use_math: true
feature_image: https://academy.avast.com/hubfs/New_Avast_Academy/What%20is%20RAM%20memory/What_is_RAM-Hero.jpg
---

<!-- more -->
_이 포스트는 [In-Memory Layout of a Program (Process)](https://gabrieletolomei.wordpress.com/miscellanea/operating-systems/in-memory-layout/) 를 공부하며 번역한 포스트입니다._

이번 포스트에서는, program 이 실행될 때 실제로 main memory 에 어떤 구조를 가지고 실행되는지 설명한다. 여기에서는 **32-bit x86** 아키텍쳐 머신에서 실행되는 **multitasking Linux OS** 환경을 가정한다.

Multitasking OS 환경에서 실행되는 모든 process 는 각각의 **virtual address space**를 가진다. 이 address space 는 32-bit 환경에서 2<sup>32</sup> = 4GB 의 크기를 가지며, OS kernel 의 page table 에 의해 관리되고 실제 physical address 로 mapping 된다.  
OS kernel 또한 그 자체로 하나의 process 이므로, virtual address space 를 가지며 OS kernel process 를 위한 전용 address space 를 가진다. Linux OS 에서 총 4GB 의 virtual address space 중 앞 3GB 부분은 user process 를 위해 사용되며, 마지막 1GB (`0xC0000000`~`0xFFFFFFFF`) 부분은 OS kernel 의 전용 address space 로 사용된다. 이를 그림으로 나타내면 아래와 같다.

{% include figure.html image="/assets/img/20201016/memory_layout.jpg" position="center" caption="Virtual memory spaces for kernel and user processes" %}

자세한 내용은, [Virtual Memory, Paging, and Swapping](https://gabrieletolomei.wordpress.com/virtual-memory-paging-and-swapping) post 를 참고하자.

아래 그림은 User mode process 의 virtual memory space 부분에 대한 상세 내용을 나타낸다.
{% include figure.html image="/assets/img/20201016/program_in_memory2.png" position="center" caption="User mode process memory space" %}

- __Text__: program 의 code (instructions) 가 저장되는 공간. Overflow 로 인해 덮어써지는 현상을 방지하기 위하여 `heap` 이나 `stack` 영역 보다 앞쪽 (lower address) 에 자리한다.
- __Data__: initialized data segment, 명시적으로 초기화된 __global(전역)__ 변수와 __static__ 변수들을 저장하는 공간이다. 기본적으로 data segment 는 read-only 속성을 가지지만, 실제로는 read-only 영역과 read-write 영역으로 구분된다. 예를 들어, 변수의 선언 구문에 따른 저장 영역은 아래와 같다.
 
```c
char s[] = "hello world";
int debug = 1;
// global (declared outside the main or any func.)
static int i = 10;
// 변수의 값이 추후 변경될 가능성이 있으므로, read-write 영역에 저장

const char* str = "hello world";
// 'str' pointer 변수는 read-write 영역에 저장,
// 'hello world' string literal 은 read-only 영역에 저장.
```

- __BSS__: uninitialized data segment. 명시적으로 초기화되지 않은 모든 변수는 OS kernel 에 의해 arithmetic 0 값으로 초기화되어 이 영역에 저장된다. BSS 영역은 (당연히) read-write 속성을 가짐.
- __Stack__: program stack 을 저장하는 영역. `0xBFFFFFFF` (OS kernel 영역의 바로 아래) 에서부터 아래쪽 (lower address) 으로 증가하며, 프로그램의 흐름에서 function call 에 필요한 모든 data 가 이곳에 저장된다. function call 시 stack 에 저장되는 정보를 __stack frame__ 이라 하며, 호출하는 함수의 입력으로 주어질 parameter set, 함수 내부에서 선언된 local variables, 함수 종료 후 되돌아올 return address 로 구성된다. __stack pointer__ register 에는 현재 stack 의 top point 가 저장되는데, 함수의 호출에 따라 stack 의 용량이 증가하다가 이 pointer 가 __heap pointer__ 를 만나거나 혹은 `RLIMIT_STACK` 값에 도달하게 되면 우리가 잘 알고 있는 __stack overflow__ 가 발생한다.
- __Heap__: Dynamically allocated memory 를 위한 공간. runtime 에 할당이 필요한 모든 메모리는 이 영역에 저장된다. (C 에서의 `malloc` 등) BSS 영역의 바로 위에서 시작하여 위쪽 (higher address) 으로 증가하며, process 에서 사용되는 모든 shared library 등 동적으로 로드된 module 들과 공유된다.

