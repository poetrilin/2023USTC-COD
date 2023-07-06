# 实验五

- 学号: PB21061373
- 姓名:刘兆宸
- 实验日期: 2022-5-24

[TOC]

## 实验目标

- 完成流水线cpu完成仿真与上板

## 实验内容

#### Imm Module && Control Module

生成立即数模块比较简单只需根据指令的特点根据组合逻辑即可.imm_type给出情况后我们对指令解析即可(注意符号拓展),这里直接用的上次单周期的源码,

数据通路基本没变,Control模块基本只要改改线网名称就行.(主要是查连线的错误耽误时间:cry:).

#### Hazard模块

控制冒险由IM和DM的哈佛结构解决,数据冒险考虑前递和气泡,

先分析前递的条件:

前一条指令的目标寄存器是当前 EX 阶段指令的源寄存器时, 需要将数据前递给等待该数据的单元, 对应的逻辑如下 forward 模块的 Verilog 代码, 即

- 当 MEM 阶段的目标寄存器为当前 EX 阶段的源寄存器 (不为 `x0`) 时, 发生 EX 冒险, 需要将上一阶段 ALU 的输出前递给 EX 阶段, 
- 当 WB 阶段的目标寄存器为当前 EX 阶段的源寄存器 (不为 `x0`) 时, 发生 MEM 冒险, 需要将寄存器写回值前递给 MEM 阶段
- 如果两个冒险都发生, 则处理前者, 即  rf_rd0_fd = alu_ans_mem (也就是上一步的ALU结果)
- 如果不发生数据冒险, 则不需要前递

```verilog
 if (rf_we_mem && rf_re0_ex && (rf_ra0_ex != 0) && (rf_ra0_ex == rf_wa_mem)) 
    begin
        rf_rd0_fe = 1'b1;
        rf_rd0_fd = alu_ans_mem;
     end
     else if(rf_we_wb&&rf_re0_ex && (rf_ra0_ex != 0) && (rf_ra0_ex == rf_wa_wb))
     begin
        rf_rd0_fe = 1'b1;
        rf_rd0_fd =rf_wd_wb;
     end
```



load-use

当 EX 阶段指令需要从内存中读取数据写回寄存器, 且写回的寄存器为当前 ID 阶段指令的源寄存器时, 发生 load-use 数据冒险, 该冒险无法被前递解决, 这时需要在流水线中插入停顿, 即停顿一次,包括此时已进入流水线的IF,ID阶段(EX即自身),同时冲刷已到MEM的一级防止数据冒险



```verilog
// Load-use hazard
    if (rf_re0_ex && (rf_ra0_ex != 0) && (rf_ra0_ex == rf_wa_mem) && rf_we_mem && (rf_wd_sel_mem ==2'b10)) begin
    stall_if = 1'b1;
    stall_id = 1'b1;
    stall_ex = 1'b1;
    flush_mem = 1'b1;
    end
```

3.控制冒险

 当分支预测失败时, 发生控制冒险, 这时需要冲刷流水线, 即丢弃 IF/ID, ID/EX 寄存器中的值, PC 将变为分支的地址. 在本次实验中, 由于采用假设分支不发生的策略, 所以冲刷流水线的条件为(必做部分)

```verilog
// jal(jalr)&br
    if (pc_sel_ex == 2'b10 || pc_sel_ex == 2'b01 || pc_sel_ex == 2'b11) begin
        flush_if  = 1'b1;
        flush_id  = 1'b1;
        flush_ex  = 1'b1;
    end
```

#### 选做1

注意的是，当跳转指令在不同阶段出现时，我们需要对其进行优先级的区分。例如在 EX 段
执行的指令是 jalr 的同时，在 ID 段执行的指令是 jal，请思考此时如何正确跳转?

在这种情况下，应该优先进行 jalr 指令的跳转，因为 jalr 的目标地址是通过寄存器计算得出的，需要在 EX 阶段才能得到。而 jal 指令的目标地址可以在 ID 阶段就计算出来，因此可以等待 jalr 指令的跳转结果。如果在 ID 阶段执行的是 jal 指令，而在 EX 阶段执行的是 jalr 指令，那么需要在 ID 阶段记录下 jal 的目标地址，并在 EX 阶段得到 jalr 的跳转地址后，再进行跳转决策，选择跳转到 jalr 的目标地址还是 jal 的目标地址。

修改后:

```verilog
//jal
    if(pc_sel_ex == 2'b10) begin
        flush_id  = 1'b1;
    end 
    // jalr&br
    if (pc_sel_ex == 2'b01 || pc_sel_ex == 2'b11) begin
        flush_if  = 1'b1;
        flush_id  = 1'b1;
        flush_ex  = 1'b1;
    end
```

同时要注意一种情况:

```verilog
jalr x0,0(x2);
jal  x0,JP2;
```

由于我们提前了jal指令,所以jal在ID阶段判断时,jalr在EX阶段也在判断,如果两个都跳转按照汇编规则应该时执行jalr，所以应该jalr优先.Encoder应该是个优先编码器:

```verilog
always @(*) begin
    pc_sel = 2'b00; // 默认为 00
    if (jalr) begin
        pc_sel = 2'b01;
    end else if (br) begin
        pc_sel = 2'b11;
    end
        else if (jal) begin
        pc_sel = 2'b10;
    end 
end
```



### Q2.

>  估算你实现的流水线 CPU 的最小需要时钟周期与其相对单周期的指令平均所需时间的改进（自选数据大致估算即可，无需精确计算）；

最小需要时钟周期应该取决于访存那一级流水的延迟，感觉5级流水线大概比单周期快了3倍的样子,因为原来单周期要考虑最长数据通路的延迟,流水线将这条路直接分为5级,原来的一条指令对应现在的一级流水(最佳理论应该是5倍),但是各级延迟不太一样，比如访存可能耗时较长,总体来说应该会比以前快3倍左右

### Q3.

> 3.实验过程中遇到的一些问题，或者难以解决的内容。你可以记录自己试错的过程，也可以展开自己的心路历程（本项内容不作为评分依据）

上次的单周期cpu的debug基本在Control unit 信号生成,这次主要在CPU连线和Hazard模块,还有初始化,一开始PC难以进入着实搞心态,我记得是把一个初始化信号定义成reg型(wire型无法初始化)解决了.

这次生成的RTL图比上次大多了(因为段间寄存器),为检查连线增加了难度,好在vivado综合可以帮忙检查连线错误(比如位宽和未定义的wire变量).Hazard模块这次数据冒险第一部分做的很快,load use卡了一下,后面选做一分析卡了一小会.不过做的还快,这次的速度比上次要慢(不知道为什么明明上次自己分析感觉更多?)

