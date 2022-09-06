# uvm AXI master/slave driver 驱动器编写思路

## 本文要实现的目标

- AXI4 的master driver and slave driver
- master/slave 支持outstanding

## AXI4 master driver实现思路

### `axi_master_driver`成员变量与方法

#### 成员变量

- `uvm_seq_item_pull_port#(REQ, RSP) seq_item_port` 内建的pull port，用作写pull port
- `uvm_seq_item_pull_port#(REQ, RSP) seq_read_item_port` 用作读pull port
- `virtual axi_interface vif`
- `REQ`
- `bit write_done, read_done`

#### 方法

- `funtion new`
- `task run_phase`
- `task drive`
- `task send_write_address`
- `task send_read_address`
- `task send_write_data`

### 方法解析

#### `task run_phase()`的执行内容

```verilog
task run_phase();
	vif.mst_drv_cb.BREADY <= 1'b1;
	vif.mst_drv_cb.RREADY <= 1'b1;
  while(1)begin
    drive();
    #1;
  end
endtask
```

- 驱动`BREADY/RREADY`保持为high
- 循环中执行`drive()`

#### `task drive()`中执行内容

- 复位时对VALID信号拉低
- 通过fork join_none启动两个线程，一个判断是否`write_done`，一个判断是否`read_done`，如果`write_done`判断成功，再在子线程内部启动一个fork join，再启动两个子线程，写操作创建`send_write_address() and send_write_data()`，读操作创建`send_read_address()`

#### `task send_write_address()`

第一个时钟沿驱动下列信号

- AWID
- AWADDR
- AWLEN: burst length
- AWSIZE: burst size of every burst transfer
- AWBURST

第二个时钟沿驱动下列信号

- AWVALID

第三个时钟沿等待下列信号

- 等待AWREADY == 1
- 如果在本时钟沿采样到AWREADY == 1，那么拉低AWVALID
- 等待BREADY == 1

#### `task send_write_data()`

