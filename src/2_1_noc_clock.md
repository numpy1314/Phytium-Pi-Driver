# 2.1 NOC时钟驱动

## generic timer介绍
generic timer arm公司提出的一种硬件上的设计框架。在早期单核时代，每个SOC vendor厂商购买arm core的IP，然后自己设计soc上的timers。这就导致了对于每一种soc，驱动代码都会不同。此外，linux对硬件timer也有一定的要求，有些硬件厂商设计的timer支持的不是很好。在进入多核时代之后，核与核之间有同步的需求，为单核设计timer的方案会变的更加复杂。由此，arm公司提供了timer的硬件设计，集成在了自己的多核结构中。也就是[generic_timer](https://developer.arm.com/documentation/102379/0104/What-is-the-Generic-Timer-).

对于软件开发者来说，总体框架如下：

非虚拟化：

![system_counter](img/2_1_system_counter.png)

虚拟化：

![system_counter_virtual](img/2_1_system_counter_virtual.png)

每一个核有单独的timer，以及一个全局的system counter，它们通过system time bus相连。此外，对于虚拟化场景，还额外增加了CNTVOFF这个寄存器，主要作用是对于让虚拟机感受不到非调度时间的流逝。对于这个架构保证了对于所有的核来说，timer的clock都是一致的。产生的中断位于gic的ppi区，产生的中断会分发到对应的核上。

## 主要寄存器介绍
- CNTFRQ_EL0: EL1级别物理system counter时钟的频率，对于EL1及以上，这个寄存器是只读的。
- CNTP_CTL_EL0： EL1级别物理timer的控制器，用于开启/关闭这个核的timer中断
- CNTP_TVAL_EL0：EL1级别物理timer的时钟值，当system counter值 >= 这个值，会产生一个可能的中断
- CNTPCT_EL0：EL1级别物理timer的比较值，当这个值被写时，会将 system_count + CNTPCT_EL0 写入 CNTP_TVAL_EL0。

## 多核时钟驱动实验步骤
- timer 初始化代码
  - main core获取 system counter的 frequency
  ```rust
    /// Early stage initialization: stores the timer frequency.
    pub(crate) fn init_early() {
        let freq = CNTFRQ_EL0.get();
        unsafe {
            // crate::time::NANOS_PER_SEC = 1_000_000_000;
            CNTPCT_TO_NANOS_RATIO = Ratio::new(crate::time::NANOS_PER_SEC as u32, freq as u32);
            NANOS_TO_CNTPCT_RATIO = CNTPCT_TO_NANOS_RATIO.inverse();
        }
    }
  ```
- 每个核都开启timer中断，注册timer回调
  ```rust
    /// Set a one-shot timer.
    ///
    /// A timer interrupt will be triggered at the specified monotonic time deadline (in nanoseconds).
    #[cfg(feature = "irq")]
    pub fn set_oneshot_timer(deadline_ns: u64) {
        let cnptct = CNTPCT_EL0.get();
        let cnptct_deadline = nanos_to_ticks(deadline_ns);
        if cnptct < cnptct_deadline {
            let interval = cnptct_deadline - cnptct;
            debug_assert!(interval <= u32::MAX as u64);
            CNTP_TVAL_EL0.set(interval);
        } else {
            CNTP_TVAL_EL0.set(0);
        }
    }
    #[cfg(feature = "irq")]
    fn init_interrupt() {
        use axhal::time::TIMER_IRQ_NUM;

        // Setup timer interrupt handler
        const PERIODIC_INTERVAL_NANOS: u64 =
            axhal::time::NANOS_PER_SEC / axconfig::TICKS_PER_SEC as u64;

        #[percpu::def_percpu]
        static NEXT_DEADLINE: u64 = 0;

        fn update_timer() {
            let now_ns = axhal::time::monotonic_time_nanos();
            // Safety: we have disabled preemption in IRQ handler.
            let mut deadline = unsafe { NEXT_DEADLINE.read_current_raw() };
            if now_ns >= deadline {
                deadline = now_ns + PERIODIC_INTERVAL_NANOS;
            }
            unsafe { NEXT_DEADLINE.write_current_raw(deadline + PERIODIC_INTERVAL_NANOS) };
            axhal::time::set_oneshot_timer(deadline);
        }

        axhal::irq::register_handler(TIMER_IRQ_NUM, || {
            update_timer();
            #[cfg(feature = "multitask")]
            axtask::on_timer_tick();
        });

        # // Enable IRQs before starting app
        # axhal::asm::enable_irqs();
    }

    pub(crate) fn init_percpu() {
        #[cfg(feature = "irq")]
        {
            CNTP_CTL_EL0.write(CNTP_CTL_EL0::ENABLE::SET);
            // 设置为0,马上就会触发一次中断。
            CNTP_TVAL_EL0.set(0);
            // gic 中断时能
            crate::platform::irq::set_enable(crate::platform::irq::TIMER_IRQ_NUM, true);
        }
    }
  ```
- 通过这个命令`make A=examples/helloworld ARCH=aarch64 PLATFORM=aarch64-qemu-virt FEATURES=irq LOG=trace ACCEL=n SMP=4 run`在qemu上运行得到下列结果
  <details>
    <summary>运行结果</summary>
        qemu-system-aarch64 -m 128M -smp 4 -cpu cortex-a72 -machine virt -kernel examples/helloworld/helloworld_aarch64-qemu-virt.bin -nographic

        d8888                            .d88888b.   .d8888b.
        d88888                           d88P" "Y88b d88P  Y88b
        d88P888                           888     888 Y88b.
        d88P 888 888d888  .d8888b  .d88b.  888     888  "Y888b.
        d88P  888 888P"   d88P"    d8P  Y8b 888     888     "Y88b.
        d88P   888 888     888      88888888 888     888       "888
        d8888888888 888     Y88b.    Y8b.     Y88b. .d88P Y88b  d88P
        d88P     888 888      "Y8888P  "Y8888   "Y88888P"   "Y8888P"

        arch = aarch64
        platform = aarch64-qemu-virt
        target = aarch64-unknown-none-softfloat
        build_mode = release
        log_level = trace
        smp = 4

        [  0.003638 axruntime:130] Logging is enabled.
        [  0.004146 axruntime:131] Primary CPU 0 started, dtb = 0x44000000.
        [  0.004338 axruntime:133] Found physcial memory regions:
        [  0.004530 axruntime:135]   [PA:0x40200000, PA:0x40207000) .text (READ | EXECUTE | RESERVED)
        [  0.004790 axruntime:135]   [PA:0x40207000, PA:0x4020a000) .rodata (READ | RESERVED)
        [  0.004938 axruntime:135]   [PA:0x4020a000, PA:0x4020e000) .data .tdata .tbss .percpu (READ | WRITE | RESERVED)
        [  0.005096 axruntime:135]   [PA:0x4020e000, PA:0x4030e000) boot stack (READ | WRITE | RESERVED)
        [  0.005232 axruntime:135]   [PA:0x4030e000, PA:0x40311000) .bss (READ | WRITE | RESERVED)
        [  0.005372 axruntime:135]   [PA:0x40311000, PA:0x48000000) free memory (READ | WRITE | FREE)
        [  0.005530 axruntime:135]   [PA:0x9000000, PA:0x9001000) mmio (READ | WRITE | DEVICE | RESERVED)
        [  0.005672 axruntime:135]   [PA:0x9100000, PA:0x9101000) mmio (READ | WRITE | DEVICE | RESERVED)
        [  0.005808 axruntime:135]   [PA:0x8000000, PA:0x8020000) mmio (READ | WRITE | DEVICE | RESERVED)
        [  0.005944 axruntime:135]   [PA:0xa000000, PA:0xa004000) mmio (READ | WRITE | DEVICE | RESERVED)
        [  0.006080 axruntime:135]   [PA:0x10000000, PA:0x3eff0000) mmio (READ | WRITE | DEVICE | RESERVED)
        [  0.006216 axruntime:135]   [PA:0x4010000000, PA:0x4020000000) mmio (READ | WRITE | DEVICE | RESERVED)
        [  0.006386 axruntime:150] Initialize platform devices...
        [  0.006506 axhal::platform::aarch64_common::gic:51] Initialize GICv2...
        [  0.007098 axhal::platform::aarch64_common::gic:27] GICD set enable: 30 true
        [  0.007574 axhal::platform::aarch64_common::gic:27] GICD set enable: 33 true
        [  0.007854 axruntime::mp:20] starting CPU 1...
        [  0.007976 axhal::platform::aarch64_common::psci:115] Starting CPU 1 ON ...
        [  0.008234 axruntime::mp:37] Secondary CPU 1 started.
        [  0.008236 axruntime::mp:20] starting CPU 2...
        [  0.008672 axhal::platform::aarch64_common::gic:27] GICD set enable: 30 true
        [  0.008768 axhal::platform::aarch64_common::psci:115] Starting CPU 2 ON ...
        [  0.008974 axruntime::mp:47] Secondary CPU 1 init OK.
        [  0.009110 axruntime::mp:37] Secondary CPU 2 started.
        [  0.009300 axruntime::mp:20] starting CPU 3...
        [  0.009550 axhal::platform::aarch64_common::gic:27] GICD set enable: 30 true
        [  0.009712 axhal::platform::aarch64_common::psci:115] Starting CPU 3 ON ...
        [  0.009934 axruntime::mp:47] Secondary CPU 2 init OK.
        [  0.010186 axruntime::mp:37] Secondary CPU 3 started.
        [  0.010218 axruntime:176] Initialize interrupt handlers...
        [  0.010548 axhal::platform::aarch64_common::gic:27] GICD set enable: 30 true
        [  0.010790 axruntime::mp:47] Secondary CPU 3 init OK.
        [  0.010818 axhal::platform::aarch64_common::gic:36] register handler irq 30
        [  0.011058 axhal::platform::aarch64_common::gic:27] GICD set enable: 30 true
        [  0.011578 axhal::irq:18] IRQ 30
        [  0.012612 axruntime:188] Primary CPU 0 init OK.
        Hello, world!
        [  0.012760 3 axhal::irq:18] IRQ 30
        [  0.012750 1 axhal::irq:18] IRQ 30
        [  0.012944 2 axhal::irq:18] IRQ 30
        [  0.024292 0 axhal::irq:18] IRQ 30
        [  0.024338 2 axhal::irq:18] IRQ 30
        [  0.024350 3 axhal::irq:18] IRQ 30
        [  0.024344 1 axhal::irq:18] IRQ 30
        [  0.034532 2 axhal::irq:18] IRQ 30
        [  0.034566 1 axhal::irq:18] IRQ 30
        [  0.034568 0 axhal::irq:18] IRQ 30
        [  0.034564 3 axhal::irq:18] IRQ 30
        [  0.043722 2 axhal::irq:18] IRQ 30
        [  0.043730 0 axhal::irq:18] IRQ 30
        [  0.043724 1 axhal::irq:18] IRQ 30
        [  0.043730 3 axhal::irq:18] IRQ 30
        [  0.053962 2 axhal::irq:18] IRQ 30
        [  0.053986 0 axhal::irq:18] IRQ 30
  </details>
- 可以看出，4个cpu的中断每10ms被触发一次，符合预期。

## 实验总结
验证了arm多核架构下的timer。

## 参考资料
[arm_generic_timer](https://developer.arm.com/documentation/102379/0104/What-is-the-Generic-Timer-)
[generic_timer_in_linux](https://cloud.tencent.com/developer/article/1518249)