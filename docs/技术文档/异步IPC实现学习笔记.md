## 内核对象

### capability

可以被用户态程序所持有的一种标识符(令牌)。一个标识符绑定系统中的一种资源（内存，端点等）。

用户态程序访问资源需要提供相应的标识符。

### capability space（cspace）

存储一个进程所有capability的数据结构。由很多个capability node组成。

一个进程对应一个cspace

`sel4::BootInfo::init_cspace_local_cptr::<sel4::cap_type::Notification>`

获取指定类型的cspace的指针

### capability node (cnode)

cspace中的节点，存储一定数量的cslot。

cslot中存放cspace。capability必须放在cslot中才能被使用

`sel4::BootInfo::init_thread_cnode()` 获取初始线程的cnode

`cnode.relative` 通过偏移量查找一个cnode

`cnode.mint` 根据原有的一个capability复制出一个新的cnode

### notification

通知对象。发送方通过该对象发送消息，接收方通过该对象接收消息。

线程需要拥有一个指向notification对象的capability才能与之交互。

### badget

一个Notification内核对象只能绑定到一个接收者线程，而一个接收者线程也只能绑定一个Notification内核对象。但是，同一个Notification内核对象的Capability可由不同的发送者线程持有，并可以接受多个不同的发送者线程的发送请求，不同发送者依靠特定的标识符（称为Badge）进行区分。

### endpoint

通知机制的作用对象，发送和接收的两端。

### sel4::BootInfo

C++包装器，提供了对微内核启动时提供的信息的访问,包含Capability等。

### tcb

`sel4::BootInfo::init_thread_tcb();` 初始化线程控制块

## 代码注释

用户态测试函数

```rust
pub fn async_ipc_test(_bootinfo: &sel4::BootInfo) -> sel4::Result<!>  {
    //异步运行时初始化
    runtime_init();
    //taic初始化
    crate::device::taic::taic_init(_bootinfo);
    //用户态分配器初始化
    let obj_allocator = &GLOBAL_OBJ_ALLOCATOR;
    debug_println!("exec size: {}", size_of::<Executor>());
    let mut async_args = AsyncArgs::new();
    let badged_notification = obj_allocator.lock().alloc_ntfn().unwrap();
    // 初始化receiver线程
    let recv_tcb = sel4::BootInfo::init_thread_tcb();//sel4 系统调用创建线程
    recv_tcb.tcb_bind_notification(badged_notification)?;// 绑定内核对象
    let server_process_id = alloc_receiver(recv_tcb, badged_notification, 0)?;//taic 和 os 分配一个接受方

    // 把server_process_id和buffer填进去
    let _lock = async_args.lock.lock();
    async_args.server_process_id = Some(server_process_id);
    let ipc_new_buffer = unsafe {
        obj_allocator.lock().alloc_new_buffer_without_free()
    };
    async_args.ipc_new_buffer = unsafe { Some(ipc_new_buffer) };

    drop(_lock);
    //生产客户端的线程
    let child_tcb = Some(obj_allocator.lock().create_thread(async_helper_thread, async_args.get_ptr(), 255, 0, true)?.cptr().bits());

    // 把子线程tcb(客户端)填进去
    let _lock = async_args.lock.lock();
    async_args.child_tcb = child_tcb;
    drop(_lock);
    // wait,等reply_ntfn初始化完毕
    loop {
        let _lock = async_args.lock.lock();
        if async_args.reply_ntfn.is_some() {
            break;
        }
        drop(_lock);
        r#yield();
    }
    //生成服务端的协程  cid相当于句柄
    let cid = Box::new(coroutine_spawn_with_prio(Box::pin(recv_req_coroutine(async_args.get_ptr())), 1));
    //拿客户端的线程id
    let client_process_id = async_args.client_process_id.unwrap();
    debug_println!("[server] cid: {}", cid.0);
    //taic注册接收者，发送方
    register_receiver(client_process_id, cid.0 as usize);
    register_sender(client_process_id);

    //把server_ready 置 true
    let _lock = async_args.lock.lock();
    async_args.server_ready = true;
    drop(_lock);
    // coroutine_run_until_complete();

    //等所有协程运行完
    while !coroutine_is_empty() {
        coroutine_run_until_blocked();
        r#yield();
    }
    debug_println!("TEST_PASS");
    let uintr_trigger_info = format!("server uintr cnt: {}",
        unsafe { UINT_TRIGGER });
    mutex_print(uintr_trigger_info);

    sel4::BootInfo::init_thread_tcb().tcb_suspend()?;
    unreachable!()
}
```

taic 分配接收者

```rust
// pid,LQ
pub static mut LQ_MAP: BTreeMap<usize, Arc<LocalQueue>> = BTreeMap::new();
#[thread_local]
pub static mut process_id: usize = 0;
pub fn alloc_receiver(tcb: TCB, ntfn: Notification, hart_id: usize) -> Result<usize, Error> {
    // super::init_utrap_handler();
    ntfn.register_receiver(tcb.cptr())?;//通知系统调用，注册接收者
    let mut recv_idx = 0;
    with_ipc_buffer(|buffer| {
        unsafe {
            //一个buffer对应一个queue
            recv_idx = buffer.inner().uintr_flag as usize;
            process_id = recv_idx;
            let lq = Arc::new(TAIC.alloc_lq(1, recv_idx).unwrap());//taic 分配一个本地队列
            //写hartid
            lq.whart(hart_id);
            //把当前的LQ插进去
            LQ_MAP.insert(recv_idx, lq.clone());
            //Executor的初始化，给ready_queue赋值
            local_queue_init(lq);
        }
    });
    Ok(recv_idx)
}
```

taic 运行时注册发送&接收

```rust
pub fn register_receiver(sender_idx: usize, handler: usize) {
    unsafe {
        let lq = LQ_MAP.get(&process_id).unwrap();
        lq.register_receiver(1, sender_idx, handler);
    }
}

#[inline]
pub fn register_sender(recv_idx: usize) {
    // debug_println!("Registering sender: {}, {}", recv_idx, sender_idx);
    unsafe {
        let lq = LQ_MAP.get(&process_id).unwrap();
        lq.register_sender(1, recv_idx);
    }
}
```

服务端协程函数

```rust
// 接收请求的协程 （服务端）
async fn recv_req_coroutine(arg: usize) {
    debug_println!("hello recv_req_coroutine");
    //初始化，解构参数
    static mut REQ_NUM: usize = 0;
    let current_cid = coroutine_get_current().0 as usize;
    let async_args= AsyncArgs::from_ptr(arg);
    let client_process_id = async_args.client_process_id.unwrap() as usize;
    let new_buffer = async_args.ipc_new_buffer.as_mut().unwrap();
    //循环处理请求
    loop {
        //缓冲区有请求
        if let Some(mut item) = new_buffer.req_items.get_first_item() {
            // item.msg_info += 1;
            // debug_println!("hello get item");
            // let _res = matrix_test::<MATRIX_SIZE>();
            //把回复写回去
            new_buffer.res_items.write_free_item(&item).unwrap();
			//状态检测
            if new_buffer.recv_reply_status.load(SeqCst) == false {
                new_buffer.recv_reply_status.store(true, SeqCst);
                //发taic
                unsafe {
                    crate::device::taic::interface::send_signal(client_process_id);
                }
            }
            //更新req_num 如果测试结束就退出
            unsafe {
                REQ_NUM += 1;
                if REQ_NUM == SEND_NUM {
                    break;
                }
            }
            
        } else {//如果空了的话
            register_receiver(client_process_id, current_cid);
            new_buffer.recv_req_status.store(false, SeqCst);
            //阻塞
            yield_now().await;
        }
    }
}
```

发送IPC的子线程

```rust
//子线程(发送ipc)
pub fn async_helper_thread(arg: usize, ipc_buffer_addr: usize) {
    //注册 ipc_buffer
    let ipc_buffer = ipc_buffer_addr as *mut sel4::sys::seL4_IPCBuffer;
    let ipcbuf = unsafe {
        IPCBuffer::from_ptr(ipc_buffer)
    };
    sel4::set_ipc_buffer(ipcbuf);
    //子线程中的运行时初始化
    runtime_init();
    debug_println!("async_helper_thread start2");
    let async_args = AsyncArgs::from_ptr(arg);
    while true {
        let _lock = async_args.lock.lock();
        if async_args.child_tcb.is_some() && async_args.server_process_id.is_some() && async_args.ipc_new_buffer.is_some() {
            break;
        }
        drop(_lock);
        r#yield();
    }
    // while async_args.child_tcb.is_none() || async_args.req_ntfn.is_none() || async_args.ipc_new_buffer.is_none() {
    //     // debug_println!("{} {} {}", async_args.child_tcb.is_none(), async_args.req_ntfn.is_none(), async_args.ipc_new_buffer.is_none());
    // }
    debug_println!("[client] exec_ptr: {:#x}", get_executor_ptr());
    let tcb = LocalCPtr::<TCB>::from_bits(async_args.child_tcb.unwrap());
    let reply_ntfn = GLOBAL_OBJ_ALLOCATOR.lock().alloc_ntfn().unwrap();

    tcb.tcb_bind_notification(reply_ntfn).unwrap();
    let client_process_id = alloc_receiver(tcb, reply_ntfn, 0).unwrap();
    let server_process_id = async_args.server_process_id.unwrap();

    let new_buffer = async_args.ipc_new_buffer.as_mut().unwrap();
    let cid = Box::new(
        coroutine_spawn_with_prio(Box::pin(recv_reply_coroutine(arg, SEND_NUM)), 0)
    );

    // register_usoft_handler(Box::new(move || {
    //     coroutine_delay_wake(*cid);
    //     // re_register(server_process_id);
    // }));

    register_sender_buffer2(server_process_id, new_buffer);
    register_receiver(server_process_id, cid.0 as usize);
    register_sender(server_process_id);

    let _lock = async_args.lock.lock();
    async_args.client_process_id = Some(client_process_id);
    async_args.reply_ntfn = Some(reply_ntfn.bits());
    drop(_lock);
    while true {
        let _lock = async_args.lock.lock();
        if async_args.server_ready {
            break;
        }
        drop(_lock);
        r#yield();
    }
    let base = 100;
    for i in 0..COROUTINE_NUM {
        coroutine_spawn(Box::pin(client_call_test(server_process_id as i64, (base + i) as u64)));
    }
    
    debug_println!("test start");
    let start = get_clock();
    while !coroutine_is_empty() {
        // let start_inner = get_clock();
        coroutine_run_until_blocked();
        // debug_println!("coroutine_run_until_blocked: {}", get_clock() - start_inner);
        r#yield();
    }
    // coroutine_run_until_complete();
    let end = get_clock();
    let uintr_trigger_info = format!("client uintr trigger cnt: {}",
        unsafe { UINT_TRIGGER});
    mutex_print(uintr_trigger_info);
    let async_test_res_info = format!("async client passed: cost: {}", end - start);

    mutex_print(async_test_res_info);

    tcb.tcb_suspend().unwrap();
}
```

客户端协程函数

```rust
async fn client_call_test(sender_id: SenderID, msg: u64) {
    unsafe {
        while MUTE_SEND_NUM > 0 {
            MUTE_SEND_NUM -= 1;
            let item = IPCItem::from(coroutine_get_current(), msg as u32);;
            if let Ok(_reply) = seL4_Call_with_item(&sender_id, &item).await {

            } else {
                panic!("client test fail!")
            }
        }
    }
}
```

异步系统调用

```rust
pub fn async_syscall_test(bootinfo: &sel4::BootInfo) -> sel4::Result<!> {
    debug_println!("Enter Async Syscall Test");
    runtime_init();
    debug_println!("exec addr :{}", get_executor_ptr());
    crate::device::taic::taic_init(bootinfo);
    // crate::device::init(bootinfo);
    //buffer
    let new_buffer_layout = Layout::from_size_align(size_of::<NewBuffer>(), 4096)
        .expect("Failed to create layout for page aligned memory allocation");
    let new_buffer_ref = unsafe {
        let ptr = alloc_zeroed(new_buffer_layout);
        if ptr.is_null() {
            panic!("Failed to allocate page aligned memory");
        }
        &mut *(ptr as *mut NewBuffer)
    };
    let new_buffer_ptr: usize = new_buffer_ref as *const NewBuffer as usize;
    // debug_println!("async_syscall_test: new_buffer_ptr vaddr: {:#x}", new_buffer_ptr);
    // debug_println!("async_syscall_test: new_buffer_ptr paddr: {:#x}", UserImageUtils.get_user_image_frame_paddr(new_buffer_ptr));

    let obj_allocator = unsafe { &GLOBAL_OBJ_ALLOCATOR };
    //ntfn
    let unbadged_reply_ntfn = obj_allocator.lock().alloc_ntfn().unwrap();
    //初始化cspace
    let badged_reply_ntfn = sel4::BootInfo::init_cspace_local_cptr::<sel4::cap_type::Notification>(
        obj_allocator.lock().get_empty_slot(),
    );
    debug_println!("async_syscall_test: spawn recv_reply_coroutine");

    let cid = coroutine_spawn(Box::pin(recv_reply_coroutine_async_syscall(
        new_buffer_ptr,
        REPLY_NUM,
    )));
    debug_println!("async_syscall_test: cid: {:?}", cid);
    let badge = register_recv_cid(&cid).unwrap() as u64;
    let cnode = sel4::BootInfo::init_thread_cnode();
    cnode
        .relative(badged_reply_ntfn)
        .mint(
            &cnode.relative(unbadged_reply_ntfn),
            sel4::CapRights::write_only(),
            badge,
        )
        .unwrap();

    let recv_tcb = sel4::BootInfo::init_thread_tcb();
    recv_tcb.tcb_bind_notification(unbadged_reply_ntfn)?;
    // register_receiver(recv_tcb, unbadged_reply_ntfn, uintr_handler as usize)?;
    alloc_receiver(recv_tcb, unbadged_reply_ntfn, 0);

    register_async_syscall_buffer(new_buffer_ptr);

    let new_buffer_cap =
        CPtr::from_bits(UserImageUtils.get_user_image_frame_slot(new_buffer_ptr) as u64);
    // debug_println!("async_syscall_test: new_buffer_cap: {}, new_buffer_ptr: {:#x}", new_buffer_cap.bits(), new_buffer_ptr);
    badged_reply_ntfn.register_async_syscall(new_buffer_cap)?;

    // 输出类系统调用演示
    // coroutine_spawn(Box::pin(test_async_output_section(new_buffer_ptr)));

    // Notification机制类类系统调用演示
    // coroutine_spawn(Box::pin(test_async_notification_section(obj_allocator)));

    // 内存映射类系统调用演示
    // show_error_async_riscv_page_map();
    // test_sync_riscv_page_map(obj_allocator);
    // coroutine_spawn(Box::pin(test_async_riscv_page_section(obj_allocator)));
    // 内存映射类Unmap系统调用演示
    // test_sync_riscv_page_unmap(obj_allocator);
    // coroutine_spawn(Box::pin(test_async_riscv_page_unmap(obj_allocator)));

    // 功能测试请将此行解除注释
    // coroutine_run_until_complete();

    // 传入参数表示is_sync，输入true测试同步系统调用，输入false测试异步系统调用
    // run_performance_test(false);
    run_performance_test_all();

    debug_println!("TEST PASS");
    sel4::BootInfo::init_thread_tcb().tcb_suspend()?;
    unreachable!()
}
```

