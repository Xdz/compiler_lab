# TableGen的大致结构

tablegen分为前端和后端,前端解析tablegen使用的语言,后端生成需要的机器代码

tablegen由record,definition,class,multiclass几个基本概念组成

tablegen接收 .td 文件并生成C++代码,并且有命令行llvm-tblgen可以执行