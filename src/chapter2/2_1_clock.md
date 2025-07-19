# 2.2 时钟源驱动
时钟是嵌入式系统的"心跳"，它驱动着系统的运行和数据的流动，同时影响着系统的能耗．因此，外设的时钟以一种树状的形式在SoC中被组织起来，我们称之为"时钟树"，每个"节点"都代表着具有不同频率或性质的时钟源．

根据时钟源的性质，除了某些特定的时钟源，大部分设备的时钟源在Linux内核中可以被分成6大类：
+ fixed-clock: 表示频率固定，不可更改的时钟源．在设备树或驱动中，其只需指定频率，无需动态调整．
+ fixed-factor-clock: 表示由一个父时钟通过固定的倍频（乘法因子）和分频（除法因子）得到的时钟源。它的频率由父时钟频率乘以一个固定的分子再除以一个固定的分母，不能动态调整，只能在设备树或驱动中静态配置．
+ divider-clock: 表示通过对父时钟进行分频（除法因子）得到的时钟源。它的输出频率等于父时钟频率除以一个可配置的分频值。通常用于降低时钟频率以满足外设需求，分频值可以在驱动或设备树中配置，有些情况下支持动态调整.
+ gate-clock: 表示可以通过使能或关闭来控制时钟信号输出的时钟源。它通常用于控制外设的时钟开关，实现节能或功能管理。`gate-clock` 只负责时钟的开关，不改变时钟频率，相关配置可在驱动或设备树中完成．
+ mux-clock: 表示可以在多个父时钟之间选择一个作为输出源的时钟节点。它通过切换父时钟，实现时钟源的灵活选择，适用于需要根据不同工作模式或性能需求动态切换时钟源的场景。相关配置可在驱动或设备树中完成．
+ composite-clock: `composite-clock` 是 Linux 时钟框架中的一种复合型时钟节点，集成了分频（divider）、选择（mux）和开关（gate）等多种功能。它可以灵活地组合这些功能，实现复杂的时钟控制需求。`composite-clock` 适用于需要同时支持时钟源切换、分频和使能控制的场景，相关配置可在驱动或设备树中完成．

在linux中，时钟的管理依赖于[Common Clk Framework](https://docs.kernel.org/driver-api/clk.html)．CCF将这些时钟设备的共性抽象出来，使用`struct clk_hw`来表示
```rust
/**
 * struct clk_hw - handle for traversing from a struct clk to its corresponding
 * hardware-specific structure.  struct clk_hw should be declared within struct
 * clk_foo and then referenced by the struct clk instance that uses struct
 * clk_foo's clk_ops
 *
 * @core: pointer to the struct clk_core instance that points back to this
 * struct clk_hw instance
 *
 * @clk: pointer to the per-user struct clk instance that can be used to call
 * into the clk API
 *
 * @init: pointer to struct clk_init_data that contains the init data shared
 * with the common clock framework. This pointer will be set to NULL once
 * a clk_register() variant is called on this clk_hw pointer.
 */
struct clk_hw {
	struct clk_core *core;
	struct clk *clk;
	const struct clk_init_data *init;
};
```
