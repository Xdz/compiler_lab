# 总体架构

编译器前端 -> llvm-ir中间表示 -> 编译器后端

# LLVM IR 高级结构

LLVM IR 代码由 Module 构成,每个Module是一个翻译单元.

每个Module由函数,全局变量,符号表项组成,还可能有 LLVM 链接器以解决定义和前置声明的问题.

## LLVM IR 示例代码
```asm
; Declare the string constant as a global constant.
@.str = private unnamed_addr constant [13 x i8] c"hello world\0A\00"

; External declaration of the puts function
declare i32 @puts(ptr nocapture) nounwind

; Definition of main function
define i32 @main() {
  ; Call puts function to write out the string to stdout.
  call i32 @puts(ptr @.str)
  ret i32 0
}

; Named metadata
!0 = !{i32 42, null, !"string"}
!foo = !{!0}
```

.str 是全局变量, puts 是函数声明, main 是函数定义, foo是元数据, i32 是类型, nounwind 是属性, ret 是指令


# 其他

llvm所有内存对象均通过指针访问
llvm使用C++ STL库
除了标准库以外,llvm还提供其他的数据结构来辅助存数据,基本上都在llvm/ADT目录下
llvm提供了其定义的新的类型,均有Type后缀
llvm的module类表示整个llvm块
value类表示一个拥有类型的值
user类表示引用value的llvm节点
instruction类是所有指令类的基类
constant类是所有常量类的基类
GlobalValue类是constant类的一个子类
function类表示llvm中表示一个高级语言中的函数
GlobalVariable类是表示全局变量的类
BasicBlock类是表示编译原理中基本块的类,有类型且被分支指令引用
Argument类是表示函数中参数的类