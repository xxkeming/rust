# 2023/10/14

学习rcore book文档和下载相关os学习代码, 了解开发环境配置,操作系统启动流程,并配置好环境.rust riscv编译器,qemu模拟器,并尝试运行,调试rcore os ch1.


# 2023/10/15

重新搭建开发环境,参考rcore book文档从0开始用rust编写os内核.
1. 了解rust不用std库,main函数入口,并实现panic,linker,内核启动的流程.
2. 了解riscv汇编,编写入口函数,通过汇编,跳转到rust
3. 了解ecall系统调用,调用rustsbi打印字符函数
4. makefile编写,并实现通qemu启动内核,并输出字符串,然后关机

# 2023/10/21

继续完善os,学习riscv汇编,中断,异常处理
1. 对打印函数的封装,实现print宏,支持对字符串的封装,了解各个段的地址bss的初始化
2. 了解riscv的各种寄存器,特权模式,中断
3. 熟悉rust内嵌汇编,操作各种寄存器等基本操作

# 2023/10/22

继续完善os,实现处理中断
1. 异常处理,实现trap.S,从汇编代码跳转到rust函数,主要实现是保存当前的执行环境（各种寄存器）,栈的切换(s层触发,可不用切换)
2. 通过修改寄存器stvec设置异常中断的处理函数
3. 在s模式直接用ebreak触发断点异常,在异常函数里并返回,并继续执行.（断点指令压缩的话只有2字节 sepc += 2）
4. 添加时钟中断,调用sbi接口,设置时钟的回调间隔时间处理


# 2023/10/28

继续完善os,实现u模式到s模式的切换,调用处理
1. app应用层环境,和内核环境差不多nostd,nomain等
2. 异常处理,trap.S 主要是栈的切换,判断是u模式触发的,需要切换堆栈
2. 实现应用层的调用write,exit,yield
3. 实现yield函数,任务的切换,主要是修改运行状态,并保存和替换运行上下文
4. 实现通过时钟中断来切换任务

# 2023/10/29

继续完善os,了解地址空间sv39
1. 了解页表,虚拟地址,物理地址转换,页表项查询
2. 修改寄存器stap,设置页表,实现sv39虚拟地址模式,内核地址的重新隐射,elf的解析,映射
3. trap的入口和返回地址的计算,trap_context固定的保存
4. 信号的处理,通过发送信号保存在对应的进程环境,在进程回调结束后,检查当前进程是否有信号要处理（任务的切换，通过保存修改trap_context的参数和返回地址来实现应用层的回调函数）
5. 了解easy-fs,并实现应用程序打包到文件系统里面,应用通过读取文件的方式加载

# 2023/11/05

继续完善os,了解smp
1. 了解多核的启动方式
2. 公共资源加锁,主要是任务队列,内存的申请回收
3. 获取当前任务的方式,保存在tp寄存器,任务调度和系统调用保存的环境增加tp字段,直接执行当前任务的地址
