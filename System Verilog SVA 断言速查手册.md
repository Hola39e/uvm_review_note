# System Verilog SVA 断言速查手册

## 基础知识

### 并行断言的调度机制

- 在preponed阶段采样稳定的变量值
- 在observe阶段执行并行断言
- 在reactive区执行pass/fail语句

### assert，property，sequence关系

- assertion可以直接包含一个property

- assertion也可以清晰地独立声明property
- 在property内部可以有条件地关闭
- property可以包含sequence
- 复杂的property可以独立声明多个sequence

#### sequence的角色

- sequence是用来表示一个或者多个时钟周期内的时序描述
- 是property的基本构成模块，并经过组合来描述复杂的功能属性

## 基本操作符号

### 交叠非交叠符号

`|->`:条件满足，评估后续算子序列，不满足表现为空成功

`|=>`:非交叠交错符号，条件满足，下一个周期评估后续算子序列

### 周期延时符号

`##`: delay n cycles

`##[min:max]`: 一个范围内的时钟延时

`##[1:$]`：无穷大周期

### 事件重复符号

`[*n]`:表示事件重复，n为非负数，但是不能为$，保持需要是连续的

`[*2:5]`:表示一定范围内的重复

`[=m]`:表示一个事件的连续性，但是不需要重复

### 关键重点 Consecutive repetition  Goto repetition and Nonconsecutive repetition 

事件重复符号有三种

- consecutive repetition 重复事件符号
- goto repetition 直达事件符号
- nonconsecutive repetition 非重复事件符号

重复事件和非重复事件符号区别在上文已经说过了，**最重要的是goto repetition and nonconsecutive repetition的区别**

- goto repetition指的是在连续或者非连续的条件满足之后**立即进行后面的判断**
- nonconsecutive repetition指的是在连续或者非连续的条件满足的当时和后面延时的几个时间，但是在下一个满足时间之前

> **Nonconsecutive repetition** The overall repetition sequence matches at or after the last iterative match of the operand, but before any later match of the operand.
>
> **Goto repetition** The overall repetition sequence matches at the last iterative match of the operand.





### and操作符

- 从同一个起点开始seq1和seq2均满足
- 满足的时刻发生在两个序列都满足的周期，即稍晚序列满足的时刻（前面那个序列可以先满足）
- 两个序列满足的时间可以不同
- 如果两边是采样信号的话，那么就在相同周期内左边右边都满足

### intersect操作符（相交）

- 需要两个序列在同一时钟周期内匹配，即同时开始，同时结束

### or操作符

- 表示两个序列至少需要一个满足
- seq1和seq2都同一个时刻被触发
- 最终满足seq1或者seq2
- 每个seq的结束时间可以不同，结束时间以序列满足的最后一个为准

### first_match操作符

- 用来从多次满足的序列中选择第一次满足的时刻

```verilog
sequence t1;
  te1 ## [2:5] te2;
endsequence

sequence ts1;
  first_match(te1 ##[2:5] te2);
endsequence

sequence ts2;
  first_match(t1);
endsequence

// 实际应用，总线上进入idle状态的同时状态机的状态也需要进入idle状态
sequence checkBusidle;
  (##[2:$] (frame && irdy));
endsequence

property first_match_idle;
  @(posedge clk) first_match(checkBusidle) |-> (state == busidle);
endproperty
```

### throughout贯穿操作符

- 用来检查一个信号或者表达式在贯穿一个序列时是否满足要求
- `sig1/exp1 throughout seq`（sig1需要在seq持续的时间内保持）

```verilog
// burst mode 拉低两个周期后，trdy和irdy也需要连续7个周期内保持为低，
// 同时burst模式信号也应该在这一连续周期内保持为低
sequence burst_rule;
  @(posedge clk)
  $fell(burst_mode) ##0 (!burst_mode) throughout (##2 ((trdy == 0) && (irdy == 0)) [*7]);
endsequence
```

### within操作符

- 用来检查一个序列与另外一个序列在部分周期长度上的重叠
- seq1 within seq2
- 如果seq1满足在seq2的一部分连续时钟周期内成立，seq1 within seq2成立
- 相当于seq1在seq2成立的时间内成立了

### if else 操作符

- 在sequence中可以使用if else

```verilog
// cache访问中，如果cache lookup满足，那么状态机状态为READ CACHE，否则应该为REQ OUT
property cache_hit_check;
  @(posedge clk) (state == CACHE_LOOKUP) ##1 (CHit || CMiss) |-> 
  if(CHit)
    ##1 (state == CACHE_READ);
  if(CMiss)
    ##1 (state == REQ_OUT);
endproperty
```

## 访问采样方法

- `$rose()`：表示与上一个采样周期相比，是否从0跳边为1
- `$inunkown()`:当前采样值为X值 （通常可以使用 `not $isunkown(data)`）
- `$stable()`用来表示在连续两个采样周期内，表达式的值不变

- `$past(expr [, num_cycles])`用来访问在过去若干采样周期前的数值