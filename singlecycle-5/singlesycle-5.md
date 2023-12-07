# 单周期CPU-5条指令



## 一、实验内容

完成一个5条指令的单周期CPU设计方案并进行验证

miniCPU顶层接口信号的描述如下：

| 名称            | 宽度 | 方向 | 描述                            |
| --------------- | ---- | ---- | ------------------------------- |
| clk             | 1    | I    | 时钟信号，来自cik_pll的输出时钟 |
| resetn          | 1    | I    | 复位信号，低电平同步复位        |
| inst_sram_we    | 1    | O    | RAM写使能信号，高电平有效       |
| inst_sram_addr  | 32   | O    | RAM读写地址，字节寻址           |
| inst_sram_wdata | 32   | O    | RAM写数据                       |
| inst_sram_rdata | 32   | I    | RAM读数据                       |
| data_sram_we    | 1    | O    | RAM写使能信号，高电平有效       |
| data_sram_addr  | 32   | O    | RAM读写地址，字节寻址           |
| data_sram_wdata | 32   | O    | RAM写数据                       |
| data_sram_rdata | 32   | I    | RAM读数据                       |



## 二、设计详解

### 1. 指令详解

#### :one: add.w(add word)

add.w  rd, rj, rk	GR[rd] = GR[rj] + GR[rk]

00000000000100000 rk[14: 10] ri[9: 5] rd[4: 0]

#### :two: addi.w(add immediate word)

addi.w  rd, rj, si12	GR[rd] = GR[rj] + sext32(si12)

0000001010 si12[21: 10] rj[9: 5] rd[4: 0]

#### :three: ld.w(load word signed)

ld.w  rd, rj, si12	GR[rd] = MEM[GR[rj] + sext32(si12)] [31: 0]

0010100010 si12[21: 10] rj[9: 5] rd[4: 0]

#### :four: st.w(store word)

st.w  rd, rj, si12	MEM[GR[rj] + sext32(si12)] [31: 0] = GR[rd] [31: 0]

0010100110 si12[21: 10] rj[9: 5] rd[4: 0]

#### :five: bne(branch on not equal)

bne  rj, rd, offs16	if(GR[rj] != GR[rd]) PC = PC + sext32({off16, 2'b0})

010111 offs[15: 0] [25: 10] rj[9: 5] rd[4: 0]

### 2. 单周期设计

<img src="assets\singlecycle5.png" alt="Alt" style="zoom:80%;" />



## 三、工程详解

### 1. 使用tcl创建工程

点击`Tcl console`，cd到tcl文件所在目录，输入`source ./creat_project.tcl`

### 2. 指令判断

将指令截取为31: 26、25: 22、21: 20、19: 15，送入译码器进行译码（为什么译码器不写一个可变位宽的？但是这样确实是十分清晰符合准则）

利用[]进行指令码判断，如`op_31_26_d[6'h00]`

将所有需要的opcode进行&运算，确定指令

### 3. 仿真时记录所有信号

点击左侧“PROGECT MANAGER” → “Settings”，在弹出的设置界面中选择“Project Settings” → “Simulation”

在弹出的界面右部选择“Simulation”标签，找到`“xsim.simulate.log_all_signals”`选项并勾选，点击OK保存设置

### 4. 关于仿真顶层

因为整个SoC_Mini的设计都要实现到FPGA芯片中，所以在进行综合实现的时候，所选择的顶层应该是`soc_mini_top`，而不是minicpu_top

### 5. 关于.coe

在FPGA中，.coe文件是一种常见的文件格式，用于描述**存储器初始化数据**

```verilog
memory_initialization_radix = 16;	//指定初始化数据的进制，这里是16进制
memory_initialization_vector =		
//实际的初始化数据，每行包含一个值，将按照其在存储器中的位置进行排序
00,
FF,
A5,
2B,
...
```

如本实验的coe文件，其中包含斐波那切数列计算程序的每条指令值

注意对工程中的inst_ram进行重新定制，选择对应func的coe文件（已经加载好了）

### 6. 仿真结果

switch赋值为4，结果led为5

![alt](assets\originalnegate.png)

为什么testbench和顶层文件中将switch和led取反再取反？仿真结果取反才是正确的结果，但是不取反最后结果很明显啊直接是正的，所以为什么取反（他这么写一定有他这么写的理由我是菜鸡我要向LA官方学习🤯😢🥺😣😭）

![alt](assets\after.png)