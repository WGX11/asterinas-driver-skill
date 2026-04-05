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

## ⚠️ 提交前必做检查（零容忍）

**以下错误会导致直接编译失败，必须在提交前自行验证：**

### 1. 编译验证（必须通过！）
```bash
cd /root/test_repo/asterinas
make build  # 或 cargo build
```
**必须**：0 errors，0 warnings

### 2. 常见低级错误（100%可避免）
- ❌ **括号不匹配**：`Arc::new(DmaStream::alloc(1, false).unwrap();`  // 缺少闭合 `)`
- ❌ **代码重复粘贴**：同一个代码块出现两次
- ❌ **花括号不匹配**：`{` 和 `}` 数量对不上
- ❌ **缺少 trait 导入**：调用 trait 方法但未 `use` 该 trait

**检查方法**：
1. IDE / 编辑器会标红语法错误
2. `cargo check` 会立即报错
3. **不要等 Judge 告诉你这些基础问题**

### 3. Trait 可见性规则
**错误示例**：
```rust
let writer = buffer.writer().unwrap();  // ❌ 编译错误
```
**编译器提示**：
```
error[E0599]: no method named `writer` found for struct `DmaStream`
  --> device.rs:50:25
   |
50 |         let writer = buffer.writer().unwrap();
   |                         ^^^^^^ method not found in `DmaStream`
   |
   = help: items from traits can only be used if the trait is in scope
   = note: the following trait is implemented but not in scope:
           `HasVmReaderWriter`
```

**正确做法**：
```rust
use ostd::mm::io_util::HasVmReaderWriter;  // ✅ 添加 trait 导入

let writer = buffer.writer().unwrap();     // ✅ 现在可以调用
```

**关键**：编译器已经告诉你缺什么了，**立即按提示修复**，不要跳过。

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

**标准模式**（循环处理所有数据）：
1. 写入一块数据到 buffer（`writer.write(&mut reader)`）
2. 同步到设备（`sync_to_device(0..len)`）
3. 创建 Slice 并添加到队列（`add_dma_buf(&[&slice], &[])`）
4. 通知设备（`should_notify()` + `notify()`）
5. **等待当前块传输完成**（`while !can_pop() { spin_loop(); }`）
6. 取回传输完成的缓冲区（`pop_used()`）
7. **循环回到步骤1**，直到所有数据发完

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

## 七（补充）、设备特定规则简表

**Input驱动**：批处理事件+SYN_REPORT必须提交，参考input/device.rs
**Socket驱动**：SlotVec+缓冲池+流控制+guest_cid分两次读取，参考socket/device.rs完整流程

---

## 七（补充2）、VirtIO Socket 驱动（连接管理+流控制）

### ⚠️ 核心架构原则

**1. 配置空间读取必须使用 `field_ptr!` 宏**
- ❌ 错误：`ConfigManager::read_config()` - 该方法不存在
- ✅ 正确：使用 `field_ptr!` 宏访问配置字段（guest_cid 分两次读取高低32位）

**2. ConnectionInfo 必须包含流控字段**
- ❌ 错误：只定义基本字段（src_cid, dst_cid, src_port, dst_port）
- ✅ 正确：包含 `buf_alloc`、`fwd_cnt`、`last_fwd_cnt`、`tx_cnt` 等流控字段
- **约束**：`buf_alloc - fwd_cnt` = 对端可用缓冲区大小

**3. Header 构造必须使用辅助方法**
- ❌ 错误：手动填充所有字段（容易遗漏或填错）
- ✅ 正确：使用 `ConnectionInfo::new_header()` 自动填充通用字段

**4. Rust 所有权陷阱**
- ❌ 错误：move 后又 borrow（编译错误）
- ✅ 正确：先 borrow，再 move（最后一次使用）

### 流控制实现要点
- 发送前：检查 `buf_alloc - fwd_cnt` >= 发送长度
- 发送后：更新 `tx_cnt`
- 接收 CreditUpdate：更新 `fwd_cnt`

---

## 八、VirtIO Block 驱动（异步队列模式）

### ⚠️ 核心架构原则（必读！）

**1. 绝对禁止删除 mod.rs 中的类型定义！**
- ❌ 错误：将 mod.rs 从 206 行缩减到 6 行
- ✅ 正确：保留所有未标记 MASK 的代码（BioRequestSingleQueue、VirtioBlockConfig等）
- **失败案例**：删除类型定义 → 50+ 编译错误

**2. 必须使用 BioRequest 架构**
- ❌ 错误：直接实现 `BlockDeviceOps for SubmittedBio`
- ✅ 正确：使用 `BioRequestSingleQueue` + `BioRequest` 管理队列

**3. 异步队列模式**
- ❌ 错误：busy-wait（`while !can_pop() { spin_loop() }`）
- ✅ 正确：提交请求→立即返回→中断处理完成

**4. VirtIO 配置结构体必须使用 Pod trait**
- ❌ 错误：手动定义结构体不添加标记
- ✅ 正确：使用 `#[padding_struct]` + `#[derive(Pod, Clone, Copy)]`
- **失败案例**：`VirtioBlockConfig` 缺少标记 → 无法满足 Pod trait → 85+ 编译错误

### 核心架构模式
**VirtIO Block 驱动使用异步队列模式**，与 Console/Input 的同步模式完全不同：

**关键差异**：
- ❌ 错误：使用 busy-wait 等待完成（`while !can_pop() { spin_loop() }`）
- ✅ 正确：提交请求后立即返回，通过中断处理完成回调

**标准架构**：
1. **提交请求**（不等待）：构造请求 → 提交到 VirtQueue → notify 设备 → 立即返回
2. **中断处理完成**：从 VirtQueue 弹出请求 → 处理响应 → 通知上层 BioRequest 完成

### Block API 正确使用

**核心类型**：
- `BioRequest`：块 I/O 请求（包含多个 bio segments）
- `BioRequestSingleQueue`：软件暂存队列（管理 pending bio requests）
- `BioType`：请求类型（Read/Write/Flush等）

**⚠️ 关键：访问 BioRequest 的数据（最易错！）**
- ❌ 错误：`bio.type`、`bio.direction()`、`seg.read()` - 不存在的方法
- ✅ 正确：迭代 `bio_request.bios()` → `bio.segments()` → 使用 `seg.reader()` 或 `seg.writer()`
- ✅ 正确：`bio_request.type_()` - 返回 BioType（注意下划线）
- ✅ 正确：`bio_request.sid_range().start.to_raw()` - 获取扇区号

**易错点**：
1. ❌ panic 处理错误 → ✅ 使用 `BioStatus::IoError` 优雅处理
2. ❌ 忘记调用 `bio_request.end_with()` → ✅ 完成请求

### Pod Trait 和配置结构体（关键！）

**VirtIO 配置结构体必须实现 Pod trait**：
- ❌ 错误：缺少 `#[padding_struct]` 或 `#[derive(Pod)]` → 85+ 编译错误
- ✅ 正确：使用 `#[repr(C)]` + `#[padding_struct]` + `#[derive(Debug, Copy, Clone, Pod)]`
- ✅ 正确：初始化使用 `VirtioBlockConfig::new_zeroed()`

**约束**：ConfigManager 要求泛型参数 `T: Pod`

### 参考 mod.rs 的完整结构（Block 驱动）

**必须保留的类型定义**（在 mod.rs 中）：
- `BlockFeatures` bitflags
- `ReqType` 枚举（In/Out/Flush/GetId/Discard/WriteZeroes）
- `RespStatus` 枚举（Ok/IoErr/Unsupported）
- `VirtioBlockConfig` 结构体（带 `#[padding_struct]` 和 `#[derive(Pod)]`）
- `VirtioBlockGeometry` 和 `VirtioBlockTopology` 辅助结构体

### Descriptor Chain 构造规则

**读操作（ReqType::In）**：
- inputs: `[request_header_slice]`（设备读取请求头）
- outputs: `[data_slice, response_slice]`（设备写入数据和响应）

**写操作（ReqType::Out）**：
- inputs: `[request_header_slice, data_slice]`（设备读取请求头和数据）
- outputs: `[response_slice]`（设备写入响应）

**关键约束**：
- **input** = 设备从内存读取（Driver → Device）
- **output** = 设备向内存写入（Device → Driver）

### 请求-响应匹配（关键！）

**使用 ID 分配器追踪请求**：
- 提交时：`id = id_allocator.alloc()` → `pending_requests.insert(id, bio_request)`
- 完成时：`bio_request = pending_requests.remove(&id)` → 处理响应 → `id_allocator.dealloc(id)`

**易错点**：忘记回收 ID → ID 耗尽 → 驱动卡死

### DMA 缓冲区管理

**预分配缓冲池**（在 init 时，不是每次请求时）：
- 使用 `DmaStream::alloc` 分配大块缓冲区
- 使用 `Slice::new` 分片管理多个请求槽位
- 槽位计算：`id * SIZE..(id + 1) * SIZE`

**DMA 同步规则**：
- **提交前**：`sync_to_device()` 同步请求头和数据（写操作）
- **完成后**：`sync_from_device()` 同步响应和数据（读操作）

### 中断处理流程

**标准步骤**：
1. 禁用中断避免嵌套：`queue.disable_irq().lock()`
2. 循环弹出完成的请求：`while can_pop() { pop_used() }`
3. 根据 token 找到请求：从 `pending_requests` 移除
4. 检查响应状态：读取 `VirtioBlockResp.status`
5. DMA 同步数据（读操作）：`sync_from_device()`
6. 完成请求：`bio_request.end_with(BioStatus::Success/IoError)`
7. 回收资源：`id_allocator.dealloc(id)`

**关键约束**：必须在中断上下文中禁用中断锁，避免嵌套中断

---

## 九、常见反模式（禁止！）

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
1. 使用 while 循环：`while reader.remain() > 0`
2. 每次发送一块数据
3. **必须**等待当前块完成：`while !can_pop() { spin_loop(); }` + `pop_used()`
4. 循环直到所有数据发完

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
3. 如果需要定义配置结构体，**必须实现所有必需的 trait**：使用 `#[derive(Pod, Clone, Copy)]` 自动派生

**关键约束**：
- VirtIO 配置结构体**必须**实现 `Pod` trait（来自 `ostd_pod`）
- `Pod` trait 要求：`KnownLayout` + `Immutable` + `Copy` + `FromZeros` 等
- 不要手动实现，使用 `#[derive(Pod, Clone, Copy)]` 自动派生

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

## 九、编码步骤（精简版）

### Step 0: 检查 MASK 范围（关键！）
1. 确认哪些文件被标记为需要修改（查看 quiz meta.yaml）
2. 确认每个文件中哪些行是 MASK 区域
3. **绝不要删除或修改 MASK 外的代码**

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
- 检查方法名称是否正确
- **每次修改后立即 cargo check**

### Step 5: 验证功能
运行测试脚本，检查日志输出

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

## 十二、DMA Zero-Copy 与 Descriptor Chain（核心！）

### ⚠️ 核心原则

**零拷贝原则**：让设备直接读写最终的数据缓冲区，避免中间缓冲区中转。

### 🔴 反模式：中间缓冲区（virtio_block 失败原因）

**错误**：分配中间 DMA buffer → 设备写入 → 再拷贝到 bio segment → panic: AccessDenied

**原因**：bio segment 已映射给设备，无法同时通过 writer() 写入

### ✅ 正确做法

**直接使用 `seg.inner_dma_slice()`**：
- 获取 bio segment 底层 DMA buffer
- 传递给 VirtQueue，让设备直接读写
- 设备完成后，数据已在 segment 中，无需二次拷贝

### Descriptor Chain 规则

- **input**：设备从该内存**读取**（Driver → Device）
- **output**：设备向该内存**写入**（Device → Driver）

**读操作**：`add_dma_buf(&[req_slice], &[data_slice, resp_slice])`  
**写操作**：`add_dma_buf(&[req_slice, data_slice], &[resp_slice])`

### Bio Segment API

- `seg.reader()/writer()` - Driver 读写数据（⚠️ 检查 Vmar 权限和设备占用）
- `seg.inner_dma_slice()` - 获取 DMA 地址传给 VirtQueue（零拷贝关键）

**判断顺序**：设备需要读写 segment → 使用 inner_dma_slice() → Driver 需要访问 → 检查 reader()/writer() 可用性

### DMA 同步

- **设备写入**：完成后 `sync_from_device`
- **设备读取**：提交前 `sync_to_device`

---

**记住**：Asterinas 驱动开发不是创造新架构，而是理解并应用已有模式。先跑通，再优化！
