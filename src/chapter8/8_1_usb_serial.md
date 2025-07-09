# 8.1 USB转串口驱动情境分析

USB转串口测试过程请阅读此文档：[USB转串口测试流程](https://github.com/Jasonhonghh/arceos_experiment/blob/usb-camera-base/doc/USB_TO_SERIAL/README.md)

#### usb协议栈的初始化

1. **把usb协议栈编译进内核空间**。编译的命令是`make A=apps/usb-hid ...`，在`usb-hid`程序的配置文件`Cargo.toml`命令里指定了要依赖`driver_usb`，这样就决定了会把USB协议栈`driver_usb`编译进内核空间。

2. **初始化XHCI控制器**。在`usb-hid`的`main`函数会执行`USBSystem::new().init()`，这样就会进一步调用`USBHostSystem::init()`初始化XHCI控制器。执行流程如下：

   ```rust
   // host/data_structures/host_controllers/xhci/mod.rs
   fn init(&mut self) -> &mut Self {
       // 1. 硬件重置XHCI控制器
       self.chip_hardware_reset()
       // 2. 设置设备槽数量与内存映射
       .set_max_device_slots()
       .set_dcbaap()  // 设置设备上下文基地址
       // 3. 配置命令环与事件环
       .set_cmd_ring()
       .init_ir()      // 初始化中断
       // 4. 分配 scratchpad 缓冲区
       .setup_scratchpads()
       // 5. 启动控制器
       .start()
       .test_cmd()     // 测试命令环
       .reset_ports(); // 重置所有USB端口
       self
   }
   ```

3. **XHCI控制器检测USB设备的接入**。使用`probe()`方法扫描所有USB端口，检测物理连接的设备。执行流程如下：

   ```rust
   // host/data_structures/host_controllers/xhci/mod.rs
   fn probe(&mut self) -> Vec<usize> {
       let mut founded = Vec::new();
       let port_len = self.regs.port_register_set.len();
       // 1. 检查所有端口的连接状态
       for i in 0..port_len {
           let portsc = self.regs.port_register_set.read_volatile_at(i).portsc;
           if portsc.current_connect_status() {
               founded.push(i);
           }
       }
       // 2. 为每个连接的设备分配slot ID并初始化
       for port_idx in founded {
           let slot_id = self.device_slot_assignment();       // 分配slot ID
           self.dev_ctx.new_slot(slot_id as usize, 0, port_idx + 1, 32);
           self.address_device(slot_id, port_idx + 1);       // 分配设备地址
           self.set_ep0_packet_size(slot_id, self.control_fetch_control_point_packet_size(slot_id));
           founded.push(slot_id);
       }
       founded
   }
   ```

4. **获取USB设备的设备描述符**。

   ```rust
   // host/data_structures/host_controllers/xhci/mod.rs
   fn control_fetch_control_point_packet_size(&mut self, slot_id: usize) -> u8 {
       // 发送GET_DESCRIPTOR请求获取设备描述符
       let mut buffer = DMA::new_vec(0u8, 8, 64, self.config.lock().os.dma_alloc());
       self.control_transfer(
           slot_id,
           ControlTransfer {
               request_type: bmRequestType::new(
                   Direction::In,
                   DataTransferType::Standard,
                   Recipient::Device,
               ),
               request: bRequest::GetDescriptor,
               value: DescriptorType::Device.forLowBit(0).bits(),
               data: Some((buffer.addr() as usize, buffer.length_for_bytes())),
           },
       ).unwrap();
       // 解析描述符获取最大包大小
       let data = buffer.to_vec();
       data.last().copied().unwrap_or(8)
   }
   ```

5. **USB驱动的匹配**。在`usb-hid`的`main`函数会执行`USBSystem::new().init()`，这样也会进一步调用`USBDriverSystem::init()`注册所有的USB驱动模块。例如`CdcSerialDriverModule`的`should_active()`方法，其主要功能是判断是否应该激活 CDC（通用串行总线控制设备类）串口驱动。具体步骤如下：

   > 1. **设备描述符检查**：检查传入的独立设备实例的描述符是否已初始化。
   > 2. **设备类、厂商 ID 和产品 ID 检查**：如果描述符已初始化，获取设备的类、厂商 ID 和产品 ID，判断是否为特定的厂商和产品（厂商 ID 为 `0x1a86`，产品 ID 为 `0x7523`，对应沁恒电子的 CH340 串口转换器）。
   > 3. **端点信息收集**：若匹配成功，收集设备的端点信息。
   > 4. **驱动初始化**：使用收集到的信息初始化 `CdcSerialDriver` 实例，并将其封装在 `Option<Vec<Arc<SpinNoIrq<dyn USBSystemDriverModuleInstance<'a, O>>>>` 中返回。
   > 5. **不匹配处理**：若不匹配，则返回 `None`。

6. **驱动实例的创建**。匹配成功后创建驱动实例，调用`new_and_init`方法初始化设备功能。例如`CdcSerialDriver`的`new_and_init`方法，其具体的初始化过程如下：

   > 1. **为读缓冲区分配空间**。
   > 2. **查找输入端点和输出端点**。
   > 3. **创建写缓冲区并对其进行初始化**。
   > 4. **创建控制缓冲区和状态缓冲区**。
   > 5. **创建CdcSerialDriver实例**。

7. **驱动实例初始化设备**。在`usb-hid`的`main`函数会执行`USBSystem::new().init().init_probe()`，驱动实例最终就可以调用`prepare_for_drive`方法通过 URB 请求配置设备接口、端点等。例如`CdcSerialDriver`的`prepare_for_drive`方法，其具体的初始化过程如下：

   > 1. **初始化一个空向量todo_list以保存URB**。
   > 2. **设置配置描述符**。
   > 3. **厂商自定义配置流程**。这些代码创建了多个控制传输的URB，包括输出和输入操作，涉及不同的命令（如CH341_CMD_C3, CH341_CMD_C1, CH341_CMD_W, CH341_CMD_R等）、索引和值，这些操作参考了Linux源码里的drivers/usb/serial/ch341.h中的`ch341_configure`函数以及wireshark抓包数据。
   > 4. **配置115200波特率**。创建了一系列控制传输的URB，用于将设备的波特率配置为115200。
   > 5. **返回配置所需的URB列表**。将todo_list封装在some中返回。

8. **通过事件系统通知初始化完成**。驱动初始化完成后通过事件总线通知上层系统：

   ```rust
   // usb系统主循环
   pub fn drive_all(mut self) -> Self {
       loop {
           let tick = self.usb_driver_layer.tick();  // 收集URB请求
           if tick.len() != 0 {
               self.host_driver_layer.tock(tick);  // 处理URB并获取UCB
           }
       }
   }
   ```


#### 用户向串口发送数据

1. **用户把数据写入发送缓冲区**。用户需要调用`CdcSerialDriver`的`write`方法，把数据写入发送缓冲区。当前的测试流程，是在`CdcSerialDriver`的`new_and_init`方法里，把`hello,world`硬写入发送缓冲区。

2. **生成URB并发送**。`gather_urb`方法会检查`write_data_bnuffer`队列，若队列中有数据，就生成用于批量传输的URB并发送。

   ```rust
       fn gather_urb(&mut self) -> Option<Vec<crate::usb::urb::URB<'a, O>>> {
           // 总的逻辑是遍历写入数据缓冲区，如果有数据就发送，如果没有数据就接收
           let mut todo_list = Vec::new();
   
           if self.write_data_buffer.len() > 0 {
               // 如果写入数据缓冲区有数据，就发送数据
               let write_buffer = if let Some(buffer) = self.write_data_buffer.front() {
                   buffer
               } else {
                   panic!("write_data_buffer is empty, but write_data_buffer.len() > 0");
               };
   
               todo_list.push(URB::new(
                   self.device_slot_id,
                   RequestedOperation::Bulk(BulkTransfer {
                       endpoint_id: self.out_endpoint as usize,
                       buffer_addr_len: write_buffer.lock().addr_len_tuple(),
                   }),
               ));
               self.driver_state_machine = StateMachine::Writing;
               Some(todo_list)
           } else {
               // ... 接收数据逻辑 ...
           }
       }
   ```

3. **USB转串口转换器转换并发送数据**。USB 转串口驱动把 URB 封装成符合 USB 协议的数据包，经 USB 总线传给 USB 转串口转换器。转换器解析数据包，将数据转为串口信号（如 UART 信号），再通过串口引脚发送出去。

#### 串口数据传给用户

1. **USB 转串口转换器接收并转换信号**。USB 转串口转换器检测到串口引脚的信号变化，按串口协议解码信号，提取原始数据，接着封装成 USB 数据包，通过 USB 总线传给计算机。

2. **生成URB并发送**。`gather_urb`方法生成用于接收数据的URB。

   ```rust
       fn gather_urb(&mut self) -> Option<Vec<crate::usb::urb::URB<'a, O>>> {
           // ... 发送数据逻辑 ...
   
           } else {
               // 如果写入数据缓冲区没有数据，就接收数据
               todo_list.push(URB::new(
                   self.device_slot_id,
                   RequestedOperation::Bulk(BulkTransfer {
                       endpoint_id: self.in_endpoint as usize,
                       buffer_addr_len: self.read_data_buffer.as_ref().unwrap().lock().addr_len_tuple(),
                   }),
               ));
               self.driver_state_machine = StateMachine::Reading;
               Some(todo_list)
           }
       }
   ```

3.  **处理接收完成事件**。USB 转串口驱动收到数据后，触发接收完成事件，调用 `CdcSerialDriver` 的 `receive_complete_event` 方法。该方法把接收到的数据存储在`accepted_date`中。

4.  **用户应用程序接收数据**。如果是进行测试，可以在Arceos的日志中看到`accepted data: xxx`，其中`xxx`为串口接收到的数据，这表明串口数据接收成功。如果用户需要串口数据，则应用程序应从`accepted_data`读取数据。