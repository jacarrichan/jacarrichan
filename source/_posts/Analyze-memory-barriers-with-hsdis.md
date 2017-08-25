title: 使用hsdis分析Java内存屏障
date: 2017-01-12 15:48:46
tags: [volatile,hsdis,JIT]
---


内存屏障（Memory Barrier，或有时叫做内存栅栏，Memory Fence）是一种CPU指令，用于控制特定条件下的重排序和内存可见性问题。Java编译器也会根据内存屏障的规则禁止重排序。

<!-- more -->

# 工具准备
1. 从 [FCML](https://sourceforge.net/projects/fcml/files/fcml-1.1.2/ 'fcml')下载适合自己操作系统和CPU的HSDIS文件,我的是hsdis-1.1.1-win32-amd64.zip

1. 打开压缩包，将hsdis-*.dll  复制到jvm.dll所在的目录。有可能是jre，也有可能是jdk的jre，看你环境变量配置了的


# 编写JVM会JIT的代码

```
class VolatileBarrierExample {

	int a;
	volatile int v1 = 1;
	volatile int v2 = 2;

	void readAndWrite() {
		int i = v1;		//volatile read 
		int j = v2;		//volatile read 	
		a = i + j;		
		v1 = i + 1;		//volatile write
		v2 = j * 2;		//volatile write
	}

    public static void main(String[] args){
        final VolatileBarrierExample ex=new VolatileBarrierExample();
        for(int i=0;i<10000000;i++)
            ex.readAndWrite();
    }
}
```

* 循环多一点，触发JVM的JIT机制

##  编译和执行
javac编译就忽略不写；

执行的命令如下
> java -XX:CompileThreshold=1 -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand="compileonly VolatileBarrierExample 
readAndWrite" VolatileBarrierExample

解释如下：

1. CompileThreshold 将方法编译成机器码的触发阀值，可以理解为调用方法的次数
1. UnlockDiagnosticVMOptions  解锁诊断指令，不然PrintAssembly 没用
1. PrintAssembly 打印汇编指令
1. -XX:CompileCommand="compileonly class method"  指定要分析的方法

执行上述命令后得到如下汇编信息

```
0	C:\Users\Jacarri\logs>java -XX:CompileThreshold=1 -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand="compileonly VolatileBarrierExample readAndWrite" VolatileBarrierExample
1	Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
2	CompilerOracle: compileonly VolatileBarrierExample.readAndWrite
3	Loaded disassembler from D:\Java\jdk1.7.0_51\jre\bin\server\hsdis-amd64.dll
4	Decoding compiled method 0x0000000002db2b90:
5	Code:
6	Argument 0 is unknown.RIP: 0x2db2cc0 Code size: 0x00000078
7	[Disassembling for mach='amd64']
8	[Entry Point]
9	[Constants]
10   # {method} 'readAndWrite' '()V' in 'VolatileBarrierExample'
11   #           [sp+0x20]  (sp of caller)
12   0x0000000002db2cc0: mov     r10d,dword ptr [rdx+8h]
13   0x0000000002db2cc4: shl     r10,3h
14   0x0000000002db2cc8: cmp     rax,r10
15   0x0000000002db2ccb: jne     2d87a60h          ;   {runtime_call}
16   0x0000000002db2cd1: nop
17   0x0000000002db2cd4: nop     dword ptr [rax+rax+0h]
18   0x0000000002db2cdc: nop
19	[Verified Entry Point]
20   0x0000000002db2ce0: sub     rsp,18h
21   0x0000000002db2ce7: mov     qword ptr [rsp+10h],rbp  ;*synchronization entry
22                                                 ; - VolatileBarrierExample::readAndWrite@-1 (line 8)
23   0x0000000002db2cec: mov     r11d,dword ptr [rdx+10h]
24                                                 ;*getfield v1
25                                                 ; - VolatileBarrierExample::readAndWrite@1 (line 8)
26   0x0000000002db2cf0: mov     r10d,dword ptr [rdx+14h]
27                                                 ;*getfield v2
28                                                 ; - VolatileBarrierExample::readAndWrite@6 (line 9)
29   0x0000000002db2cf4: mov     r9d,r11d
30   0x0000000002db2cf7: add     r9d,r10d
31   0x0000000002db2cfa: mov     dword ptr [rdx+0ch],r9d
32   0x0000000002db2cfe: inc     r11d
33   0x0000000002db2d01: mov     dword ptr [rdx+10h],r11d
34   0x0000000002db2d05: shl     r10d,1h
35   0x0000000002db2d08: mov     dword ptr [rdx+14h],r10d
36   0x0000000002db2d0c: lock add dword ptr [rsp],0h  ;*putfield v2
37                                                 ; - VolatileBarrierExample::readAndWrite@28 (line 12)
38   0x0000000002db2d11: add     rsp,10h
39   0x0000000002db2d15: pop     rbp
40   0x0000000002db2d16: test    dword ptr [0ea0000h],eax
41                                                 ;   {poll_return}
42   0x0000000002db2d1c: ret
43   0x0000000002db2d1d: hlt
44   0x0000000002db2d1e: hlt
45   0x0000000002db2d1f: hlt
46	[Exception Handler]
47	[Stub Code]
48   0x0000000002db2d20: jmp     2dafa60h          ;   {no_reloc}
49	[Deopt Handler Code]
50   0x0000000002db2d25: call    2db2d2ah
51   0x0000000002db2d2a: sub     qword ptr [rsp],5h
52   0x0000000002db2d2f: jmp     2d89000h          ;   {runtime_call}
53   0x0000000002db2d34: hlt
54   0x0000000002db2d35: hlt
55   0x0000000002db2d36: hlt
56   0x0000000002db2d37: hlt

```

##  汇编代码注释

部分代码的解释如下

```
12   0x0000000002db2cc0: mov     r10d,dword ptr [rdx+8h] ##把寄存器地址[rdx+8h]中的双字型（32位）数据赋给寄存器r10d，ptr指针 地址
13   0x0000000002db2cc4: shl     r10,3h  ##r10左移3h
14   0x0000000002db2cc8: cmp     rax,r10
15   0x0000000002db2ccb: jne     2d87a60h          ;   {runtime_call} ##上文cmp结果不为零(或不相等)则转移
16   0x0000000002db2cd1: nop   ##延时一个机器周期
17   0x0000000002db2cd4: nop     dword ptr [rax+rax+0h]
18   0x0000000002db2cdc: nop
19	[Verified Entry Point]
20   0x0000000002db2ce0: sub     rsp,18h   ## rsp=18-rsp
---> [int i = v1; //volatile read ]
21   0x0000000002db2ce7: mov     qword ptr [rsp+10h],rbp  ;*synchronization entry  ##将64位的rbp赋给栈地址为10h上
22                                                 ; - VolatileBarrierExample::readAndWrite@-1 (line 8)
23   0x0000000002db2cec: mov     r11d,dword ptr [rdx+10h]  ##将32位的寄存器地址10h赋给寄存器r11d上
24                                                 ;*getfield v1
25                                                 ; - VolatileBarrierExample::readAndWrite@1 (line 8)
---> [int j = v2; //volatile read ]
26   0x0000000002db2cf0: mov     r10d,dword ptr [rdx+14h] ##将32位的寄存器地址14h赋给寄存器r10d上
27                                                 ;*getfield v2
28                                                 ; - VolatileBarrierExample::readAndWrite@6 (line 9)
29   0x0000000002db2cf4: mov     r9d,r11d  ##寄存器11d赋给寄存器9d
---> [a = i + j; ]
30   0x0000000002db2cf7: add     r9d,r10d ##寄存器10d加寄存器9d赋给寄存器9d
31   0x0000000002db2cfa: mov     dword ptr [rdx+0ch],r9d ##寄存器9d赋给寄存器地址为0ch上
--->   [v1 = i + 1; //volatile write]
32   0x0000000002db2cfe: inc     r11d  ##寄存器11d 自增1 
33   0x0000000002db2d01: mov     dword ptr [rdx+10h],r11d ## 寄存器11d的赋给寄存器地址为10h上
--->   [v2 = j * 2; //volatile write]
34   0x0000000002db2d05: shl     r10d,1h  ##  寄存器10d左移1h 
35   0x0000000002db2d08: mov     dword ptr [rdx+14h],r10d  ##寄存器10d赋给双字地址为14h
36   0x0000000002db2d0c: lock add dword ptr [rsp],0h  ;*putfield v2 ##  0h加双字节地址为rsp的值赋给rsp，并立即对其他cpu可见
37                                                 ; - VolatileBarrierExample::readAndWrite@28 (line 12)
38   0x0000000002db2d11: add     rsp,10h
39   0x0000000002db2d15: pop     rbp
40   0x0000000002db2d16: test    dword ptr [0ea0000h],eax ## 两数与操作，ZF反映结果是否为零；SF反映AL的最高位是否为0
41                                                 ;   {poll_return}
42   0x0000000002db2d1c: ret ## 将栈顶字单元保存的偏移地址作为下一条指令的偏移地址
43   0x0000000002db2d1d: hlt ## HLT是CPU指令CPU遇到该指令停止执行命令
44   0x0000000002db2d1e: hlt 
45   0x0000000002db2d1f: hlt
46	[Exception Handler]
```
 有关汇编的关键字：

1. rdx  64位通用寄存器
1. dword 双字 32字节
1. qword  64个字节
1. rsp 指向64位栈顶

有个纳闷的，代码中是两行volatile write，但汇编中只有一个lock指令，**猜测这个lock指令应该是把整个栈给锁**了。


## 关于volatile的实现

有volatile变量修饰的共享变量进行写操作的时候会多第二行汇编代码，通过查IA-32架构软件开发者手册可知，lock前缀的指令在多核处理器下会引发了两件事情。

1. 将当前处理器缓存行的数据会写回到系统内存。
1. 这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效。

处理器为了提高处理速度，不直接和内存进行通讯，而是先将系统内存的数据读到内部缓存（L1,L2或其他）后再进行操作，但操作完之后不知道何时会写到内存，如果对声明了Volatile变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里。

这两件事情在IA-32软件开发者架构手册的第三册的多处理器管理章节（第八章）中有详细阐述。

Lock前缀指令会引起处理器缓存回写到内存。Lock前缀指令导致在执行指令期间，声言处理器的 LOCK# 信号。在多处理器环境中，LOCK# 信号确保在声言该信号期间，处理器可以独占使用任何共享内存。（因为它会锁住总线，导致其他CPU不能访问总线，不能访问总线就意味着不能访问系统内存），但是在最近的处理器里，LOCK＃信号一般不锁总线，而是锁缓存，毕竟锁总线开销比较大。在8.1.4章节有详细说明锁定操作对处理器缓存的影响，对于Intel486和Pentium处理器，在锁操作时，总是在总线上声言LOCK#信号。但在P6和最近的处理器中，如果访问的内存区域已经缓存在处理器内部，则不会声言LOCK#信号。相反地，它会锁定这块内存区域的缓存并回写到内存，并使用缓存一致性机制来确保修改的原子性，此操作被称为“缓存锁定”，缓存一致性机制会阻止同时修改被两个以上处理器缓存的内存区域数据。

一个处理器的缓存回写到内存会导致其他处理器的缓存无效。IA-32处理器和Intel 64处理器使用MESI（修改，独占，共享，无效）控制协议去维护内部缓存和其他处理器缓存的一致性。在多核处理器系统中进行操作的时候，IA-32 和Intel 64处理器能嗅探其他处理器访问系统内存和它们的内部缓存。它们使用嗅探技术保证它的内部缓存，系统内存和其他处理器的缓存的数据在总线上保持一致。例如在Pentium和P6 family处理器中，如果通过嗅探一个处理器来检测其他处理器打算写内存地址，而这个地址当前处理共享状态，那么正在嗅探的处理器将无效它的缓存行，在下次访问相同内存地址时，强制执行缓存行填充。

文字介绍 copy  from  [深入分析Volatile的实现原理](http://ifeve.com/volatile/ '聊聊并发（一）深入分析Volatile的实现原理')




