

![image-20250326140431673](C:/Users/wangj/AppData/Roaming/Typora/typora-user-images/image-20250326140431673.png)

buffer的优化

```rust
pub struct ItemsIdxQueue {
    buffer: SafeRingBuffer<usize, MAX_ITEM_NUM>,
}

impl ItemsIdxQueue {
    pub fn new() -> Self {
        Self {
            buffer: SafeRingBuffer::new(),
        }
    }

    #[inline]
    pub fn write_free_idx(&mut self, idx: usize) -> Result<(), ()> {
        return self.buffer.push_safe(&idx);
    }

    #[inline]
    pub fn get_first_idx(&mut self) -> Option<usize> {
        return self.buffer.pop_safe();
    }
}

#[repr(align(4096))]
pub struct NewBuffer {
    pub recv_req_status: AtomicBool,
    pub recv_reply_status: AtomicBool,
    pub req_items: ItemsIdxQueue,
    pub res_items: ItemsIdxQueue,
    pub data: [IPCItem; MAX_ITEM_NUM],
    pub idx_allocator: IndexAllocator<4096>,
}

impl NewBuffer {
    pub fn new() -> Self {
        Self {
            recv_req_status: AtomicBool::new(false),
            recv_reply_status: AtomicBool::new(false),
            req_items: ItemsIdxQueue::new(),
            res_items: ItemsIdxQueue::new(),
            data:[IPCItem::default(); MAX_ITEM_NUM],
            idx_allocator:IndexAllocator::new()
        }
    }
    #[inline]
    pub fn get_ptr(&self) -> usize {
        self as *const Self as usize
    }

    #[inline]
    pub fn from_ptr(ptr: usize) -> &'static mut Self {
        unsafe { &mut *(ptr as *mut Self) }
    }
}

```

新的buffer：

存入

```rust
   //ipc item存入buffer
    let idx = match new_buffer.idx_allocator.allocate() {
        Some(value) => value,
        None => return Err("Failed to allocate index in buffer"),
    };
    new_buffer.data[idx] = *item;
    if new_buffer.req_items.write_free_idx(idx).is_err() {
        return Err("Failed to write free index");
    }
```

取出

```rust
Ok(new_buffer.data[idx])
```

