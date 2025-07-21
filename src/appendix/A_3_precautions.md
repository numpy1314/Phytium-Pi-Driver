# 注意事项

在驱动开发中，缓存一致性和内存屏障是两个至关重要的概念。正确理解和处理这些问题对于编写稳定、高性能的驱动程序至关重要。

## 缓存一致性

### 高速缓存的基础知识

高速缓存（Cache）是位于CPU和主内存之间的高速存储器，用于缓解CPU和内存之间的速度差异。

#### 缓存层次结构

现代处理器通常具有多级缓存：

- **L1 缓存**：最接近CPU核心，容量小但速度最快（通常几十KB）
- **L2 缓存**：中等容量和速度（通常几百KB到几MB）
- **L3 缓存**：容量最大但相对较慢（通常几MB到几十MB）

#### 缓存行（Cache Line）

```rust
// 缓存行通常是 64 字节
const CACHE_LINE_SIZE: usize = 64;

// 结构体对齐到缓存行边界
#[repr(align(64))]
struct CacheAligned {
    data: [u8; 64],
}
```

#### 缓存映射方式

1. **直接映射**：每个内存地址只能映射到特定的缓存行
2. **全相联映射**：任何内存地址可以映射到任何缓存行
3. **组相联映射**：直接映射和全相联映射的折中

### 高速缓存的共享属性

在多核系统中，缓存共享带来了复杂性

Inner share

Outer share

#### 伪共享（False Sharing）

当两个无关的变量位于同一缓存行时，会导致性能问题：

```rust
// 错误示例 - 可能导致伪共享
struct BadLayout {
    counter1: AtomicU64,  // 可能与 counter2 在同一缓存行
    counter2: AtomicU64,
}

// 正确示例 - 避免伪共享
#[repr(align(64))]
struct GoodLayout {
    counter1: AtomicU64,
    _pad1: [u8; 56],     // 填充到缓存行边界
    counter2: AtomicU64,
    _pad2: [u8; 56],
}
```

### 高速缓存的维护指令

#### 常见的缓存维护指令

1. **刷新（Flush）**：将缓存行写回内存并标记为无效
2. **无效化（Invalidate）**：标记缓存行为无效
3. **清理（Clean）**：将脏缓存行写回内存但保持有效

```rust
// 在 Rust 中使用内联汇编进行缓存操作
#[cfg(target_arch = "aarch64")]
unsafe fn flush_cache_line(addr: *const u8) {
    asm!("dc civac, {}", in(reg) addr, options(nostack));
}
```

#### 缓存维护的时机

```rust
// DMA 操作前后的缓存维护
unsafe fn dma_coherent_write(buffer: &mut [u8], device_addr: u64) {
    // 1. 清理缓存，确保数据写入内存
    for chunk in buffer.chunks(CACHE_LINE_SIZE) {
        flush_cache_line(chunk.as_ptr());
    }
    
    // 2. 启动 DMA 传输
    start_dma_transfer(buffer, device_addr);
    
    // 3. 等待传输完成
    wait_dma_complete();
    
    // 4. 无效化缓存，确保读取到最新数据
    for chunk in buffer.chunks(CACHE_LINE_SIZE) {
        invalidate_cache_line(chunk.as_ptr());
    }
}
```

### 软件维护缓存一致性

#### 显式缓存管理

```rust
pub struct CacheManager;

impl CacheManager {
    /// 为 DMA 操作准备缓冲区
    pub unsafe fn prepare_for_dma_to_device(buffer: &[u8]) {
        // 清理缓存，确保数据同步到内存
        Self::clean_dcache_range(buffer.as_ptr(), buffer.len());
    }
    
    /// DMA 从设备读取后的处理
    pub unsafe fn finish_dma_from_device(buffer: &mut [u8]) {
        // 无效化缓存，确保读取到设备写入的数据
        Self::invalidate_dcache_range(buffer.as_ptr(), buffer.len());
    }
    
    /// 双向 DMA 操作
    pub unsafe fn prepare_for_bidirectional_dma(buffer: &mut [u8]) {
        // 刷新缓存（清理 + 无效化）
        Self::flush_dcache_range(buffer.as_ptr(), buffer.len());
    }
    
    #[cfg(target_arch = "aarch64")]
    unsafe fn clean_dcache_range(start: *const u8, len: usize) {
        let end = start.add(len);
        let mut addr = (start as usize) & !(CACHE_LINE_SIZE - 1);
        
        while addr < end as usize {
            asm!("dc cvac, {}", in(reg) addr);
            addr += CACHE_LINE_SIZE;
        }
        asm!("dsb sy");
    }
    
    #[cfg(target_arch = "aarch64")]
    unsafe fn invalidate_dcache_range(start: *const u8, len: usize) {
        let end = start.add(len);
        let mut addr = (start as usize) & !(CACHE_LINE_SIZE - 1);
        
        while addr < end as usize {
            asm!("dc ivac, {}", in(reg) addr);
            addr += CACHE_LINE_SIZE;
        }
        asm!("dsb sy");
    }
    
    #[cfg(target_arch = "aarch64")]
    unsafe fn flush_dcache_range(start: *const u8, len: usize) {
        let end = start.add(len);
        let mut addr = (start as usize) & !(CACHE_LINE_SIZE - 1);
        
        while addr < end as usize {
            asm!("dc civac, {}", in(reg) addr);
            addr += CACHE_LINE_SIZE;
        }
        asm!("dsb sy");
    }
}
```

### 利用 `dma-api` 简化缓存维护操作

[dma-api](https://crates.io/crates/dma-api)

#### 一致性 DMA 内存分配

#### 使用示例

[nvme](https://crates.io/crates/nvme-driver)

## 内存屏障

### CPU 乱序执行

现代 CPU 为了提高性能，会对指令进行乱序执行，这可能导致内存访问顺序与程序代码顺序不一致。

#### 乱序执行的类型

1. **编译器重排序**：编译器优化可能改变指令顺序
2. **CPU 重排序**：CPU 在执行时可能调整指令顺序
3. **内存系统重排序**：缓存和内存控制器可能影响访问顺序

```rust
// 示例：可能被重排序的代码
static mut FLAG: bool = false;
static mut DATA: u32 = 0;

// 线程 1
unsafe fn producer() {
    DATA = 42;           // 可能被重排到 FLAG = true 之后
    FLAG = true;
}

// 线程 2
unsafe fn consumer() {
    while !FLAG {}       // 可能读取到 FLAG = true
    let value = DATA;    // 但 DATA 可能还是 0
}
```

#### 重排序规则

不同架构有不同的重排序规则：

- **x86/x64**：相对较强的内存模型，主要是 store-load 重排序
- **ARM/AArch64**：较弱的内存模型，允许更多重排序
- **RISC-V**：弱内存模型，类似 ARM

### 内存屏障的类型

#### 全屏障（Full Barrier）

mb

```rust
use std::sync::atomic::Ordering;

// 防止所有重排序
std::sync::atomic::fence(Ordering::SeqCst);
```

#### 获取屏障（Acquire Barrier）

rmb

```rust
// 防止后续操作重排到屏障之前
std::sync::atomic::fence(Ordering::Acquire);
```

#### 释放屏障（Release Barrier）

wmb

```rust
// 防止前面的操作重排到屏障之后
std::sync::atomic::fence(Ordering::Release);
```

#### 编译器屏障

```rust
// 只防止编译器重排序，不影响 CPU
std::sync::atomic::compiler_fence(Ordering::SeqCst);
```

### 如何使用内存屏障

#### 生产者-消费者模式

```rust
// 生产者-消费者模式
fn producer_consumer_example() {
    // 生产者
    unsafe {
        // 写入数据
        core::ptr::write_volatile(data_ptr, 42);
        
        // 写屏障确保数据写入完成
        wmb();
        
        // 设置标志
        core::ptr::write_volatile(flag_ptr, true);
    }
    
    // 消费者
    unsafe {
        // 读取标志
        if core::ptr::read_volatile(flag_ptr) {
            // 读屏障确保标志读取完成
            rmb();
            
            // 读取数据
            let value = core::ptr::read_volatile(data_ptr);
        }
    }
}
```

#### DMA 一致性保证

```rust
struct DmaDescriptor {
    addr: AtomicU64,
    length: AtomicU32,
    flags: AtomicU32,
}

impl DmaDescriptor {
    fn setup_transfer(&self, buffer_addr: u64, size: u32) {
        // 设置传输参数
        self.addr.write(buffer_addr);
        self.length.write(size);
        
        // 释放屏障：确保参数设置完成
        wmb();
        
        // 设置有效标志
        self.flags.write(0x80000000);
    }
    
    fn is_complete(&self) -> bool {
        // 获取屏障：确保读取到最新状态
        rmb();
        let flags = self.flags.read();
        flags & 0x40000000 != 0
    }
}
```

### 跨平台的内存屏障

[mbarrier](https://crates.io/crates/mbarrier)