---
name: asterinas-driver-dev
description: 给 Student 使用的 Asterinas 总驱动技能，用来总结跨题可复用的驱动写法、判断顺序、约束和反模式。何时使用：当你在实现或修改 Asterinas 驱动代码，需要知道先检查什么、什么时候这样写、为什么这样写、哪些地方最容易出错时。触发短语：Asterinas 驱动开发, virtio 驱动, 驱动写法, 队列与中断, DMA 约束.
roles: [student]
---

# Asterinas VirtIO 驱动开发指南

## ⚠️ 核心原则（必读）

1. **严格遵循标准答案结构**：不要"过度创新"或发明新的架构
2. **只修改 MASK 标记区域**：绝不删除或修改 quiz 范围外的代码
3. **先参考已有实现**：在动手前，必须先阅读并理解同类驱动的标准写法
4. **PAGE_SIZE 是硬限制**：所有 DMA buffer 分配不能超过 4096 字节

**✅ 实战验证**：这套指南已通过 VirtIO Console 和 Input 两个完整驱动的验证（L3 整驱动级别，100% 通过率）。

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
2. 分配 DMA 缓冲区（DmaStream::alloc）
3. 构建设备结构体（用 Arc 包装）
4. 预填充接收队列（add_dma_buf + notify）
5. 注册中断回调（register_queue_callback）
6. 注册配置变更回调（register_cfg_callback）
7. 完成初始化（transport.finish_init）
8. 注册到子系统

**关键约束**：必须先创建队列、分配缓冲区，再注册回调，最后调用 finish_init。

---

## 三、参考已有实现（最重要！）

**务必先阅读** `kernel/comps/virtio/src/device/` 下的已有实现：
- Console 驱动：`console/device.rs`
- Input 驱动：`input/device.rs`
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

**关键洞察**：
- Console：每个缓冲区单独管理（send_buffer, receive_buffer）
- Input：所有事件缓冲区在单个 DMA 大块中，通过 Slice 分片管理

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

### 完整发送流程
1. 写入数据到预分配的 DMA buffer
2. 调用 sync_to_device 同步
3. 创建 Slice 并添加到队列
4. 检查 should_notify 后通知设备
5. 等待 can_pop 返回 true
6. 调用 pop_used 取回完成确认

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

## 八、常见反模式（禁止！）

### 🔴 反模式1：过度创新架构
**错误做法**：自己发明新的 buffer 管理机制（如 ReceiveBuffers + SafePtr）

**正确做法**：使用标准的 `DmaStream` + `Slice` 模式，参考 console/device.rs

### 🔴 反模式2：每次发送都分配新 buffer
**错误做法**：在 send() 中动态分配 DMA buffer

**正确做法**：复用 init 时预分配的 buffer（存储在设备结构体中）

### 🔴 反模式3：错误的 API 使用
**错误做法**：猜测 API 名称（如 `read_once()`）

**正确做法**：探索代码时注意方法签名，确认正确的方法名（如 `read()`）

### 🔴 反模式4：修改 quiz 范围外的代码
**错误做法**：删除未标记 MASK 的函数或修改其他区域

**正确做法**：只修改 MASK 标记的区域

### 🔴 反模式5：过度重构导致编译失败
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

### 其他反模式
1. 队列索引混淆：确认每个队列的索引号
2. DMA 同步遗漏：必须在数据传输前后同步
3. 中断嵌套：在回调中直接获取锁而不禁用中断
4. 缓冲区泄漏：从队列取出后忘记放回

---

## 九、编码步骤

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
