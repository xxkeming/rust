### risc-v hyper-v
##### 教程地址: https://github.com/arceos-hypervisor/2023-virtualization-campus/blob/master/README.md

### 搭建acreos-hypervisor运行环境
- 地址: https://github.com/arceos-hypervisor/hypercraft
- 编译并运行
- acreos-hypervisor是type 1模式,没有主机操作系统

### 处理器引入虚拟化扩展后的模式
- M模式 -> HS模式 -> U模式
- M模式 -> HS模式 -> VS模式 -> VU模式
- M模式 -> SBI固件
- HS模式 -> 虚拟机管理程序
- VS模式 -> 虚拟机操作系统
- VU模式 -> 虚拟机应用程序
- U模式 -> 主机应用程序
  
### 第1周学习总结-了解虚拟机指令的使用流程,用途和原理
- 基本概念,特权模式的切换,运行环境的载入保存
- cpu的虚拟化,相当于主机的线程
- 内存的虚拟化,页表的映射转换
- io/中断设备的虚拟化

### 第2周尝试实践了解虚拟化的个各个功能过程
- fdt设备树解析 start(hardid, dtb）
- clint中断控制器包含软件中断和时钟中断,时钟中断原理(msip, mtimecmp, mtime)
- uart串口通信(输出文字)
...
