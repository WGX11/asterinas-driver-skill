---
name: asterinas-driver-dev
description: 给 Student 使用的 Asterinas 总驱动技能，用来总结跨题可复用的驱动写法、判断顺序、约束和反模式。何时使用：当你在实现或修改 Asterinas 驱动代码，需要知道先检查什么、什么时候这样写、为什么这样写、哪些地方最容易出错时。触发短语：Asterinas 驱动开发, virtio 驱动, 驱动写法, 队列与中断, DMA 约束.
roles: [student]
---

# Asterinas VirtIO 驱动开发指南

## ⚠️ 核心原则（必读）

1. **严格遵循标准答案结构**：不要"过度创新"或发明新的架构
2. **只修改 MASK 标记区域**：绝不删除或修改 quiz 范围外的代码
   - **关键**：即使某个模块（如 config.rs）看起来"没用"，也不要删除它
   - **关键**：mod.rs 中的 `mod config;` 声明不能删除，否则会导致依赖链断裂
3. **先参考已有实现**：在动手前，必须先阅读并理解同类驱动的标准写法
4. **PAGE_SIZE 是硬限制**：所有 DMA buffer 分配不能超过 4096 字节

**✅ 实战验证**：这套指南已通过 VirtIO Console、Input 和 Socket 三个完整驱动的验证（L3 整驱动级别）。

**违反核心原则2的典型后果**：
```
error[E0583]: file not found for module `config`
  --> kernel/comps/virtio/src/device/console/mod.rs:XX:Y
   |
XX | mod config;
   | ^^^^^^^^^^
   = help: to create the module `config`, create file ".../config.rs"
```
**原因**：删除了 config.rs 文件或 `mod config;` 声明，导致整个驱动架构断裂。

---

## 一、开始前的检查清单

1. 确认设备类型：查看 `transport.device_type()` 返回的 `VirtioDeviceType`
2. 确认队列数量和索引：不同设备有不同的队列布局
3. 确认需要的 DMA 缓冲区数量和大小
4. **先阅读标准实现**：在 `kernel/comps/virtio/src/device/` 下找到同类驱动

---

## 二、驱动初始化流程

必须按以下顺序执行：
1. 创建 VirtQueue（调用 VirtQueue::new）
2. 分配 DMA 缓冲区（DmaStream::alloc 或使用 buffer pool）
3. 构建设备结构体（用 Arc 包装）
4. 预填充接收队列（add_dma_buf + notify）
5. 注册中断回调（register_queue_callback）
6. 注册配置变更回调（register_cfg_callback）
7. 完成初始化（transport.finish_init）
8. 注册到子系统

**关键约束**：必须先创建队列、分配缓冲区，再注册回调，最后调用 finish_init。

**Socket 驱动特殊步骤**：
1. 在 mod.rs 中初始化全局缓冲池：`buffer::init()`
2. 创建 **3 个队列**：recv_queue (index=0), send_queue (index=1), event_queue (index=2)
3. 使用 `SlotVec<RxBuffer>` 管理 64 个接收缓冲区
4. 从配置空间读取 guest_cid（64位，分两次读取高低32位）
5. 注册设备到 VSOCK_DEVICE_TABLE（而非直接注册到子系统）

---

## 三、参考已有实现（最重要！）

**务必先阅读** `kernel/comps/virtio/src/device/` 下的已有实现：
- Console 驱动：`console/device.rs`（简单，适合入门）
- Input 驱动：`input/device.rs`（事件处理，缓冲池）
- Socket 驱动：`socket/device.rs`（最复杂，连接管理+流控制）
- Network 驱动：`network/device.rs`
- Block 驱动：`block/device.rs`

### 如何参考
1. 找到功能最相似的驱动
2. 理解其 buffer 管理方式（用什么结构、如何分配）
3. 理解其队列操作流程（add → notify → wait → pop）
4. **复制其设计模式**，而非自己发明

---

## 四、DMA 缓冲区约束

### 分配规则
- ✅ 正确：按页分配（1页 = 4096字节），使用 Slice 灵活管理
- ❌ 错误：计算 buffer 数量 × size，结果超过 PAGE_SIZE 会触发 panic

**PAGE_SIZE 是硬限制**：任何 DMA 分配都不能超过 4096 字节！

### 缓冲区数量设计（关键！）
不同设备有不同的缓冲区策略，**必须参考标准实现**：

**Console 设备**：使用少量缓冲区
- 1 个 send_buffer（发送数据）
- 1 个 receive_buffer（接收数据）
- ✅ 实战验证：Console 使用 2 个队列，每个队列大小为 2

**Input 设备**：使用缓冲池（标准：64 个 event buffers）
- ✅ 正确：预分配足够大的 DMA buffer 来存储多个事件
- ✅ 实战验证：Input 使用单个 DMA buffer（按页分配），但计算为 `(NUM_EVENT_BUFFERS * EVENT_SIZE + 4095) / 4096` 页
- ❌ 错误：只用 1 个 buffer，高频率输入会丢失事件
- **原因**：VirtIO virtqueue 需要多个 in-flight buffers 才能处理并发输入

**Socket 设备**：使用 SlotVec 管理多个 RX buffers（最复杂）
- ✅ 正确：使用 `SlotVec<RxBuffer>` 管理多个接收缓冲区
- ✅ 正确：必须初始化缓冲池（`buffer::init()`），使用 `RX_BUFFER_POOL` 和 `TX_BUFFER_POOL`
- ✅ 正确：使用 `RxBuffer` 和 `TxBuffer`（来自 `aster_network`，不是 DmaStream！）
- ✅ 实战验证：Socket 使用 3 个队列（recv/send/event），每个队列大小为 64
- ❌ 错误：像Console那样只用1-2个buffer
- **原因**：Socket需要处理多个并发连接和数据流

**关键洞察**：
- Console：每个缓冲区单独管理（send_buffer, receive_buffer）
- Input：所有事件缓冲区在单个 DMA 大块中，通过 Slice 分片管理
- Socket：使用 SlotVec 动态管理多个 RxBuffer，通过缓冲池分配

### 发送数据流程（Driver → Device）
1. 用 `writer()` 写入数据到 DMA buffer
2. **必须调用 `sync_to_device(range)` 同步**
3. 用 `Slice::new(&buffer, range)` 创建切片（range 是实际写入长度）
4. 将 slice 添加到队列

### 接收数据流程（Device → Driver）
1. **必须调用 `sync_from_device(range)` 同步**
2. 用 `reader()` 读取数据
3. 用 `limit()` 设置实际数据长度

**最易错点**：
- Slice 创建时使用实际写入长度，而非固定缓冲区大小
- 忘记同步缓冲区会导致数据不一致

---

## 五、VirtQueue 操作流程

### 添加缓冲区
- 设备读取的数据放在 inputs 数组
- 设备写入的数据放在 outputs 数组
- 调用 add_dma_buf 后检查 should_notify，决定是否通知设备

### 取回缓冲区
- 用 can_pop 检查是否有已处理缓冲区
- 用 pop_used 取回缓冲区和长度
- 处理完后要把缓冲区重新放回队列

### 完整发送流程（大数据必须分块循环处理！）

**⚠️ 关键：大数据（> buffer size）必须分块发送，循环等待每次完成**

**标准模式**（Console send 为例）：
```rust
let mut reader = VmReader::from(data);
while reader.remain() > 0 {  // ✅ 循环处理所有数据
    // 1. 写入一块数据到 buffer
    let mut writer = self.send_buffer.writer().unwrap();
    let len = writer.write(&mut reader);
    
    // 2. 同步到设备
    self.send_buffer.sync_to_device(0..len).unwrap();
    
    // 3. 创建 Slice 并添加到队列
    let slice = Slice::new(&self.send_buffer, 0..len);
    transmit_queue.add_dma_buf(&[&slice], &[]).unwrap();
    
    // 4. 通知设备
    if transmit_queue.should_notify() {
        transmit_queue.notify();
    }
    
    // 5. 等待当前块传输完成（必须在循环内！）
    while !transmit_queue.can_pop() {
        spin_loop();
    }
    
    // 6. 取回传输完成的缓冲区
    transmit_queue.pop_used().unwrap();
}
```

**易错点**：
- ❌ 只发送前 N 字节就返回（如 `write_len.min(BUFFER_SIZE)`）
- ❌ 不等待传输完成就返回（跳过 can_pop/pop_used）
- ✅ **必须**：while 循环 + 每块等待完成 + pop_used 确认

---

## 六、中断处理

- 回调中获取队列锁时要使用 `disable_irq().lock()`，避免嵌套中断
- 处理完事件后要把缓冲区重新放回队列

---

## 七、特性协商

- 使用 `from_bits_truncate` 处理特性位，而非 `from_bits`
- 根据设备类型移除不支持的特性

---

## 七（补充）、VirtIO Input 事件处理

### 事件类型
- **EV_KEY**: 键盘/按键事件（状态：0=释放，1=按下，2=重复）
- **EV_REL**: 相对运动事件（鼠标移动）
- **EV_ABS**: 绝对坐标事件（触摸屏）
- **SYN_REPORT**: 同步事件（标记一组事件的结束）

### 事件处理流程
1. 从 EventBuf 读取 `VirtioInputEvent`
2. 根据 event.type 分发到不同处理器
3. **SYN_REPORT 必须提交到子系统**，而不是忽略或提交空数组
4. 使用 `InputEvent::from_sync_event(SynEvent::Report)` 创建同步事件

**事件批处理语义（重要！）**：
- ✅ 正确：收集一批事件（存入 Vec），收到 SYN_REPORT 时统一提交
- ❌ 错误：每个事件立即单独提交，破坏事件流语义
- **标准模式**：`registered_device.submit_events(&[event1, event2, syn_event])`

**易错点**：SYN_REPORT 不是边界标记，而是必须提交的实际事件，用于通知输入子系统"一组事件已完成"。

---

## 七（补充2）、VirtIO Socket 连接与流控制

### 队列结构（3个队列！）
- **recv_queue (index=0)**：接收数据包
- **send_queue (index=1)**：发送数据包
- **event_queue (index=2)**：处理事件通知（如live migration）
- 每个队列大小：64

### 配置空间读取（guest_cid）
**必须分两次读取64位guest_cid**：
1. 用 `field_ptr!` 读取 `guest_cid_low`（低32位）
2. 用 `field_ptr!` 读取 `guest_cid_high`（高32位）
3. 合并：`low as u64 | (high as u64) << 32`

**原因**：VirtIO spec要求>32位的读取不保证原子性。

### 缓冲区管理（SlotVec + Buffer Pool）
**关键**：
- 必须从缓冲池分配（RX_BUFFER_POOL / TX_BUFFER_POOL）
- 使用 `SlotVec` 动态管理，支持按token取回
- 初始时必须预填充所有64个buffer（循环调用add_dma_buf）
- Socket 使用 `RxBuffer` 和 `TxBuffer`（来自 `aster_network`），不是 `DmaStream`！

### 连接管理（状态机）
Socket支持的连接操作：
- **request**：发起连接请求（op=Request）
- **response**：响应连接请求（op=Response）
- **shutdown**：优雅关闭连接（op=Shutdown）
- **reset**：强制重置连接（op=Rst）
- **credit_request**：请求对端发送信用信息（op=CreditRequest）
- **credit_update**：通知对端自己的信用状态（op=CreditUpdate）

### 流控制与信用机制（最关键！）
**问题**：发送方不能发送超过接收方缓冲区的数据量。

**解决方案**：信用机制（buf_alloc, fwd_cnt, peer_free）
- `peer_free = buf_alloc - (fwd_cnt - last_fwd_cnt)`
- **发送前必须检查** `peer_free >= buffer_len`
- 如果不足：发送 `CreditRequest` → 等待对端回复 `CreditUpdate` → 重试
- 标记 `has_pending_credit_request` 避免重复请求

### 数据发送流程（TxBuffer）
**步骤**：
1. 从 TX_BUFFER_POOL 分配 TxBuffer
2. 用 TxBuffer::new 封装 header + payload（VmReader）
3. add_dma_buf 到 send_queue
4. notify 通知设备
5. **必须等待完成**：`while !can_pop { spin_loop(); }` + `pop_used()`

**关键**：Socket 使用 `TxBuffer` 和 `RxBuffer`（来自 `aster_network`），不是 `DmaStream`！

### 数据接收流程
**步骤**：
1. `recv_queue.pop_used()` 获取 (token, len)
2. 从 `rx_buffers` 按 token 移除对应的 RxBuffer
3. 读取 header 和 payload
4. **必须补充新的 rx_buffer** 到队列（从 RX_BUFFER_POOL 分配）
5. notify 通知设备

### 设备注册（全局表）
**Socket 注册到 VSOCK_DEVICE_TABLE**（模块内部全局表）
**API**：`register_device()`, `get_device()`, `all_devices()`, `register_recv_callback()`

**关键差异**：
- Console：直接注册到 `aster_console` 子系统
- Socket：注册到 `VSOCK_DEVICE_TABLE`（模块内部全局表）

### 最易错点
1. ❌ 混用 `DmaStream` 和 `RxBuffer/TxBuffer`（Socket必须用后者）
2. ❌ 忘记初始化缓冲池（`buffer::init()`）
3. ❌ 不预填充64个rx buffer（导致接收队列空）
4. ❌ 发送前不检查 `peer_free`（导致缓冲区溢出）
5. ❌ 读取guest_cid时用 `read_once::<u64>()`（必须分两次读32位）
6. ❌ 忘记等待发送完成（`can_pop` + `pop_used`）
7. ❌ 设备注册位置错误（应注册到 `VSOCK_DEVICE_TABLE`）

---

## 八、常见反模式（禁止！）

### 🔴 反模式1：过度创新架构
**错误做法**：自己发明新的 buffer 管理机制（如 ReceiveBuffers + SafePtr）

**正确做法**：使用标准的 `DmaStream` + `Slice` 模式，参考 console/device.rs

### 🔴 反模式2：每次发送都分配新 buffer
**错误做法**：在 send() 中动态分配 DMA buffer

**正确做法**：复用 init 时预分配的 buffer（存储在设备结构体中）

### 🔴 反模式3：错误的 API 使用
**错误做法**：
- 猜测 API 名称（如 `read_once()`）
- 调用 `sync_from_device()` 不传参数
- 调用 `add_dma_buf(stream, &[])` 传递错误类型

**正确做法**：
- `sync_*` 方法**必须**传递 `Range<usize>` 参数
- `add_dma_buf` 第一个参数是 `&[&Slice]`，不是 `&Arc<DmaStream>`
- 先用 `Slice::new(&buffer, range)` 创建切片，再传递

### 🔴 反模式4：修改 quiz 范围外的代码
**错误做法**：删除未标记 MASK 的函数或修改其他区域

**正确做法**：只修改 MASK 标记的区域

### 🔴 反模式5：大数据不循环处理
**错误做法**：
- 只发送前 N 字节就返回（如 `write_len.min(BUFFER_SIZE)`）
- 不等待传输完成就返回
- 不使用 while 循环处理剩余数据

**正确做法**：
```rust
let mut reader = VmReader::from(data);
while reader.remain() > 0 {  // ✅ 循环直到所有数据发完
    // 写入一块
    let len = writer.write(&mut reader);
    // 发送这块
    // ...
    // 等待完成
    while !queue.can_pop() { spin_loop(); }
    queue.pop_used().unwrap();
}
```

**实战教训**：VirtIO buffer 有大小限制，大数据必须分块发送并同步等待每块完成。

### 🔴 反模式6：过度重构导致编译失败
**错误做法**：
- 完全重写文件（如从 573 行重写到 499 行）
- 引入新的 trait/struct 命名冲突（如重复定义 `InputDevice`）
- 使用错误的泛型参数（如 `Rcu<Box<Vec<Arc<DmaStream>>>>`）
- 在未编译验证的情况下提交代码

**正确做法**：
1. **基于标准答案增量修改**：永远不要从零重写
2. **每次修改后立即 `cargo check`**：确保编译通过再继续
3. **遇到编译错误立即停止**：不要提交未完成代码
4. **使用明确的类型标注**：避免类型推断失败
5. **复用已有命名空间**：不要引入与现有 trait/struct 冲突的名称

**编译失败诊断**：
- `trait bound not satisfied` → 检查泛型参数是否符合 trait 约束
- `found struct but expected trait` → 命名冲突，使用完整路径或改名
- `type annotations needed` → 添加明确的类型标注

### 🔴 反模式7：删除或破坏配置空间模块
**错误做法**：
- 删除 `config.rs` 文件或大幅缩减其内容（如从 73 行减到 1 行）
- 删除 `mod config;` 声明，导致模块依赖链断裂
- 自定义 `Config` 结构体时不实现必需的 trait

**正确做法**：
1. **保留完整的 config.rs 模块**，不要删除任何未标记 MASK 的代码
2. **保留 mod.rs 中的模块声明**：`mod config;`
3. 如果需要定义配置结构体，**必须实现所有必需的 trait**：
```rust
#[derive(Debug, Pod, Clone, Copy)]  // ✅ Pod trait 是必需的！
#[repr(C)]
pub struct VirtioConsoleConfig {
    pub cols: u16,
    pub rows: u16,
    pub max_nr_ports: u32,
    pub emerg_wr: u32,
}
```

**关键约束**：
- VirtIO 配置结构体**必须**实现 `Pod` trait（来自 `ostd_pod`）
- `Pod` trait 要求：`KnownLayout` + `Immutable` + `Copy` + `FromZeros` 等
- 使用 `#[derive(Pod, Clone, Copy)]` 自动派生，不要手动实现

**为什么重要**：
- `ConfigManager<T>` 泛型约束要求 `T: Pod`
- 配置空间管理是 VirtIO 驱动的核心部分，删除会导致编译失败
- 模块依赖链断裂会引发连锁错误（22+ 编译错误）

### 其他反模式（续）
1. 队列索引混淆：确认每个队列的索引号
2. DMA 同步遗漏：必须在数据传输前后同步
3. 中断嵌套：在回调中直接获取锁而不禁用中断
4. 缓冲区泄漏：从队列取出后忘记放回

---

## 九、编码步骤

### Step 0: 检查 MASK 范围（关键！）
**在动手前必须确认**：
1. 哪些文件被标记为需要修改（查看 quiz meta.yaml）
2. 每个文件中哪些行是 MASK 区域（`// MASK_START` / `// MASK_END`）
3. **绝不要删除或修改 MASK 外的代码**，即使看起来"没用"

**常见陷阱**：
- 删除 `config.rs` 或缩减其内容 → 编译失败
- 删除 `mod config;` → 模块依赖链断裂
- 修改结构体定义（非 MASK 区域） → 类型不匹配

### Step 1: 先看同类驱动（5分钟）
查看 `kernel/comps/virtio/src/device/` 下的标准实现

### Step 2: 理解核心设计
- 它用什么结构管理 buffer？
- 队列操作流程是什么？
- 中断处理如何实现？

### Step 3: 严格复制其模式
**不要"改进"或"优化"标准答案，先跑通再说！**

### Step 4: 确保编译通过
- 检查类型签名是否匹配
- 检查方法名称是否正确（探索时注意看方法定义）

### Step 5: 验证功能
- 运行测试脚本
- 检查日志输出

---

## 十、API 快速参考

### DMA Stream
- `DmaStream::alloc(pages, direction)` - 分配 DMA 缓冲区（按页）
- `buffer.writer()` / `buffer.reader()` - 获取读写器
- `buffer.sync_to_device(range)` / `sync_from_device(range)` - 同步数据

### Slice
- `Slice::new(buffer, range)` - 创建 DMA buffer 的切片

### VirtQueue
- `VirtQueue::new(queue_idx, size, transport)` - 创建队列
- `queue.add_dma_buf(inputs, outputs)` - 添加缓冲区
- `queue.should_notify()` / `queue.notify()` - 通知设备
- `queue.can_pop()` / `queue.pop_used()` - 取回缓冲区

### EventBuf (Input)
- `event_buf.read()` - 读取输入事件（✅ 正确）
- ❌ `event_buf.read_once()` - 不存在的方法

---

## 十一、调试技巧

1. **编译错误**：检查方法名是否正确，类型是否匹配
2. **运行时 panic**：检查 buffer 大小是否超过 PAGE_SIZE
3. **测试超时**：检查队列索引是否正确，notify 是否被调用
4. **数据不一致**：检查 sync_to_device/sync_from_device 是否遗漏

---

## 十二、实战验证的编码流程（✅ 已验证有效）

以下流程已通过 VirtIO Console 和 Input 两个完整驱动的验证：

### Step 1: 理解设备需求（5分钟）
1. 确认设备类型和队列布局
2. 确认缓冲区策略（单缓冲区 vs 缓冲池）
3. 确认事件/数据流方向

### Step 2: 阅读参考实现（10分钟）
**关键**：阅读 `kernel/comps/virtio/src/device/` 下的标准实现
- Console: 简单的双队列模式，适合学习基础
- Input: 复杂的事件处理，适合学习批处理和缓冲池

### Step 3: 实现核心结构（15分钟）
1. 定义设备结构体（参考标准实现的字段）
2. 实现 negotiate_features（使用 from_bits_truncate）
3. **立即 cargo check 确保编译通过**

### Step 4: 实现初始化流程（20分钟）
**必须严格按顺序**：
1. 创建 VirtQueue（确认队列索引正确）
2. 分配 DMA 缓冲区（按页计算）
3. 构建设备结构体
4. 预填充接收队列（如果需要）
5. 注册回调
6. 调用 finish_init
7. **每步后 cargo check**

### Step 5: 实现数据流（20分钟）
**发送数据**（Console 为例）：
1. 用 writer() 写入数据
2. sync_to_device 同步
3. 创建 Slice（使用实际长度）
4. add_dma_buf + should_notify + notify
5. 等待 can_pop + pop_used

**接收/处理数据**（Input 为例）：
1. 在中断回调中处理
2. sync_from_device 同步
3. 读取数据
4. 处理后重新提交缓冲区

### Step 6: 验证和调试（10分钟）
1. 运行测试脚本
2. 检查 QEMU 输出日志
3. 如果失败，逐步检查每个环节

**成功关键**：
- 严格遵守初始化顺序
- 不要跳过同步步骤
- 使用正确的 API（探索时注意方法签名）
- 每步验证编译

---

**记住**：Asterinas 驱动开发不是创造新架构，而是理解并应用已有模式。先跑通，再优化！
