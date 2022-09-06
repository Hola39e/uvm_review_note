# UVM sequence复习 get/get_next_item区别

## driver seq_item_port的方法

- driver的`seq_item_port` 是 `uvm_seq_item_pull_port#(REQ,RSP)`类型。

- sequencer的`seq_item_export`是`uvm_seq_item_pull_imp #(REQ, RSP, this_type)`类型。

`seq_item_port`其包含了以下的内建方法

- `task get_next_item(output REQ)`
- `task try_next_item(output REQ)`
- `function void item_done(input RSP)`
- `task wait_for_sequence()`
- `function bit has_do_available()`
- `function void put_response(input RSP)`
- `task get(output REQ)`
- `task peek(output REQ)`
- `task put(input RSP) `

其中可以看到`get_next_item`和 `get()`都可以获得REQ item，以下是两者的区别。

## `get_next_item` 和 `get`区别

两者的区别在dirver与sequencer和sequence的执行关系有关。先说结论吧：

- get_next_item只是获得req fifo中的item，并不会调用item_done
- get除了从fifo中获得item，同时还会直接调用item_done

**为什么会区分这种方法呢，可以更加细致的区分应用场景**

- 举例来说 get_next_item更适合使用在apb

### `get_next_item`的调用情况

![在这里插入图片描述](https://s2.loli.net/2022/09/05/xWSs3GDiaYKumjw.png)

**在sequence侧发生的事情**

![image-20220904225542088](https://s2.loli.net/2022/09/05/4lnX9twc3aAYmGg.png)

1. sequence通过`start_item()`方法向sequencer请求仲裁，请求获得权限，在没有被grant之前会一直被阻塞
2. start item执行后可以对item进行随机约束，因为此时还没将item送给sequencer
3. 然后执行`finish_item`，可以看到主要包含四个部分，mid_do和post_do可以不用关心，是留给用户自定义的回调函数。重要的是另外两个函数
   1. `sequencer.send_request(item)`这个函数完成的是将item发送给sequencer，将item发送到req fifo中
   2. finish_item会一直阻塞等待driver 侧的item done

**在driver侧发生的事情**

- driver在调用get_next_item，尝试从req fifo中取出数据，但实际上get_next_item真正是在哪里执行的呢，实际上是在sequencer中执行的，这与TLM1.0通信有关。port调用get_next_item函数，port的get_next_item函数调用imp的get_next_item函数，imp的get_next_item函数再调用组件的get_next_item函数，归根结底实际上调用的是sequencer中的get函数。可以在UVM源码中看到，get_next_item最终执行的部分是在sequencer中完成的，内容如下：

  ![image-20220904233739410](https://s2.loli.net/2022/09/05/PVNTYHSK4nkhrF2.png)

- `m_req_fifo`虽然是fifo，但实际上也是tlm端口，其类型是`uvm_tlm_fifo #(REQ) m_req_fifo;`是在`uvm_sequencer_param_base`中定义的

### driver中get的发生的事情

![image-20220905174322778](https://s2.loli.net/2022/09/05/TewXYkh4jNb8O3a.png)

- 可以看到，在driver中调用的seq_item_port.get依旧是最终调用的是sequencer.get
- 其中除了对fifo进行peek以外，还执行了item_done()的操作，也因此，get方法是不会返回RSP的,需要额外调用put_response来返回RSP

**RSP是在哪里返回的？**

- 对于`get_next_item`来说，RSP是使用`item_done(RSP)`来返回的，item done函数内部会调用put_response(在RSP != null的情况下)
- 对于`get`来说，get内部调用的item done没有传递参数，所以如果需要返回RSP需要手动调用put函数返回rsp

**源码中使用`m_req_fifo.peek(t)需要注意的事情`**

- 使用了peek的方法，代表了并不会数据并不会弹出fifo，而是继续保留在fifo中，直到调用item done之后，数据才会弹出，使用peek获得的是下一个item
- ![image-20220905203246092](https://s2.loli.net/2022/09/05/IBRGTqeysWvnd1F.png)
- 可以看到，在item done中才对fifo进行了try get。在这之前fifo中的数据没有取出，还只是用的peek

![在这里插入图片描述](https://s2.loli.net/2022/09/05/VZy6OAGigrDhNmq.png)

