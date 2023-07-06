# 实验四

- 学号: PB21061373
- 姓名:刘兆宸
- 实验日期: 2022-5-7

[TOC]

(pandoc导出pdf排版太难看了,也传了.md文件,但是图片限于大小就没传了)

## 实验目标

- 完成单周期流水线完成仿真与上板

## 实验内容

### Q1.

>  1.请根据自己的理解描述本次实验的实验内容以及设计流程，包括部分模块的设计思路，如 Control、Imm、Branch 等；

实验部分主要是对CPU部分的设计,ALU，Regfile模块已在之前的实验完成,并且最重要数据通路已经给出.本次实验的工作量(除了debug:cry:)主要在于Control模块的编写,小部分在于Imm,Branch模块的编写.

![](./datapath.png)

#### Imm Module

生成立即数模块比较简单只需根据指令的特点根据组合逻辑即可.imm_type给出情况后我们对指令解析即可(注意符号拓展),code如下:

```verilog
module imm(
    input       [31:0]  inst,
    input       [2:0]   imm_type,
    output reg  [31:0]  imm
);
    always @(*) begin
        case (imm_type) 
        3'b001:  // I-type  [11:0]imm
            imm = {{21{inst[31]}}, inst[30:20]};                               
        3'b010:  // B-type
              
        3'b011:  // S-type
                            
        3'b100:  // J-type 
           
        3'b101:  // U-type
            12'h0000};                                     
        default: imm = 32'h0;
        endcase
    end
endmodule
```

#### Branch Module

分支判断只要解析了指令后判断br是比较容易的,code如下:

```verilog
module Branch(
    input[31:0] op1,
    input[31:0] op2,
    input[2:0]  br_type,
    output reg br
);
    always@(*)begin
    case (br_type)
        3'b000 : 
        3'b001 : 
        3'b010 : 
        3'b011 : 
        3'b101 : 
        3'b110 : 
        default: 
    endcase
    end
endmodule
```

#### Control Module

控制模块相对前两个较为复杂,观察下面图片我们来分析各信号的解析.

<img src="C:\Users\mlzc1\AppData\Roaming\Typora\typora-user-images\image-20230508184754220.png" alt="image-20230508184754220" style="zoom:50%;" />

jal,jalr两个信号相对容易,只需对opcode判断即可

mem_we只有inst为S型指令才为1,

wb_en和wb_sel信号,写回可能有以下几种情况:ALU运算结果(R型,大部分I型指令)、PC + 4 的结果、lw:内存读取出的数据结果(lw)，以及立即数单元的输出结果(),所以解析这些命令即可

immtype有几种情况,感觉根据指令分类即可: 

br_type也是解析指令,先判断是B-type,再根据funct3判断具体是哪种然后给BR模块传信息

alu_ctrl,alu_op1,2,对各种命令把每个命令过一遍即可.(大部分debug都是根据仿真完成,边debug边加深印象qwq)

最后代码如下

```verilog
module Control(input [31:0]inst,
               output jal,
               output jalr,
               output reg[2:0] br_type,  //
               output wb_en,             //
               output [1:0] wb_sel,
               output alu_op1_sel,
               output alu_op2_sel,
               output reg[3:0] alu_ctrl,
               output reg [2:0]immtype,
               output mem_we);

// 解析指令
wire [6:0] opcode = inst[6:0];
wire [2:0] funct3 = inst[14:12];
wire [6:0] funct7 = inst[31:25];
// 解析类型
reg [5:0] type;
localparam R_type = 6'b000001;
localparam I_type = 6'b000010;
localparam B_type = 6'b000100;
localparam S_type = 6'b001000;
localparam J_type = 6'b010000;
localparam U_type = 6'b100000;
always @(*) begin
    case (opcode)
        7'b0110011: type = R_type;
        7'b1100111,                  //jalr
        7'b0000011,                 //Load指令
        7'b0010011: type = I_type;//带立即数的算术逻辑指令
        7'b1100011: type = B_type;
        7'b0100011: type = S_type;
        7'b1101111: type = J_type;
        7'b0010111,7'b0110111: type = U_type;
        default:    type = 6'b000000;
    endcase
end

assign jal  = (opcode == 7'b1101111);
assign jalr = (opcode == 7'b1100111);
/// R
assign wb_en = 
// 写回数据选择,0-3分别代表ALU 运算结果、PC + 4 的结果、lw:内存读取出的数据结果，以及立即数单元的输出结果。
assign wb_sel = 

// 存储器写使能
assign mem_we = (type == S_type);
// ALU第一个操作数选择  J-typle或者B_type 或者auipc  选PC ,否则选寄存器堆的
assign alu_op1_sel =

// ALU第二个操作数选择 1:立即数  0: 寄存器堆文件
assign alu_op2_sel = 

// ALU控制信号   alu_ctrl
always @(*)
begin
    casez (opcode)
    7'b1100011: alu_ctrl = 4'b0000; //branch +

    7'b0010011: begin //带立即数的算术逻辑指令 0000011
        case (funct3)
            3'b000:  alu_ctrl = 4'b0000; //addi
            3'b100:  alu_ctrl = 4'b0111; //xori
            3'b110:  alu_ctrl = 4'b0110; //ori
            3'b111:  alu_ctrl = 4'b0101; //andi
            3'b010:  alu_ctrl = 4'b0100; //slti
            3'b011:  alu_ctrl = 4'b0011; //sltiu
            default: alu_ctrl = 4'b0000;
        endcase
    end
    7'b0110011: begin //R_type
        case (funct3)
            3'b000:  alu_ctrl = (funct7[5]) ? 4'b0001 : 4'b0000; //add&sub
            3'b001:  alu_ctrl = 4'b1001; //sll
            3'b010:  alu_ctrl = 4'b0100; //slt
            3'b011:  alu_ctrl = 4'b0011; //sltu
            3'b100:  alu_ctrl = 4'b0111; //xor
            3'b110:  alu_ctrl = 4'b0110; //or
            3'b111:  alu_ctrl = 4'b0101; //and
            default: alu_ctrl = 4'b0000;
        endcase
    end
    7'b0110111: alu_ctrl = 4'b1010; //Choose_B //lui
    

    default: alu_ctrl = 4'b0000;
endcase
end

                //imm-type
                always @(*)
                begin
                    case (type)
                        6'b000010: immtype = 3'b001; // I-type
                        6'b000100: immtype = 3'b010; // B-type
                        6'b001000: immtype = 3'b011; // S-type
                        6'b010000: immtype = 3'b100; // J-type
                        6'b100000: immtype = 3'b101; // U-type
                        default:   immtype = 3'b000;
                    endcase
                end

                //br_type
                always @(*)
                begin
                    if (type[2])begin
                        case (inst[14:12])
                            3'b000 : br_type = 3'b001;//beq
                            3'b001 : br_type = 3'b011;//bne
                            3'b100 : br_type = 3'b010;//blt
                            3'b101 : br_type = 3'b101;//bge
                            3'b110 : br_type = 3'b110;//bltu
                            default : br_type = 3'b000;
                        endcase
                    end
                    else  br_type = 0;//不跳转
                end
            
            endmodule

```

### Q2.

> 2.分析本次实验提供的单周期 CPU 数据通路和教材上的单周期 CPU 数据通路的差异。 这些差异会在哪些指令上体现出来？

书上的数据通路:

相对于书上的数据通路本实验

1.存储器与CPU分离了，所以CPU相对于MEM可直接获取Inst和rd_mem,并传递next_PC,wd_mem等等.

2.对nextPC做了个MUX,并添加了Branch模块,更加方便解析B型指令,只需将inst解析出BR_type交给branch模块即可.

### Q3.

> 3.实验过程中遇到的一些问题，或者难以解决的内容。你可以记录自己试错的过程，也可以展开自己的心路历程（本项内容不作为评分依据）

**写出一堆bugs**

:cry:testcase每过一两个测试点就FAIL一遍的我也是挺不容易的,

blt忘了有符号数比较和无符号数比较不一样了qwq

然后Control模块之前opcode 写的一堆数字总是犯迷糊

jal x5没保存(忘记wb了),还有imm符号拓展,ALU有符号数无符号数一开始又忽略了

**PDU对debug感觉用到什么,文档很长但是实用性没有体现出来效果?**

## 附

源代码文件树:

```shell
src
├── main
│   ├── constrains.xdc  #约束文件
│   ├── data_fls.coe    #fls 的DM 存储数列前两位和结果
│   ├── fls.coe         #fls  IM
│   ├── simul       
│   │   └── CPU_tb.v     # 仿真文件
│   ├── sources_1
│   │   ├── ip
│   │		...          #例化ip核部分,略
│   │   └── new
│   │       ├── Branch.v
│   │       ├── CPU.v
│   │       ├── Control.v
│   │       ├── MEM.v
│   │       ├── Memory_Map.v
│   │       ├── Mux1.v
│   │       ├── Mux2.v
│   │       ├── NPC_sel.v
│   │       ├── PC_reg.v
│   │       ├── PDU
│   │       │   ├── Ded.v
│   │       │   ├── Hex2ASC.v
│   │       │   ├── PDU.v
│   │       │   ├── PDU_ctrl.v
│   │       │   ├── Receive.v
│   │       │   ├── Segment.v
│   │       │   ├── Send.v
│   │       │   └── Uout.v
│   │       ├── Shift_reg.v
│   │       ├── Top.v
│   │       ├── alu.v
│   │       ├── check_data_sel.v
│   │       ├── imm.v
│   │       └── regfile.v
│   ├── test_ext.coe    #附加测试文件的IM
│   └── testcase.coe    #testcase 的IM
└── test
    ├── fls.asm         
    ├── test_2.asm
    └── testcase.asm
```

