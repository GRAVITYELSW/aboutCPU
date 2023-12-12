# 单周期CPU-20条指令



## 一、实验内容

阅读理解20条指令的单周期CPU的代码，通过仿真波形的调试修复其中的错误，使设计可以通过仿真和上板验证

myCPU顶层接口信号的描述如下：

| 名称              | 宽度 | 方向 | 描述                                                         |
| ----------------- | ---- | ---- | ------------------------------------------------------------ |
| clk               | 1    | I    | 时钟信号，来自clk_pll的输出时钟                              |
| resetn            | 1    | I    | 复位信号，低电平同步复位                                     |
| inst_sram_we      | 1    | O    | RAM写使能信号，高电平有效                                    |
| inst_sram_addr    | 32   | O    | RAM读写地址，字节寻址                                        |
| inst_sram_wdata   | 32   | O    | RAM写数据                                                    |
| inst_sram_rdata   | 32   | I    | RAM读数据                                                    |
| data_sram_we      | 1    | O    | RAM写使能信号，高电平有效                                    |
| data_sram_addr    | 32   | O    | RAM读写地址，字节寻址                                        |
| data_sram_wdata   | 32   | O    | RAM写数据                                                    |
| data_sram_rdata   | 32   | I    | RAM读数据                                                    |
| debug_wb_pc       | 32   | O    | 写回级（多周期最后一级）的PC，需要myCPU里将PC一路传递到写回级 |
| debug_wb_rf_we    | 4    | O    | 写回级写寄存器堆（regfiles）的写使能，为字节写使能           |
| debug_wb_rf_wnum  | 5    | O    | 写回级写regfiles的目的寄存器号                               |
| debug_wb_rf_wdata | 32   | O    | 写回级写regfiles的写数据                                     |

注意为什么iram和dram的we信号不是4位？后面有可能的话改一下

## 二、设计详解

### 1. 指令详解

#### (1) add.w(add word)

add.w  rd, rj, rk	GR[rd] = GR[rj] + GR[rk]

00000000000100000 rk[14: 10] ri[9: 5] rd[4: 0]

#### (2) addi.w(add immediate word)

addi.w  rd, rj, si12	GR[rd] = GR[rj] + sext32(si12)

0000001010 si12[21: 10] rj[9: 5] rd[4: 0]

#### (3) sub.w(subtract word)

sub.w  rd, rj, rk	GR[rd] = GR[rj] - GR[rk]

00000000000100010 rk[14: 10] rj[9: 5] rd[4: 0]

#### (4) slt(set less than)

slt  rd, rj, rk	GR[rd] = GR[rj] < signed GR[rk]

00000000000100100 rk[14: 10] rj[9: 5] rd[4: 0]

#### (5) sltu(set less than unsigned)

sltu  rd, rj, rk	GR[rd] = GR[rj] < unsigned GR[rk]

00000000000100101 rk[14: 10] rj[9: 5] rd[4: 0]

#### (6) slli.w(shift left logic immediate word)

slli.w  rd, rj, ui5	GR[rd] = GR[rj] << ui5

00000000010000 001 ui5 rj[9: 5] rd[4: 0]

#### (7) srli.w(shift right logic immediate word)

srli.w  rd, rj, ui5	GR[rd] = GR[rj] >> logic ui5

00000000010001 001 ui5 rj[9: 5] rd[4: 0]

#### (8) srai.w(shift right arithmetic immediate word)

srai.w  rd, rj, ui5	GR[rd] = GR[rj] >> arith ui5

00000000010010 001 ui5 rj[9: 5] rd[4: 0]

#### (9) and(and)

and rd, rj, rk	GR[rd] = GR[rj] & GR[rk]

00000000000101001 rk[14: 10] rj[9: 5] rd[4: 0]

#### (10) or(or)

or rd, rj, rk	GR[rd] = GR[rj] | GR[rk]

00000000000101010 rk[14: 10] rj[9: 5] rd[4: 0]

#### (11) nor(not or)

nor rd, rj, rk	GR[rd] = ~(GR[rj] | GR[rk])

00000000000101000 rk[14: 10] rj[9: 5] rd[4: 0]

#### (12) xor(exclusive or)

xor rd, rj, rk	GR[rd] = GR[rj] ^ GR[rk]

00000000000101011 rk[14: 10] rj[9: 5] rd[4: 0]

#### (13) lu12i(load upper from bit12 immediate)

lu12i  rd, si20	GR[rd] = {si20, 12'b0}

0001010 si20 rd[4: 0]

#### (14)​ ld.w(load word signed)

ld.w  rd, rj, si12	GR[rd] = MEM[GR[rj] + sext32(si12)] [31: 0]

0010100010 si12[21: 10] rj[9: 5] rd[4: 0]

#### (15) st.w(store word)

st.w  rd, rj, si12	MEM[GR[rj] + sext32(si12)] [31: 0] = GR[rd] [31: 0]

0010100110 si12[21: 10] rj[9: 5] rd[4: 0]

#### (16) bne(branch on not equal)

bne  rj, rd, offs16	if(GR[rj] != GR[rd]) PC = PC + sext32({off16, 2'b0})

010111 offs[15: 0] [25: 10] rj[9: 5] rd[4: 0]

#### (17) beq(branch on equal)

beq  rj, rd, off16	if(GR[rj] == GR[rd]) PC = PC + sext32({off16, 2'b0})

010110 offs[15: 0] [25: 10] rj[9: 5] rd[4: 0]

#### (18) b(branch)

b  offs26	PC = PC + sext32({off26, 2'b0})

010100 offs[15: 0] offs[25: 16]

#### (19) bl(branch and link)

bl  offs26	GR[1] = PC + 4; PC = PC + sext32({off26, 2'b0})

010101 offs[15: 0] offs[25: 16]

#### (20) jirl(jump indirect register link)

jirl  rd, rj, offs16	GR[rd] = PC + 4; PC = GR[rj] + sext32({off16, 2'b0})

010011 offs[15: 0] [25: 10] rj[9: 5] rd[4: 0]

### 2. 单周期设计

<img src="assets\singlesycle20.png" alt="alt" style="zoom:80%;" />

ALU前的source1变成二选一，选择rj或者PC；source2变成四选一，增添的一个为lu12i服务，一个是pc + 4的4

### 3. 错误详解

#### :one:myCPU_top中imm信号

<img src="assets\error1.1.png" alt="alt" style="zoom:80%;" />

缺少判断need_ui5的选项

<img src="assets\error1.2.png" alt="alt" style="zoom:80%;" />

#### :two:myCPU_top中final_result信号

<img src="assets\error2.1.png" alt="alt" style="zoom:80%;" />

final_result未定义

<img src="assets\error2.2.png" alt="alt" style="zoom:80%;" />

#### :three:myCPU_top中​debug_wb_rf_we信号

信号名写错，多一个n

<img src="assets\error3.1.png" alt="alt" style="zoom:80%;" />

<img src="assets\error3.2.png" alt="alt" style="zoom:80%;" />

#### :four:myCPU_top中​gr_we信号

<img src="assets\error4.1.png" alt="alt" style="zoom:80%;" />

寄存器堆写使能信号，bl指令也需要写寄存器，应去掉

<img src="assets\error4.2.png" alt="alt" style="zoom:80%;" />

#### :five:alu中or_result信号

<img src="assets\error5.1.png" alt="alt" style="zoom:80%;" />

多或了alu_result

<img src="assets\error5.2.png" alt="alt" style="zoom:80%;" />

#### :six:alu中sll_result信号

<img src="assets\error6.1.png" alt="alt" style="zoom:80%;" />

sll_result中操作数一和二写反

<img src="assets\error6.2.png" alt="alt" style="zoom:80%;" />

#### :seven:alu中srl和sra的result信号

<img src="assets\error7.1.png" alt="alt" style="zoom:80%;" />

sr64_result中操作数写反，sr_result的范围写错

<img src="assets\error7.2.png" alt="alt" style="zoom:80%;" />

## 三、工程详解

### 1. 关于编译

#### (1) 下载并安装loongarch交叉编译工具链

+ 打开终端，进入压缩包所在目录，将其解压至/opt目录下

```
$ sudo tar zxvf loongarch32r-linux-gnusf-2022-05-20-x86.tar.gz -C /opt/
```

+ 确保/opt/loongarch32r-linux-gnusf-2022-05-20-x86/bin/存在，执行指令将其添加至环境变量

```
$ echo "export PATH=/opt/loongarch32r-linux-gnusf-2022-05-20-x86/bin/:$PATH" >> ~/.bashrc
```

+ 重新打开终端，输入loongarch32并敲击tab键，如果可以补全，则已安装成功，此时可以编写hello.c并进行编译看其是否可以工作

```
$ loongarch32r-linux-gnusf-gcc hello.c
```

#### (2) 编译func测试程序

+ 根据func目录下的test_config.h文件配置exp6的功能点

+ 执行`make clean`指令和`make`指令，在func/obj/文件夹下，会存在inst_ram.coe、inst_ram.mif和test.s文件
+ 失败了啊啊啊啊啊啊啊啊啊啊啊啊啊not found!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!😭转战压缩包惹

### 2. 生成golden_trace

打开gettrace.xpr，运行gettrace工程的仿真，生成新的参考trace文件golden_trace.txt，注意等仿真全部运行完成才会得到完整内容

### 3. 重新定制inst_ram

在sources窗口找到inst_ram IP窗口，双击或右键后点击"Re-customize IP……"，在定制界面中选择RST&Initialization选项卡，在Coefficients File中点击Browse选择新生成的inst_ram.coe文件，点击OK

在弹出的Generate Output Product窗口中点击Generate（根据需要选择global或ooc模式），完成重新定制

**每次重新编译后，都需要重新进行golden_trace和inst_ram定制**

### 4. 仿真结果

终端打印pass即为成功

