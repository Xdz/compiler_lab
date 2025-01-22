# 代码生成器(Target-Independent Code Generator)的6个组成部分

Abstract target description,用于代表目标机器的抽象.

一些类,用于表示生成代码的抽象.

一些类与算法,用于表示 object 文件级别的抽象.

Target-Independent 算法,用于产生代码的各个阶段,比如寄存器分配,调度,帧栈表示等.

描述特定抽象机器目标的组件.

LLVM JIT

## 代码生成的6个步骤

选择指令

指令排序

代码优化

寄存器分配

代码插入

机器码优化

## 目标机器描述的类

目标机器描述的类提供了独立于具体的target machine的抽象,并且以抽象类和虚方法的C++的代码形式实现.

### TargetMachine class

这个类提供了很多虚方法来访问特定的架构,均为getxxxInfo的形式.

### DataLayout class

这个类不可以派生,它用以描述大小端,对齐,数据类型等信息.

### TargetLowering class

此类被指令选择器所调用,用以描述如何从LLVM IR变换为 SelectionDAG 操作.

此外,此类还指示目标机器支持什么操作,有什么类型的寄存器,返回类型,移位操作的方式等.

### TargetRegisterInfo class

此类用于描述目标机器的寄存器之间的交互.

每个寄存器都有一个TargetRegisterDesc 项,此项代表寄存器的文本名称,每个寄存器还有个别名,用以指示两个寄存器之间是否重叠.

### TargetInstrInfo class
TargetInstrInfo
类用于描述目标支持的机器指令.
描述定义了诸如操作码的助记符、操作数的数量、隐式寄存器使用和定义的列表、指令是否具有某些与目标无关的属性
(访问内存、可交换等)以及是否持有任何特定于目标的标志等内容.

### TargetFrameLowering class

TargetFrameLowering 类用于提供有关目标的堆栈帧布局的信息.
它保存堆栈增长的方向、进入每个函数时已知的堆栈对齐以及到局部区域的偏移量.

### TargetSubtarget class

TargetSubtarget 类用于提供有关某架构的的特定芯片组的信息.比如子目标通知代码生成支持哪些指令、指令延迟和指令执行行程;
例如，使用哪些处理单元，以什么顺序使用，以及使用多长时间.

### TargetJITInfo class

TargetJITInfo 类暴露实时代码生成器使用的抽象接口，以执行特定于目标的活动，例如打桩.如果TargetMachine支持JIT
代码生成，它应该通过getJITInfo方法提供这些对象.

## 描述机器码的类

### MachineInstr class

此类用于表示目标机器指令.

所有的目标机器指令均在 *InstrInfo.td 文件中定义,然后生成操作码.

这个类并不保存指令语义.

#### MachineInstrBuilder.h 函数的使用.

机器指令由 BuildMI 函数生成,位于 include/llvm/CodeGen/MachineInstrBuilder.h 文件中.

文档中的示例:

```c++
// Create a 'DestReg = mov 42' (rendered in X86 assembly as 'mov DestReg, 42')
// instruction and insert it at the end of the given MachineBasicBlock.
const TargetInstrInfo &TII = ...
MachineBasicBlock &MBB = ...
DebugLoc DL;
MachineInstr *MI = BuildMI(MBB, DL, TII.get(X86::MOV32ri), DestReg).addImm(42);

// Create the same instr, but insert it before a specified iterator point.
MachineBasicBlock::iterator MBBI = ...
BuildMI(MBB, MBBI, DL, TII.get(X86::MOV32ri), DestReg).addImm(42);

// Create a 'cmp Reg, 0' instruction, no destination reg.
MI = BuildMI(MBB, DL, TII.get(X86::CMP32ri8)).addReg(Reg).addImm(42);

// Create an 'sahf' instruction which takes no operands and stores nothing.
MI = BuildMI(MBB, DL, TII.get(X86::SAHF));

// Create a self looping branch instruction.
BuildMI(MBB, DL, TII.get(X86::JNE)).addMBB(&MBB);
```

#### 处理特定的寄存器

由于调用约定等的存在,必须将虚拟寄存器分配到特定的物理寄存器中.

物理寄存器的生命周期不因该跨过基本块,因此需要跨过基本块的值需要储存在虚拟寄存器中.

#### SSA形式的机器码

从LLVM IR 到寄存器分配阶段,机器码移植时SSA形式的,分配寄存器之后,由于虚拟寄存器不再存在,因此机器码也不再是SSA形式.

### MachineBlock class

此类包含LLVM 的基本块,包含多个 MachineInstr的实例 可以映射到多个机器码基本块.

可以使用getBasicBlock方法来返回基LLVM基本块.

### MachineFunction class

MachineFunction类包含一个机器基本块的列表(即MachineBasicBlock实例).
它与指令选择器的LLVM函数输入一一对应.除了一个基本块列表，MachineFunction还包含一个MachineConstantPool
，一个MachineFrameInfo，一个MachineFunctionInfo和一个MachineRegisterInfo.

### MachineInstr Bundles

机器码指令可以被捆绑在一起作为一个整体来对VLIW以及不可分割的指令进行建模.

可以嵌套 MI Bundles.

## "MC" 层

MC层用于处理机器码层面的代码,没有常量池和跳转表等结构.

### MCStreamer API

这是一个汇编API并且不负责具体实现.

这个API的方法与汇编级别的指令是对应的.

这个API有两种主流实现,第一种是 MCSAsmStreamer,用于输出 .s 文件,另一种是 MCObjectStreamer,用于输出 .o 文件.

对于特定的目标指令, MCStreamer 有一个 MCTargetStreamer 实例,并且实现了类似于 MCStreamer的两个方法, asm streamer 用于打印汇编, object streamer 用于实现汇编逻辑.

只有在初始化时调用 TargetRegistry::RegisterAsmStreamer 和 TargetRegistry::RegisterMCObjectStreamer,传递分配对应目标流的回调函数给它俩,然后将这两个函数分别传输给 createAsmStreamer 和 object streamer 构造函数.

### MCContext class

此类不能被继承,且此类负责管理所有MC层的数据.

### MCSymbol class

此类表示汇编中的符号,符号分为临时符号和普通符号,临时符号在汇编中被生成但是在 object file 中被丢弃.

### MCSection class

表示 object file 的某个 section.是对于某个 object file 的类子类,负责是实现(如 MSSectionELF).

### MCInst class

此类表示指令但是与目标机器无关.这个类包含一个特定的操作码和一组操作数,操作数分为三类,第一类是立即数,第二类是寄存器,第三类是表达式.

MCInst是汇编器,反汇编器,指令编码器和打印指令时所使用的类.

# 代码生成的算法

## 指令选择

指令选择是指从LLVM代码翻译为目标机器代码的过程.LLVM采用了基于 SelectionDAG 的指令选择器.但目前正在转向GlobalISel.

这个指令选择器部分是由 .td 文件生成的,但是需要自定义部分C++代码.

### SelectionDAG

SelectionDAG是一个有向无环图,其节点时SDNode的实例,SDNode包含操作码和相应的操作数.

具体的节点类型在 include/llvm/CodeGen/ISDOpcodes.h 中.

边由 SDValue 的实例表示,并且每个值都有对应的机器值的类型.

SelectionDAG 包含控制流和数据流两种类型的值.数据值具有整数和浮点数两种类型,控制值的类型为 MVT::Other.控制值和节点之间构成链条.

SelectionDAG中有"Entry"与"Root"节点的设计."Entry"节点是操作码为 ISD::EntryToken 的节点, "Root"节点是最后一个具有副作用的节点.

SelectionDAG 具有合法与非法的概念,只有值与类型都合法才算合法,SelectionDAG有合法化的算法阶段.

### SelectionDAG工作的几个阶段

#### 初始化 SelectionDAG

使用 SelectionDAGBuilder 类进行初始化.

#### 合法化数据

通过数据的扩展和拆分来使数据合法化,满足目标架构的要求.

#### DAG优化

每合法化一遍就跑一遍优化

#### 指令选择阶段

将LLVM代码利用指令匹配由LLVM IR 中的指令生成目标架构的指令.

此阶段使用TableGen由 .td 文件生成所需要的代码.

TableGen可以处理加法交换律,可以在编译期间进行检查,可以处理更高级的语义.

可以编写C++代码来自定义匹配规则.

#### 指令调度阶段

此阶段将指令由图结构变换为线性的顺序结构.

## 值的生命周期分析

当两个值需要同一时间占用同一寄存器时,则发生冲突,需要溢出一个虚拟寄存器.

## 寄存器分配

寄存器分配是将无限多的虚拟寄存器映射到有限的物理寄存器中的过程.

LLVM中物理寄存器编号为1~1023,对于特定架构的寄存器编号信息可以从 GenRegisterNames.inc 文件中获取.

LLVM中寄存器被分类,同类寄存器中的每个寄存器都是是等价的.寄存器类由 TargetRegisterClass 描述.

MachineRegisterInfo::createVirtualRegister() 方法可以创建新的虚拟寄存器

IndexedMap\<Foo, VirtReg2IndexFunctor\> 保存每个虚拟寄存器的信息

TargetRegisterInfo::index2VirtReg() 函数查找虚拟寄存器号

MachineOperand 下定义了许多寄存器有关的方法.

寄存器会被着色,且根据调用约定等会有隐式分配的寄存器,隐式分配的寄存器与指令有关而与 .td 文件中的寄存器个数无关.

寄存器分配分为直接和间接,直接映射是直接将虚拟寄存器映射到物理寄存器中的方式,而间接映射是插入 load 和 store 指令.

需要考虑 三地址指令和 两地址指令之间的转换

## 代码生成

由LLVM IR 的抽象转变为 MC 层的抽象.

1. 创建 AsmPrinter 的子类.

2. 创建 Instruction Printer.

3. 实现 MachineInstr 到 MCInstr 的代码,将跳转表,常量池等变为标签.

4. 实现 MCCodeEmitter 的子类,将MCInstr变为字节码或者重定位符号.
