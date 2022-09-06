# AXI事务ID与outstanding/out of oreder/interleaving实现关系

## 前言

众所周知，AXI3/AXI4支持outstanding/out of order/ interleaving的特性，但是这一特性是根据AXI哪一路实现的，以及需要注意和说明的地方是什么。

here is the analysis.

## 超前传输 outstanding的实现

### outstanding是什么

**outstanding表示AXI超前传输的特性，表示这笔transaction还没完成，可以先提起别的事务，这么说如果抽象的话，那么就是AR/AWchannel可以提起多个事物，即便在W/R channel数据还没发完或者还没发的情况下。**

### 超前传输是如何实现的？

在不考虑其他两个特性（乱序和交织）的情况下，AXI事务都是顺序完成的，这时多事务在途不需要其他信号来实现，直接根据write channel或者read channel的`LAST`信号或者response channel的信号来判断分割事务就可以了。可以认为是一种**队列或FIFO**结构，`AR/AW`channel发起事务就是顺序压入队列，当这些事务对应完成的时候，就最提起事务先弹出队列，表示结束，然后接收完成后面的事务。

### 超前传输的可支持性

超前传输需要master和slave都支持超前传输，其中outstanding depth表示了主机超前传输的性能，表示同一时刻最多支持多少个AXI 事务在途。

如果slave不支持outstanding如何响应？

> AXI 从机可选地支持超前传输，假设从机不支持超前传输，只需要在接收到 Trans0 后，置低 AxREADY 信号，阻止主机超前传输。在返回读数据后，再置高 AxREADY 信号，接收下一事务。如下图所示，主机将 Trans1 保持在总线上直至从机接收。

## 乱序 out of order的实现

#### out of order乱序是什么？

当有多个事务在途的时候，有的事务可能先准备好，因此可以先发送在总线上。那就需要面对一个问题，如何判断返回的是哪个事务的数据？这就与AXI的事务ID有关

### AXI的事务ID

AXI的事务ID包含了：

- AWID
- WID (只有AXI3有，AXI4没有，因此不支持写交织)
- BID
- ARID
- RID

### AXI out of order乱序的实现模型与思路

AXI乱序的特性是由地址channel和响应channel上的ID信号`AWID/ARID`和`WID/RID`来实现的，根据ID不同来标识事务不同，**但是并不代表不同事务传输`AWID/ARID`就已经要不同**

1. 不同事务的AxID如果一致，那么这些事务就不能实现out of order，只能进行顺序完成。（因此需要重排序模型，重排序模型包括了事务缓冲区和数据缓冲区，事务缓冲区存放在途需要完成的事务，对于slave来说，其可能对于不同事务完成的时间不同，因此事务准备好了与事务缓冲区的首个事务比较，如果匹配就输出，如果不匹配就进入数据缓冲区）
2. 如果不同事务AxID不同，那么这些事务之间可以乱序。那么不同AxID事务的数据，对于AXI读来说，如何判断返回的数据属于哪个事务呢，是通过`RID`来进行匹配的，也就是说，**在完成乱序传输的时候，需要`RID`和`ARID`保持一致，以标识不同事务的数据**
3. 那么对于实际情况来说，在实际传输中，可能有的事务AxID是不同的，有的是相同的，这是如何解决的？答：对于ID相同的就顺序完成，对于ID不同的可以乱序。
4. 在实际应用中，在slave的实现中，为每个ARID准备了一个事务缓冲区和数据缓冲区，以支持相同ID和不同ID的数据顺序传输和乱序传输。

### 有关于写乱序

**有一个重要的观点，写乱序不是针对于master来说的，说的不是不同事务发送的顺序可以不一样（即事务A先发起，事务B后发起，先发B事务的数据然后再发A事务的数据，如此叫做乱序，这种观点是错误的！）**

写乱序指的是：乱序是针对slave来说的，slave接收到了多个事务（可能是多个master传输来的事务）那么slave返回BID的顺序与发送过来的AWID顺序是不同的，这叫做写乱序。

对应写事务乱序跟读事务也是相同的

- 通过`BID`和`AWID`来表示数据所属事务

## AXI interleaving 交织的实现

### 什么是交织？

interleaving表示不同事务的数据可以被打散混合排列（**但是注意，这里说的混合排列是不同事务间的，同一个是数据是不循序被打乱的**）。例如事务1数据是0a 0b，事务2数据是1a 1b，如果不支持交织，那么总线上数据的传输需要是0a0b1a1b或者1a1b0a0b，如果支持交织的话那么就可以是0a1a0b1b（或者别的插入顺序）

### 交织的实现

对于读交织来说，读事务的response方向和读方向的相同的，不同事务交织是通过RID来进行识别的，**也就是说RID在AXI传输中即起到了out of order乱序的不同事务识别也起到了interleaving交织中不同事务数据的识别**

![ ](https://community.arm.com/cfs-file/__key/communityserver-discussions-components-files/18/7282.7181.pastedImage_5F00_0.png)

对于写交织来说，由于写方向和response方向不一样，那么`WID`就是提供了写交织的不同事务的识别，`BID`提供了乱序不同事务的识别。使用与 AWID 匹配的 BID 标识写回复所属的事务 ID。实际上从机给出写回复可以类比读事务中给出读数据的过程。

## 总结

到此为止，介绍了AxID和RID/BID在AXI乱序中的作用。那么还剩下`WID/RID`