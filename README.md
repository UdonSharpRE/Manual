# 入门

很显然，UdonSharp和大多数编程语言一样，VRChat做了类似的事情，但是当然，就像我一直跟别人说的那样：VRChat有一个垃圾的团队在写代码，他们从来没做对任何一件事情，更有个可怕且顽固的管理层。

为了简称，以下开始 U# 代表 UdonSharp。

## 前言

UdonSharp尽管和别的编程语言大体上一样，但是它垃圾的地方也显而易见。

### 糟糕的编译器

UdonAssembly 几乎是 1 对 1 直接编译生成的，它将调用的函数，变量赋值等操作直接编译为等效的 UdonAssembly 指令，毫无优化可言（根本就是没有）。

让我们打个比方，下面有一段看着非常糟糕的代码
```cs
bool somebool = false;
bool IsSafe()
{
  if(somebool)
  {
    return true;
  }
  else{
    return false;
  }
}
```
对于这样的代码，U#的编译器不会进行任何优化，直接编译，结果如下：
```
.data_start


    __refl_const_intnl_udonTypeID: %SystemInt64, null
    __refl_const_intnl_udonTypeName: %SystemString, null
    somebool: %SystemBoolean, null
    __1_const_intnl_SystemBoolean: %SystemBoolean, null
    __0_const_intnl_SystemBoolean: %SystemBoolean, null
    __0_const_intnl_SystemUInt32: %SystemUInt32, null
    __0_intnl_returnValSymbol_Boolean: %SystemBoolean, null
    __0_intnl_returnTarget_UInt32: %SystemUInt32, null

.data_end

        
         #  using UdonSharp;
        
         #  using UnityEngine;
        
         #  using VRC.SDKBase;
        
         #  using VRC.Udon;
        
         #  public class GameObject3 : UdonSharpBehaviour
.code_start
        
         #  	bool somebool = false;
        
         #  void Start()
    .export _start
        
    _start:
        
        PUSH, __0_const_intnl_SystemUInt32
        
         #  {
        PUSH, __0_intnl_returnTarget_UInt32 # Function epilogue
        COPY
        JUMP_INDIRECT, __0_intnl_returnTarget_UInt32
        
        
         #  	bool IsSafe()
    IsSafe:
        
        PUSH, __0_const_intnl_SystemUInt32
        
         #  	{
        
         #  		if(somebool)
        PUSH, somebool
        JUMP_IF_FALSE, 0x00000064
        
         #  		{
        
         #  			return true;
        PUSH, __0_const_intnl_SystemBoolean
        PUSH, __0_intnl_returnValSymbol_Boolean
        COPY
        PUSH, __0_intnl_returnTarget_UInt32 # Explicit return sequence
        COPY
        JUMP_INDIRECT, __0_intnl_returnTarget_UInt32
        JUMP, 0x0000008C
        
         #  		else{
        
         #  			return false;
        PUSH, __1_const_intnl_SystemBoolean
        PUSH, __0_intnl_returnValSymbol_Boolean
        COPY
        PUSH, __0_intnl_returnTarget_UInt32 # Explicit return sequence
        COPY
        JUMP_INDIRECT, __0_intnl_returnTarget_UInt32
        PUSH, __0_intnl_returnTarget_UInt32 # Function epilogue
        COPY
        JUMP_INDIRECT, __0_intnl_returnTarget_UInt32
        
.code_end
```
官方反编译：
```
0x00000000: PUSH, 0x00000005
0x00000008: PUSH, 0x00000007
0x00000010: COPY
0x00000014: JUMP_INDIRECT, __0_intnl_returnTarget_UInt32
0x0000001C: PUSH, 0x00000005
0x00000024: PUSH, 0x00000002
0x0000002C: JUMP_IF_FALSE, 0x00000064
0x00000034: PUSH, 0x00000004
0x0000003C: PUSH, 0x00000006
0x00000044: COPY
0x00000048: PUSH, 0x00000007
0x00000050: COPY
0x00000054: JUMP_INDIRECT, __0_intnl_returnTarget_UInt32
0x0000005C: JUMP, 0x0000008C
0x00000064: PUSH, 0x00000003
0x0000006C: PUSH, 0x00000006
0x00000074: COPY
0x00000078: PUSH, 0x00000007
0x00000080: COPY
0x00000084: JUMP_INDIRECT, __0_intnl_returnTarget_UInt32
0x0000008C: PUSH, 0x00000007
0x00000094: COPY
0x00000098: JUMP_INDIRECT, __0_intnl_returnTarget_UInt32
```
官方甚至贴心的帮你标注好源码对照...

### 大量重复的代理变量

在你看不见的地方(也可能看得见)，创建了大量的 `重复` 变量，这些变量主要用于代替赋值，我认为这是他们的编译器设计缺陷的缘故。

譬如：

这里有一段我从别的地方转储出的反汇编
```
0x0000000000000000  PUSH 0x000000000000000A(Obj[UnityEngine.GameObject[]])
0x0000000000000008  PUSH 0x0000000000000009(__instance_0[UnityEngine.GameObject[]])
0x0000000000000010  COPY
0x0000000000000014  PUSH 0x0000000000000009(__instance_0[UnityEngine.GameObject[]])
0x000000000000001C  PUSH 0x0000000000000007(__end_0[System.Int32])
0x0000000000000024  EXTERN "UnityEngineGameObjectArray.__get_Length__SystemInt32"
```

你说说，这为什么要创建一个新变量 `__instance_0` 再把`Obj`给赋值过去？直接用`Obj`不行吗？

然而，这只是一小段代码，想想看这样的东西成千上万的存在于你的代码生成的UdonAssembly中，多可怕啊。

我认为，这是一个可以证明 U# 是一个只有`栈(Stack)`而没有`堆(Heap)`的虚拟机语言

### 未规范化的调用约定

U#的调用非常随便，像是...上一秒还在和你说话的人下一秒直接消失了。

所有的函数调用都是把变量推入堆栈然后直接跳走的，我相信设计UdonAssembly的人可能多少受到了一些x86_64指令集的启发，但是无论如何，他们没有 `CALL` 指令

下面我们来看一段代码：
```
.func__start_0x3B8
0x00000000000003B8  PUSH 0x000000000000006F(__const_SystemUInt32_0[System.UInt32])
0x00000000000003C0  PUSH 0x000000000000007F(__gintnl_SystemUInt32_0[System.UInt32])
0x00000000000003C8  PUSH 0x0000000000000081(__const_SystemBoolean_0[System.Boolean])
0x00000000000003D0  PUSH 0x0000000000000080(__0_active__param[System.Boolean])
0x00000000000003D8  COPY
0x00000000000003DC  JMP 0x0000000000007C24
0x00000000000003E4  PUSH 0x0000000000000081(__const_SystemBoolean_0[System.Boolean])
0x00000000000003EC  PUSH 0x000000000000000B(_isBusy[System.Boolean])
0x00000000000003F4  COPY
```

`JMP`这个指令，除了用作带`else`的`if`的分支跳转以外，还被用作`for`和`while`的循环执行。

...当然，还用作调用函数！

我们可以清楚的看到，在`0x3DC`处，直接跳走到`0x7C24`，您认为函数会有这么大吗？

很显然，这是一个函数调用，让我们来看看 `0x7C24` 处

```
0x0000000000007C1C  PUSH 0x000000000000006F(__const_SystemUInt32_0[System.UInt32])
0x0000000000007C24  PUSH 0x0000000000000080(__0_active__param[System.Boolean])
0x0000000000007C2C  JNE 0x0000000000007D60
0x0000000000007C34  PUSH 0x00000000000000AA(__const_SystemInt32_1[System.Int32])
0x0000000000007C3C  PUSH 0x000000000000049E(__lcl_i_SystemInt32_11[System.Int32])
0x0000000000007C44  COPY
0x0000000000007C48  PUSH 0x000000000000027D(__gintnl_SystemUInt32_272[System.UInt32])
0x0000000000007C50  PUSH 0x000000000000027E(__gintnl_SystemUInt32_273[System.UInt32])
```

根据指令的特征来看，这是一个标准的U#函数头，`PUSH __const_SystemUInt32_0`可以用作判断依据。

### 没有寄存器的后果

在 `x86_64` 下，调用的返回值一般都会存放在 `eax(rax)` 下，但是由于 U# 不存在寄存器，因此，他们只能被迫将返回值使用一个新的临时变量保存。*[1]*

这意味着任何可以在游戏内调查U#对象的人，都可以轻易的得知上一次调用函数的返回值是什么。

`[1]` 不总是会产生临时变量用于保存返回值，它取决于 U# 的编译器版本。

你现在知道他们为什么要出 UdonSharp 2.0 了吗？这完全就是一个屎山！

### 沉重的指令集

在 U# 中，每个指令都是 4字节 长的，并且只有两种类型，`携带数据的`和`不携带数据的`。

- `携带数据的` 指该指令需要后面4个字节的堆地址用于输入，譬如 `PUSH 0xDEADBEEF`
- `不携带数据的` 指一些简单指令集或者基于栈的指令集，基于栈的指令集例如`COPY`，简单指令集例如`NOP` 

并且 U# 完全不合 x86_64 那样具有多样化的指令，几乎所有的 U# 运算操作都是由调用外部函数来完成的，你可以参阅[这里](https://github.com/UdonSharpRE/UdonSharpDecompiler/blob/master/UdonSharpDecompiler/Global.h#L242) 和 [这里](https://github.com/UdonSharpRE/UdonSharpDecompiler/blob/master/UdonSharpDecompiler/AssemblyLoader.h#L158) 来查看

这基本上意味着每进行一次简单的运算都会产生一次调用的开销，指令集不仅缺少多样化，并且低效。

在他们之前提到的 U# 2.0 计划中，他们似乎提到了最后将编译为WebAssembly，我猜测他们可能正在复制粘贴LLVM

## 总结

综合上面的种种说明，我们现在知道了，U# 一种简单，且不高效的语言。

在你吐槽之前，先等等，我知道这里是手册，而不是专门批判U#的地方。但是恰恰相反，正是因为U#的种种缺陷和其简单性，其对逆向工程非常友好，而且你几乎可以得到源代码级别的U#汇编代码。

# 目录

0. [理解前缀](https://github.com/UdonSharpRE/Manual/blob/main/00.Understand.Prefixes.md)
1. [读懂 UdonSharp 指令集](https://github.com/UdonSharpRE/Manual/blob/main/00.Understand.UdonAsm.md)
