# llvm 后端语法

@开头是全局变量

%开头是局部变量

; 分号是注释

匿名或者未命名的变量从0开始计数,比如 %0 %1 等以此类推

! 是元数据

## 保留字和类型

比如 add,bitcode ret等为保留字

i32等为类型

## 字符串

以双引号 " 识别一个字符串,以反斜杠转义 \ ,且反斜杠后面跟数字表示某个字母对应的数

## 链接类型

每个全局变量或者函数均指向一个内存地址,数组指向数组基地址,函数指向函数入口.

每一个全局变量或者函数定义均有一个链接类型.

### 具体链接类型

#### private

具有 private 链接类型的全局变量只可被当前module的对象访问,并且不会出现在 .o 文件的符号表中.链接的代码可能被重命名以防止名字冲突.

#### internal

对应于C语言中的 static 关键字,与private类似但是会在局部符号表(local table)中.

#### available_externally

这种链接类型只允许在定义中使用而不允许在声明中使用并且不会被归到任何一个module中.

从链接器的角度看, available_externally 相当于外部声明,从而可以提供优化.

具有此链接类型的全局变量可以被任意的优化甚至丢弃.

#### link_once

拥有 link_once 类型的全局变量将会在链接时被合并到其他同名变量中.如果未被引用则允许被丢弃.

拥有此种类型的全局变量通常用于实现模板和内联函数.

#### weak

与 link_once 类似但是未引用的时候不会被丢弃,与C中weak关键字对应.

#### common

与weak类似,但是具有C语言中的试探性定义(tentative definition),比如 `int X;`.未被引用时可能不会被删除,且必须初始化为0.

#### appending

appending 只能用于指向数组的指针.当两个具有 appending 链接类型的全局变量被合并时,全局数组被附加在一起,这相当于 .o 文件合并时,
将同名的section附加在一起.但是只能作用于llvm变量.

#### extern_weak

该链接的语义遵循ELF对象文件模型:符号在链接之前是weak，如果没有链接，则符号变为null，而不是未定义的引用。

#### linkonce_odr 与 weak_odr

"odr"代表C++中的单一定义规则,可用于内联函数和常量折叠.如果有多个定义则执行的时候随机选择一个定义.

对于链接器与无 odr 后缀版本相同.

#### external

如果不是上面任意一种链接类型,则为 external 类型,此种链接类型可用于外部符号表的符号引用的问题.

## 调用约定

"cc -<n>" 代表自定义的调用约定.从64开始计数.

## 可见性

### default

default 类型可见性在ELF中代表声明对于其他的module可见.在动态链接库中代表可以被重载.

### hidden

在同一个共享对象中的两个具有 "hidden visibility" 的对象,其声明会引用同一个对象.

具有 "hidden" 可见性的对象符号不会被放入动态符号表中,因此对于其他module不可见.

### protected

该符号不可被其他module重载.

## DLL storage 类型

具有 internal 和 private 链接类型的符号不可以有 DLL storage 类型.

### dllimport

dllimport 使编译器通过一个全局指针来引用一个由dll提供的指针

### dllexport

这个类型提供了一个dll接口因此编译器,链接器和汇编器均不能删除它

## 线程本地的存储模型

该模型与 ELF 的 TLS 模型对应.

其余略过.

## 运行时取代说明符(Runtime Preemption Specifiers)

### dso_preemptable

指示函数或变量可以在运行时由链接单元外部的符号替换。

### dso_local

编译器可能会假设标记为dso_local的函数或变量将解析为同一链接单元中的符号。即使定义不在此编译单元内，也会生成直接访问。

## structure 类型

略过

## 非整形指针类型

略过

## 全局变量

全局变量是在编译的时候分配的内存而不是在运行时分配的.

所有全局变量必须初始化,除非是声明在其他翻译单元中.

全局变量有可选的链接类型,DLL存储类型,全局属性等(见上文).

全局变量的定义和声明可以有一个显式声明的section和一个显示的对齐方式.当变量的声明和定义不匹配时则为UB.

全局变量可被标记为常量,即使其并非常量,在并非常量的情况下需要由高级语言保证常量有关的优化在全局变量的定义所在的翻译单元中无效.

全局变量的SSA值是一个指针值,指向内存中的某块内容.

全局变量可用 unnamed_addr 标记,用此符号标记的全局变量可与有相同初始化项的常量合并,结果是一个有有效地址的常量.

如果给出了local_unnamed_addr属性，则已知该地址在模块中不重要.

可以声明全局变量驻留在特定于目标的编号地址空间中.
对于支持它们的目标，地址空间可能会影响如何执行优化和/或使用什么目标指令来访问变量.默认地址空间为零.
地址空间限定符必须位于任何其他属性之前.

llvm允许显示指定全局变量的代码模型.如果目标架构支持,则翻译单元中类型会被代码中发出的全局变量覆盖.
目前(llvm20)可能取值为:tiny small kernel medium large

全局初始化器优化全局变量的一个前提是在全局初始化器初始化变量之前其值不会被修改,可以使用 externally_initialized 来改变这种假设.

全局变量可以被指定对齐,如果不指定,则会按照方便处理的方式被对齐,如果全局变量有指定的section,则不会被过度对齐(over-align).
最大对齐为 32.

对于全局变量声明，以及可能在链接时被替换的定义(linkonce, weak, extern_weak和常见链接类型)
它解析的定义的分配大小和对齐必须大于或等于声明或可替换定义的大小和对齐，否则行为未定义.

全局变量不可包含可伸缩向量,因为其大小未知.

### 语法

```asm
@<GlobalVarName> = [Linkage] [PreemptionSpecifier] [Visibility]
[DLLStorageClass] [ThreadLocal]
[(unnamed_addr|local_unnamed_addr)] [AddrSpace]
[ExternallyInitialized]
<global | constant> <Type> [<InitializerConstant>]
[, section "name"] [, partition "name"]
[, comdat [($name)]] [, align <Alignment>]
[, code_model "model"]
[, no_sanitize_address] [, no_sanitize_hwaddress]
[, sanitize_address_dyninit] [, sanitize_memtag]
(, !name !N)*
```

## 函数

llvm函数由define关键词开头,后面跟一堆可选项,然后是由花括号囊括的基本块.

基本块会构成CFG,即控制流图(control flow graph).

基本块由标签构成入口,如果没有显示指定标签名称则给予一个数字编号作为名称.

在llvm中函数可以被显式的指定放到某个section中.

可以为函数指定显式对齐。如果不存在，或者如果对齐设置为零，则函数的对齐由目标设置为其认为方便的任何值.
如果指定了显式对齐方式，则强制该函数至少有那么多对齐方式.所有对齐必须是2的幂.

如果给出了unnamed_addr属性，则已知该地址不重要，并且可以合并两个相同的函数.

如果给出了local_unnamed_addr属性，则已知该地址在模块中不重要.

如果没有给出显式地址空间，它将默认为datalayout字符串中的程序地址空间.

### 函数定义语法

```asm
define [linkage] [PreemptionSpecifier] [visibility] [DLLStorageClass]
       [cconv] [ret attrs]
       <ResultType> @<FunctionName> ([argument list])
       [(unnamed_addr|local_unnamed_addr)] [AddrSpace] [fn Attrs]
       [section "name"] [partition "name"] [comdat [($name)]] [align N]
       [gc] [prefix Constant] [prologue Constant] [personality Constant]
       (!name !N)* { ... }
```

其中参数语法为:

```asm
<type> [parameter Attrs] [name]
```

### 函数声明语法

```asm
declare [linkage] [visibility] [DLLStorageClass]
        [cconv] [ret attrs]
        <ResultType> @<FunctionName> ([argument list])
        [(unnamed_addr|local_unnamed_addr)] [align N] [gc]
        [prefix Constant] [prologue Constant]
```

## alias 别名

别名有一个名称和一个别名，别名可以是全局值或常量表达式.

### 语法

```asm
@<Name> = [Linkage] [PreemptionSpecifier] [Visibility] [DLLStorageClass] [ThreadLocal] [(unnamed_addr|local_unnamed_addr)] alias <AliaseeTy>, <AliaseeTy>* @<Aliasee>
          [, partition "name"]
```

别名不可被用于重定位,因此其表达式必须可在生成汇编时被计算.

其余略.

## IFuncs

与别名类似,不是任何新的函数,只是在运行时通过调用函数解析器来解析的所要用到的符号.

## comdats

用于支持object文件中COMDAT节的功能.具有comdat关键字的数据可以被整个丢弃,但不能只丢弃一部分.

用于链接器维护数据以及选择链接对象.

## 参数属性

函数的参数属性是函数内部属性,因此不同参数属性的函数可以具有相同的函数属性.

llvm已定义的参数属性均略过.

## attribute group

属性组,在LLVM IR 中,属性被分成组保存,然后被函数和全局变量所引用.

## GC

略过.

## prefix data

用于将前端提供的,与语言有关的额外信息与函数关联起来.

前缀数据放置于函数体的开头.

## prologue data

在函数体之前插入的数据,需要可被目标机器解读执行

## 函数属性

用于提供额外的关于函数的信息.

具体可选类型略过.

## 全局变量属性

全局变量属性用于提供关于全局变量的附加信息.

全局变量属性被分到单个属性组中.

具体可选项略过.

## data layout

数据在内存中的摆放方式

```asm
target datalayout = "layout specification"
```

可选项略过.

## Target Triple

```asm
target triple = "ARCHITECTURE-VENDOR-OPERATING_SYSTEM-ENVIRONMENT"
```

## 类型系统和常量以及内建函数略过

# 编写后端的大致流程

1. 创建 TargetMachine 类的子类来描述目标机器的字符,可以从复制LLVM现有的文件开始.
2. 描述寄存器集.使用TableGen来从RegisterInfo.td文件生成描述 寄存器定义 寄存器别名
   寄存器类 的代码.并且编写 TargetRegisterInfo 类的子类来描述寄存器堆的分配以及寄存器之间的相互作用.
3. 描述目标指令集.使用 TableGen 从 TargetInstrFormats.td 和 TargetInstrInfo.td
   中生成代码.并且编写 TargetInstrInfo 类的子类来表达机器指令.
4. 描述从 LLVM IR 的DAG(有向无环图)表示到目标机器指令的表示.使用 TableGen 根据
   TargetInstrInfo.td 的信息来匹配和选择指令.编写 XXXISelDAGToDAG.cpp 其中xxx代表目标架构.并且编写 XXXISelLowering.cpp 来移除或者替换 SelectionDAG 中不支持的操作或者数据类型.
5. 为 TargetInstrInfo.td 添加汇编字符串来将 LLVM IR 的形式转变为 GAS 的形式.并且编写 TargetAsmInfo 和 AsmPrinter 的子类来将LLVM代码转变为汇编.
6. 编写 TargetSubtarget 类的子类从而可以使用命令行选项 -mcpu= 和 -mattr= 
7. 可以选择支持JIT.

