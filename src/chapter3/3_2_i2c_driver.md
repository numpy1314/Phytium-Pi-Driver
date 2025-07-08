

# 3.2 I2C驱动开发

## I2C介绍

I2C是一种多主机、两线制、低速串行通信总线，广泛用于微控制器和各种外围设备之间的通信。它使用两条线路：串行数据线（SDA）和串行时钟线（SCL）进行双向传输。

**核心特性：**

1. 两线制：I2C 通信只需两条线路：
   - SDA（Serial Data Line）：数据线，用于传输数据。
   - SCL（Serial Clock Line）：时钟线，由主设备控制，用于同步所有设备上的数据传输。
2. 多主机和多从机：I2C 总线允许有多个主设备（可以发起通信）和多个从设备（响应主设备的请求）。这使得多个控制器可以管理不同的从设备，增加了系统的灵活性。
3. 地址分配：每个 I2C 设备都通过一个唯一的地址进行识别。这些地址在设备通信时使用，确保数据包被正确发送到指定的设备。
4. 简单的连接：由于通信线路数量少，I2C 设备通常易于安装和配置，减少了硬件布局的复杂性。
5. 同步串行通信：数据传输是同步进行的，意味着数据传输由时钟信号控制，提高了数据传输的可靠性。

**数据传输模式：**

I2C 支持多种数据传输模式，包括标准模式（100kbps）、快速模式（400kbps）、快速模式加（1Mbps）和高速模式（3.4Mbps）。根据需要选择不同的速率，可以平衡通信速度和系统资源消耗。

**应用场景：**

I2C 协议在嵌入式系统中非常流行，适用于各种应用，如：

- 传感器读取：温度、湿度、压力等传感器经常通过 I2C 与微控制器通信。
- 设备控制：在许多小型设备或计算机系统中，如笔记本电脑，I2C 用于调节音量、亮度、电源管理等。
- 存储设备：某些类型的 EEPROM 和其他存储设备通过 I2C 接口与主控制器通信。

## 在普通单片机上的I2C通信可以简化为如下步骤

- SCL为高电平的时候，SDA由高电平变化到低电平，表示开始传输数据.
- SCL变低电平，SDA开始不断高低电平表示逻辑0和1来发送数据，共8次.
- SCL变高，从设备用SDA返回低电平0，则是ACK，表示发送成功，如果SDA是高电平1，则是NACK，表示没发送成功，
- SCL又变低电平，SDA继续发送数据.
- 当SCL变高电平的时候，SDA如果由低电平变高电平，则结束.

## ArceOS 的 I2C 驱动实现

MIO是一个包含多种控制器功能的多路选择控制器,飞腾派的每个MIO均可单独当做UART/I2C。端口功能的选择，可以通过配置creg_mio_func_sel寄存器来实现，配置为00选择I2C，配置为01选择UART。

```
由MIO控制器来当作IIC来与设备通信，操作会比普通单片机中用GPIO口模拟iic时序要复杂
```

由MIO控制的I2C操作说明：

### 1.初始化

#### 1.1 飞腾派 I/O 引脚初始化：

初始化I/Opad引脚寄存器，FIOPadConfig是一个配置实例，提供配置信息，包括其基地址和设备ID号，为MIO的初始化做铺垫，

```rust
#![no_std]
#![no_main]
use super::driver_mio::{mio, mio_g, mio_hw, mio_sinit};
use super::{i2c, i2c_hw, i2c_intr, i2c_master, i2c_sinit, io};
use core::ptr;
use core::ptr::write_volatile;
use __private_api::Value;
use log::*;

use crate::driver_iic::i2c::*;
use crate::driver_iic::i2c_hw::*;
use crate::driver_iic::i2c_intr::*;
use crate::driver_iic::i2c_master::*;
use crate::driver_iic::i2c_sinit::*;


use crate::driver_mio::mio::*;
use crate::driver_mio::mio_g::*;
use crate::driver_mio::mio_hw::*;
use crate::driver_mio::mio_sinit::*;

pub fn write_reg(addr: u32, value: u32) {
    //debug!("Writing value {:#X} to address {:#X}", value, addr);
    unsafe {
        *(addr as *mut u32) = value;
    }
}

pub fn read_reg(addr: u32) -> u32 {
    let value:u32;
    unsafe {
        value = *(addr as *const u32);
    }
    //debug!("Read value {:#X} from address {:#X}", value, addr);
    value
}

pub fn input_32(addr: u32, offset: usize) -> u32 {
    let address: u32 = addr + offset as u32;
    read_reg(address)
}

pub fn output_32(addr: u32, offset: usize, value: u32) {
    let address: u32 = addr + offset as u32;
    write_reg(address, value);
}

#[derive(Debug, Clone, Copy, Default)]
pub struct FIOPadConfig {
    pub instance_id: u32,    // 设备实例 ID
    pub base_address: usize, // 基地址
}

#[feature(const_trait_impl)]
#[derive(Debug, Clone, Copy, Default)]
pub struct FIOPadCtrl {
    pub config: FIOPadConfig, // 配置
    pub is_ready: u32,        // 设备是否准备好
}

pub static mut iopad_ctrl:FIOPadCtrl = FIOPadCtrl{
    config:FIOPadConfig{
        instance_id: 0,    
        base_address: 0, 
    },
    is_ready:0,
};


static FIO_PAD_CONFIG_TBL: [FIOPadConfig; 1] = [FIOPadConfig {
    instance_id: 0,
    base_address: 0x32B30000usize,
}];

pub fn FIOPadCfgInitialize(instance_p: &mut FIOPadCtrl, input_config_p: &FIOPadConfig) -> bool {
    assert!(Some(instance_p.clone()).is_some(), "instance_p should not be null");
    assert!(
        Some(input_config_p.clone()).is_some(),
        "input_config_p should not be null"
    );
    let mut ret: bool = true;
    if instance_p.is_ready == 0x11111111u32 {
        debug!("Device is already initialized.");
    }
    // Set default values and configuration data
    FIOPadDeInitialize(instance_p);
    instance_p.config = *input_config_p;
    instance_p.is_ready = 0x11111111u32;
    ret
}

pub fn FIOPadDeInitialize(instance_p: &mut FIOPadCtrl) -> bool {
    // 确保 `instance_p` 不为 null，类似于 C 中的 `FASSERT(instance_p)`
    if instance_p.is_ready == 0 {
        return true;
    }

    // 标记设备为未准备好
    instance_p.is_ready = 0;

    // 清空设备数据
    unsafe {
        core::ptr::write_bytes(instance_p as *mut FIOPadCtrl, 0, size_of::<FIOPadCtrl>());
    }

    true
}

pub fn FIOPadLookupConfig(instance_id: u32) -> Option<FIOPadConfig> {
    if instance_id as usize >= 1 {
        // 对应 C 代码中的 FASSERT 语句
        return None;
    }

    for config in FIO_PAD_CONFIG_TBL.iter() {
        if config.instance_id == instance_id {
            return Some(*config);
        }
    }

    None
}
```

#### 1.2 MIO 控制器初始化

先对MIO进行初始化配置，包括功能寄存器地址，MIO寄存器地址以及中断编号
按照之前配置的I/Opad引脚寄存器，来设置 I2C 的 SCL 和 SDA 引脚功能。
设置MIO配置，包括 ID号、MIO基地址、中断号、频率、设备地址 和 传输速率。(这些由FMioLookupConfig来提供不同MIO的配置信息)

```rust
pub unsafe  fn FI2cMioMasterInit(address: u32, speed_rate: u32) -> bool {
    let mut input_cfg: FI2cConfig = FI2cConfig::default();
    let mut config_p: FI2cConfig = FI2cConfig::default();
    let mut status: bool = true;
    // MIO 初始化
    master_mio_ctrl.config = FMioLookupConfig(1).unwrap();
    status = FMioFuncInit(&mut master_mio_ctrl, 0b00);
    if status != true {
        debug!("MIO initialize error.");
        return false;
    }
    FIOPadSetFunc(&iopad_ctrl, 0x00D0u32, 5); /* scl */
    FIOPadSetFunc(&iopad_ctrl, 0x00D4u32, 5); /* sda */
    unsafe {
        core::ptr::write_bytes(&mut master_i2c_instance as *mut FI2c, 0, size_of::<FI2c>());
    }
    // 查找默认配置
    config_p = FI2cLookupConfig(1).unwrap(); // 获取 MIO 配置的默认引用
    if !Some(config_p).is_some() {
        debug!("Config of mio instance {} not found.", 1);
        return false;
    }
    // 修改配置
    input_cfg = config_p.clone();
    input_cfg.instance_id = 1;
    input_cfg.base_addr = FMioFuncGetAddress(&master_mio_ctrl, 0b00);
    input_cfg.irq_num = FMioFuncGetIrqNum(&master_mio_ctrl, 0b00);
    input_cfg.ref_clk_hz = 50000000;
    input_cfg.slave_addr = address;
    input_cfg.speed_rate = speed_rate;
    // 初始化
    status = FI2cCfgInitialize(&mut master_i2c_instance, &input_cfg);
    // 处理 FI2C_MASTER_INTR_EVT 中断的回调函数
    master_i2c_instance.master_evt_handlers[0 as usize] = None;
    master_i2c_instance.master_evt_handlers[1 as usize] = None;
    master_i2c_instance.master_evt_handlers[2 as usize] = None;
    if status != true {
        debug!("Init mio master failed, ret: {:?}", status);
        return status;
    }
    debug!(
        "Set target slave_addr: 0x{:x} with mio-{}",
        input_cfg.slave_addr, 1
    );
    status
}
```

#### 1.3 初始化 I2C 配置

I2C配置中包含MIO配置，具体可以见结构体FI2c。首先检查设备是否已经初始化，防止重复初始化。如果设备没有初始化，它会进行去初始化操作，设置设备的配置数据，然后重置设备，最后将设备标记为已就绪状态

```rust
pub fn FI2cCfgInitialize(instance_p: &mut FI2c, input_config_p: &FI2cConfig) -> bool {
    assert!(Some(instance_p.clone()).is_some() && Some(input_config_p).is_some());

    let mut ret = true;

    // 如果设备已启动，禁止初始化并返回已启动状态，允许用户取消初始化设备并重新初始化，但防止用户无意中初始化
    if instance_p.is_ready == 0x11111111u32 {
        debug!("Device is already initialized!!!");
        return false;
    }

    // 设置默认值和配置数据，包括将回调处理程序设置为存根，以防应用程序未分配自己的回调而导致系统崩溃
    FI2cDeInitialize(instance_p);
    instance_p.config = *input_config_p;

    // 重置设备
    ret = FI2cReset(instance_p);
    if ret == true {
        instance_p.is_ready = 0x11111111u32;
    }

    ret
}
```

最后返回到MIO控制器初始化中，初始化I2C设备中的中断函数

### 2.收发数据

#### 2.1 发送数据

```rust
pub unsafe fn FI2cMasterWrite(buf_p: &mut [u8], buf_len: u32, inchip_offset: u32) -> bool {
    let mut status: bool = true;

    if buf_len < 256 && inchip_offset < 256 {
        if (256 - inchip_offset) < buf_len {
            debug!("Write to eeprom failed, out of eeprom size.");
            return false;
        }
    } else {
        debug!("Write to eeprom failed, out of eeprom size.",);
        return false;
    }

    status = FI2cMasterWritePoll(&mut master_i2c_instance, inchip_offset, 1, buf_p, buf_len);
    //debug!("++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++");
    if status != true {
        debug!("Write to eeprom failed");
    }

    status
}

```

这个发送数据函数接收一个buffer用来装要发送的数据，由于是[u8]，只有一个u8大小的值，所以buf_len也是1(这份数据只发一次，不重复发)。还接收一个inchip_offset,指从设备内部的偏地址
先通过FI2c对象来判断是否准备好，工作模式是否为主模式 然后启动I2C主设备传输，可以看这里

```rust
pub fn FI2cMasterStartTrans(
    instance_p: &mut FI2c,
    mem_addr: u32,
    mem_byte_len: u8,
    flag: u16,
) -> bool {
    assert!(Some(instance_p.clone()).is_some());
    let base_addr = instance_p.config.base_addr;
    let mut addr_len: u32 = mem_byte_len as u32;
    let mut ret = true;

    ret = FI2cWaitBusBusy(base_addr.try_into().unwrap());
    if ret != true {
        return ret;
    }
    ret = FI2cSetTar(base_addr.try_into().unwrap(), instance_p.config.slave_addr);

    while addr_len > 0 {
        if FI2cWaitStatus(base_addr.try_into().unwrap(), (0x1 << 1)) != true {
            break;
        }
        if input_32(base_addr.try_into().unwrap(), 0x80) != 0 {
            return false;
        }
        if input_32(base_addr.try_into().unwrap(), 0x70) & (0x1 << 1) != 0 {
            addr_len -= 1;
            let value = (mem_addr >> (addr_len * 8)) & FI2C_DATA_MASK();
            if addr_len != 0 {
                output_32(base_addr.try_into().unwrap(), 0x10, value);
            } else {
                output_32(base_addr.try_into().unwrap(), 0x10, value + flag as u32);
            }
        }
    }
    ret
}
```

随后在FIFO不满的情况下，向0x10（IC_DATA_CMD）的bit[7:0]写数据，bit[8]写0表示写以外，向bit[9]写1表示停止。

#### 2.2 接收数据

```rust
pub unsafe fn FI2cMasterRead(buf_p: &mut [u8], buf_len: u32, inchip_offset: u32) -> bool {
    let mut instance_p: FI2c = master_i2c_instance;
    let mut status: bool = true;

    assert!(buf_len != 0);

    for i in 0..buf_len as usize {
        buf_p[i] = 0;
    }

    status = FI2cMasterReadPoll(&mut instance_p, inchip_offset, 1, buf_p, buf_len);

    status
}
```

和发送数据函数一样，接收数据函数接收一个buffer用来装接收到的u8大小的数据，以及buf_len，inchip_offset
先通过FI2c对象来判断是否准备好，工作模式是否为主模式
然后启动I2C主设备传输。
发送读数据命令：向0x10（IC_DATA_CMD）bit[8]写1，表示命令为读操作.
在FIFO不空的情况，读取数据，读最后一个字节数据时要加上停止信号，即除了向0x10（IC_DATA_CMD）的bit[8]仍写1表示读以外，向bit[9]写1表示停止。 最后停止I2C传输，见

```rust
pub fn FI2cMasterStopTrans(instance_p: &mut FI2c) -> bool {
    assert!(Some(instance_p.clone()).is_some());
    let mut ret = true;
    let base_addr = instance_p.config.base_addr;
    let mut reg_val = 0;
    let mut timeout = 0;

    while true {
        if input_32(base_addr.try_into().unwrap(), 0x34) & (0x1 << 9) != 0 {
            reg_val = input_32(base_addr.try_into().unwrap(), 0x60);
            break;
        } else if 500 < timeout {
            break;
        }
        timeout += 1;
        busy_wait(Duration::from_millis(1));
    }

    ret = FI2cWaitBusBusy(base_addr.try_into().unwrap());
    if ret == true {
        ret = FI2cFlushRxFifo(base_addr.try_into().unwrap());
    }
    ret
}
```

飞腾派 I2C 寄存器（待整理）：

| 寄存器         | 偏移 | 描述                                     |
| -------------- | ---- | ---------------------------------------- |
| IC_CON         | 0x00 | I2C控制寄存器                            |
| IC_TAR         | 0x04 | I2C主机地址寄存器                        |
| IC_SAR         | 0x08 | I2C从机地址寄存器                        |
| IC_HS_MADDR    | 0x0C | I2C高速主机模式编码地址寄存器            |
| IC_DATA_CMD    | 0x10 | I2C数据寄存器                            |
| IC_SS_SCL_HCNT | 0x14 | 标准模式I2C时钟信号SCL的高电平计数寄存器 |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |
|                |      |                                          |

飞腾派 MIO 寄存器基地址：

| Name  | Offset          |
| ----- | --------------- |
| MIO0  | 0x000_2801_4000 |
| MIO1  | 0x000_2801_6000 |
| MIO2  | 0x000_2801_8000 |
| MIO3  | 0x000_2801_A000 |
| MIO4  | 0x000_2801_C000 |
| MIO5  | 0x000_2801_E000 |
| MIO6  | 0x000_2802_0000 |
| MIO7  | 0x000_2802_2000 |
| MIO8  | 0x000_2802_4000 |
| MIO9  | 0x000_2802_6000 |
| MIO10 | 0x000_2802_8000 |
| MIO11 | 0x000_2802_A000 |
| MIO12 | 0x000_2802_C000 |
| MIO13 | 0x000_2802_E000 |
| MIO14 | 0x000_2803_0000 |
| MIO15 | 0x000_2803_2000 |
