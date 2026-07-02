---
title: "Alveo U50开发系列：Vitis RTL Kernel 与 XRT 的驱动契约"
date: 2026-07-02 20:00:00 +0800
description: 从顶层端口、AXI4-Lite 控制寄存器、AXI4 主端口与 kernel.xml 四个方面，系统整理 Vitis RTL Kernel 需要满足的 XRT 驱动契约。
categories:
  - FPGA
home_category: engineering-practice
series: alveo-development
tags:
  - FPGA
  - Alveo
  - XRT
  - Vitis
  - RTL
  - AXI
article_kicker: FPGA NOTE
cover_image: /assets/images/alveo_u50/image-20260701223955214.png
word_count: 约 10k 字
read_time: 约 35 分钟
---

![Vitis RTL kernel and XRT contract cover](/assets/images/alveo_u50/image-20260701223955214.png)



> **文档定位**：与具体项目脱钩的通用参考。任何一份 RTL 想被 XRT 驱动、在 Alveo 上作为 kernel 跑起来，都必须满足这里列出的约束。
>
> **适用范围**：Vitis 2020.2 – 2023.2（新版 Vitis Unified IDE 有细节增补但契约不变）、XRT 2.6+、Alveo U50/U55C/U200/U250/U280/V70/V80。
>
> **依据**：
> - AMD/Xilinx UG1393《Vitis Unified Software Platform Documentation: Application Acceleration Development》
> - AMD/Xilinx UG1023《UltraScale Architecture-Based FPGAs Memory IP》
> - XRT 源码 `src/runtime_src/core/common/api/kernel_int.h`（`xrt::kernel` 与硬件的约定）
> - Vitis HLS 自动生成的 `*_control_s_axi.v` 参考实现

---

## 目录

- [1. 大局观：XRT 眼中的 RTL kernel](#1-大局观xrt-眼中的-rtl-kernel)
- [2. 顶层端口契约](#2-顶层端口契约)
- [3. AXI4-Lite 控制端口 `s_axi_control`](#3-axi4-lite-控制端口-s_axi_control)
- [4. 控制寄存器（0x00–0x0F）—— 与 XRT 握手的核心](#4-控制寄存器0x000x0f-与-xrt-握手的核心)
- [5. 用户参数寄存器（0x10 起）](#5-用户参数寄存器0x10-起)
- [6. AXI4 主端口 `m_axi_<port>`](#6-axi4-主端口-m_axi_port)
- [7. 中断输出 `interrupt`](#7-中断输出-interrupt)
- [8. 时钟与复位](#8-时钟与复位)
- [9. `kernel.xml` 元数据](#9-kernelxml-元数据)
- [10. 打包 `.xo` 的物理组织](#10-打包-xo-的物理组织)
- [11. XRT 主机侧调用如何映射回 RTL](#11-xrt-主机侧调用如何映射回-rtl)
- [12. 完整最小示例：CTRL 寄存器参考实现](#12-完整最小示例ctrl-寄存器参考实现)
- [13. 契约核对清单](#13-契约核对清单)
- [14. 常见陷阱](#14-常见陷阱)

---

## 1. 大局观：XRT 眼中的 RTL kernel

一个 Vitis RTL kernel，本质上是一份**符合固定接口约定的 Vivado IP**，被 `package_xo` 打进 `.xo` 容器，再被 `v++ -l` 链接到 Alveo 平台上生成 `.xclbin`。主机端 XRT 驱动只做两件事：

1. **通过 AXI4-Lite（`s_axi_control`）读写寄存器**——写入参数、启动、轮询状态；
2. **等待中断（`interrupt`）**——kernel 完成后通过 PCIe MSI 通知主机。

Kernel 自己怎么读 DDR/HBM，是通过 **`m_axi_<port>` 主端口** 发出 AXI4 事务，由平台侧的 SmartConnect 路由到对应的内存 bank。

```
+-------------------------+                    +-----------------+
| Host (XRT API)          |                    | Alveo Card      |
|                         |  PCIe (MMIO)       |                 |
|  xrt::kernel(...)    ---+------------------> | s_axi_control   |
|  xrt::run(bo0, bo1)     |                    |  ↕              |
|  run.wait()          <--+------ MSI --------+  Kernel RTL     |
|                         |                    |  ↕              |
|  xrt::bo (host mem)     |  PCIe DMA          | m_axi_gmem*     |
|                     <---+-------------------+  ↕              |
|                         |                    | HBM / DDR banks |
+-------------------------+                    +-----------------+
```

**契约的三块骨架**：
- **端口形状**（第 2–8 节）：Vitis packager 与 SmartConnect 依赖固定命名与信号集来识别端口
- **寄存器语义**（第 4–5 节）：XRT `xrt::run` 依据它决定「写 arg / 触发 / 等 done」的行为
- **元数据**（第 9 节）：`kernel.xml` 是 XRT 与 v++ 唯一读得懂的清单，把端口和 arg 关联起来

违反任何一块的后果按严重程度：
- 端口命名不对 → `package_xo` 报错，或 v++ 拒绝识别接口
- 寄存器语义不对 → 打包成功、`xclbin` 生成成功、`xrt::run().wait()` 永不返回
- `kernel.xml` 不对 → 主机传参错位，硬件读到垃圾指针

---

## 2. 顶层端口契约

### 2.1 端口命名（必须精确匹配）

| 端口用途 | 精确名字 | 位宽 | 方向 | 备注 |
|---|---|---|---|---|
| Kernel 时钟 | `ap_clk` | 1 | in | 所有 AXI 接口都用此时钟；不允许多时钟 kernel（多时钟走别的路径） |
| Kernel 复位（低有效） | `ap_rst_n` | 1 | in | 名字里是 `rst_n`，不是 `resetn` |
| 控制端口 | `s_axi_control` 前缀 + AXI4-Lite 信号集 | 见 §3 | | 必须叫 `s_axi_control` 或按 `kernel.xml` 声明的 slave 端口名 |
| 数据主端口 | `m_axi_<name>` 前缀 + AXI4 全接口信号集 | 见 §6 | | `<name>` 由 kernel.xml 声明，习惯用 `gmem`、`gmem0`、`in`、`out` 之类 |
| 中断 | `interrupt` | 1 | out | 单一名字，边沿触发 |

**没有第二个合法命名。** Vitis IP packager 逐字符匹配这些前缀来推断端口类别，`ap_reset_n` / `ap_clock` / `axi_control` 都不行。

### 2.2 端口方向与信号完备性

**AXI4-Lite** 与 **AXI4 full** 都有严格的信号集要求。缺一个信号（比如漏 `AWLEN`、漏 `RLAST`），packager 就把整个端口降级为「Generic」，v++ 拒绝把它接到平台内存。

**未使用的通道要在 `kernel.xml` 声明清楚**（`readOnly` / `writeOnly`），并且**顶层不要留悬空的 output**——要么删端口，要么显式 tie-off。悬空 output 在综合期只是 warning，但在 SmartConnect 上会导致 AXI 协议违例。

### 2.3 顶层 module 名

Module 名可以任意，但**必须与 `kernel.xml` 中 `<kernel name="...">` 完全一致**，也是主机端 `xrt::kernel(device, uuid, "kernel_name")` 用的字符串。

约定：module 名习惯用小写 `snake_case`，`vadd` / `matmul` / `acc_kernel` 都可以。

### 2.4 顶层不允许出现的东西

- **inout 端口**：三态在 FPGA 内部无意义，Vitis 直接报错
- **多 `ap_clk`**：单 kernel 单时钟；多时钟 kernel 走另一套流程（需要 `ap_clk_2`，且必须在 platform 上声明多时钟支持）
- **异步复位**：`ap_rst_n` 必须与 `ap_clk` 同步
- **外部信号**（GPIO、串口、以太网）：这些走 platform，不能挂在 kernel 上

---

## 3. AXI4-Lite 控制端口 `s_axi_control`

### 3.1 信号集（必须齐全）

**Write Address / Write Data / Write Response / Read Address / Read Data** 五条通道，共 21 个信号：

| 通道 | 信号 | 位宽 | 方向 |
|---|---|---|---|
| AW | `s_axi_control_AWADDR` | `C_ADDR_WIDTH` | in |
| | `s_axi_control_AWPROT` | 3 | in |
| | `s_axi_control_AWVALID` | 1 | in |
| | `s_axi_control_AWREADY` | 1 | out |
| W | `s_axi_control_WDATA` | 32 | in |
| | `s_axi_control_WSTRB` | 4 | in |
| | `s_axi_control_WVALID` | 1 | in |
| | `s_axi_control_WREADY` | 1 | out |
| B | `s_axi_control_BRESP` | 2 | out |
| | `s_axi_control_BVALID` | 1 | out |
| | `s_axi_control_BREADY` | 1 | in |
| AR | `s_axi_control_ARADDR` | `C_ADDR_WIDTH` | in |
| | `s_axi_control_ARPROT` | 3 | in |
| | `s_axi_control_ARVALID` | 1 | in |
| | `s_axi_control_ARREADY` | 1 | out |
| R | `s_axi_control_RDATA` | 32 | out |
| | `s_axi_control_RRESP` | 2 | out |
| | `s_axi_control_RVALID` | 1 | out |
| | `s_axi_control_RREADY` | 1 | in |

**关键点**：
- `WDATA` / `RDATA` **只能是 32 位**——Vitis 强制要求。arg 再宽（64 位指针）也拆成两个 32 位 register。
- `AWPROT` / `ARPROT` **必须存在**（尽管很多设计不使用）。缺它 packager 会把接口降级识别。
- `WSTRB` **必须实现**——XRT 有时用 byte-level strobe，不能忽略。

### 3.2 地址宽度 `C_ADDR_WIDTH`

由 kernel 的寄存器空间决定，向上取 2 的幂对齐到字节：

| 最大偏移 | 需要的 addr width |
|---|---|
| ≤ 0x0F | 4 位 |
| ≤ 0x3F | 6 位 |
| ≤ 0x7F | 7 位 |
| ≤ 0xFF | 8 位 |
| ≤ 0x1FF | 9 位 |
| ≤ 0xFFF | 12 位（Vitis 上限，实际很少见） |

**必须**顶层端口位宽和内部译码位宽**严格一致**（内部若使用参数化位宽，实例化时用 `#(.C_S_AXI_ADDR_WIDTH(N))` 显式覆盖），不然内部只用低几位、外部高位被 XRT 用到时会写到错误寄存器——这是经典的「握手看起来通了但结果错」的 bug 来源。

### 3.3 slave 行为约束

- 一次读/写事务不能超过 1 个 beat（AXI4-Lite 定义）
- `BRESP` / `RRESP` 通常返回 `2'b00` (OKAY)，硬件不该产生 SLVERR 除非有明确策略
- **写事务必须及时响应 `BVALID`**：不给 BVALID → XRT 会在下一个寄存器写时永远 stall
- **读事务的 `RVALID` 必须 latch 一个已就绪的 RDATA**：不能在 RVALID 高时 RDATA 是 X
- **允许 outstanding 深度 = 1**：XRT 会保守地一次一个事务，slave 不必做深流水

### 3.4 复位后的 slave 状态

复位后：
- 所有 valid/ready = 0
- 所有寄存器内容 = 0（尤其 CTRL = 0，否则可能在复位刚释放时错误启动）

---

## 4. 控制寄存器（0x00–0x0F）—— 与 XRT 握手的核心

**这是整个契约里最容易踩坑的地方。** 前 16 字节的语义 XRT 完全依赖，一点不能自定义。

### 4.1 寄存器 map

| 偏移 | 名字 | 访问 | 位段定义 |
|---|---|---|---|
| `0x00` | **CTRL** | R/W（写有副作用） | `[0]` ap_start；`[1]` ap_done；`[2]` ap_idle；`[3]` ap_ready；`[7]` auto_restart；其他保留 |
| `0x04` | **GIER** | R/W | `[0]` Global Interrupt Enable |
| `0x08` | **IP_IER** | R/W | `[0]` ap_done 中断使能；`[1]` ap_ready 中断使能 |
| `0x0C` | **IP_ISR** | R + W1C | `[0]` ap_done pending；`[1]` ap_ready pending |

其余位保留、读回 0、写忽略。

### 4.2 CTRL 各位的精确语义

#### `[0] ap_start`

**写行为**：
- 主机写 `0x01` → 硬件把内部 `ap_start` 触发拉起
- 主机写 `0x00` → 无副作用（不能强制清除）
- 硬件**自动清零**：当 kernel 完成（`ap_done` 拉起的同一或下一时钟）时，`ap_start` 被硬件清 0

**读行为**：回读当前硬件内部的 `ap_start` 状态位。主机通过检查这一位 == 0 来确认 kernel 已经进入运行/完成。

**W1S + 自清** 是标准语义。**level-triggered、无自清** 的实现是最常见的错误——kernel 会连续启动、进入死循环，或 XRT 无法判断第二次 `run` 是否已开始。

#### `[1] ap_done`

**语义**：kernel 完成一次运算的粘性标志。
- kernel 内部一个周期的 `ap_done_pulse` 出现 → 该位 set 到 1
- 主机读取 CTRL 时如果该位 = 1，硬件**在读操作被响应的同一周期清零**（读清语义，read-clear on-access）

**关键点**：一次读会消费 done 状态。主机因此不需要发 W1C 来清 done——但这也意味着**不能重复读、不能被两个 host 线程同时轮询同一 kernel**。

某些实现允许读不清，改由下一次 `ap_start` 清 done。XRT 支持这两种，只要行为一致。**推荐用「读清」**，是 Vitis HLS 生成代码的默认方式。

#### `[2] ap_idle`

**语义**：kernel 空闲标志。
- kernel 未运行时 = 1
- kernel 收到 `ap_start` 到 `ap_done` 之间 = 0

**为什么重要**：XRT 在下一次 `run(args)` 之前会**轮询 ap_idle == 1** 来确认可以覆盖 arg 寄存器。缺失或错实现 → XRT 永远认为 kernel 忙 → 第二次 run 永远 stuck。

**推荐实现**：
```verilog
ap_idle <= ~(ap_start_reg | running);   // running 从 ap_start pulse 拉起，ap_done pulse 清零
```

#### `[3] ap_ready`

**语义**：kernel 可接受下一次输入的标志（对 pipelined kernel 有用，标志「上一次 arg 已被消费，可以准备下一次 arg」）。
- 简单 kernel（非 pipelined）：`ap_ready` == `ap_done`
- Pipelined kernel（HLS `#pragma HLS pipeline`）：`ap_ready` 早于 `ap_done`

**RTL kernel 通常**把 `ap_ready` 简单地绑定成 `ap_done`，除非你确实实现了流水启动。

#### `[7] auto_restart`

**语义**：kernel 完成后自动重启（`ap_start` 保持高）。绝大多数场景不用；主机代码要显式支持才能安全使用。**推荐实现成 R/W 但内部忽略**（除非你的 kernel 明确支持流式重启）。

### 4.3 GIER / IP_IER / IP_ISR

**必须实现**——即使你不打算用中断，`v++` 也可能查这些寄存器的存在。

#### GIER (0x04)

- `[0]`：全局中断开关。主机写 1 使能，写 0 屏蔽。
- 复位后为 0。

#### IP_IER (0x08)

- `[0]`：ap_done 事件的中断使能
- `[1]`：ap_ready 事件的中断使能
- 复位后为 0。

#### IP_ISR (0x0C)

- `[0]`：ap_done pending。当 ap_done pulse 出现且 IP_IER[0]=1 时 set；主机 W1C（写 1 清 0）
- `[1]`：ap_ready pending。同上。
- 复位后为 0。

#### `interrupt` 输出的组合逻辑

```
interrupt = GIER[0] & ( (IP_ISR[0] & IP_IER[0]) | (IP_ISR[1] & IP_IER[1]) )
```

组合逻辑，中断在 pending 位被清零后**立即**回落——不需要主动 deassert。

### 4.4 XRT 的握手序列（读一遍就知道每个位干嘛）

```
Host                                    Kernel
  |                                        |
  |-- write args (0x10, 0x14, ...)  ------>|
  |-- write GIER = 0x1               ---->|
  |-- write IP_IER = 0x1              --->|
  |-- write CTRL = 0x1 (ap_start)     --->|
  |                                        |  ap_start latched
  |                                        |  ap_idle=0, running=1
  |                                        |  ... work ...
  |                                        |  ap_done pulse
  |                                        |  ISR[0]=1, ap_idle=1
  |                                        |  interrupt=1
  |<-- MSI (PCIe)  -----------------------|
  |-- read IP_ISR (== 0x1)            --->|
  |-- write IP_ISR = 0x1 (W1C)        --->|  ISR[0]=0, interrupt=0
  |-- read CTRL                       --->|  ap_done=1 (read-clear), ap_idle=1
  |-- run.wait() returns                   |
```

理解这一序列，就能 debug 掉 90% 的 XRT hang。

---

## 5. 用户参数寄存器（0x10 起）

### 5.1 布局规则

- 起始偏移固定 **`0x10`**
- **scalar 参数**：占 1 个 32-bit 寄存器
- **global buffer 指针参数**（64-bit）：占 2 个 32-bit 寄存器（低 32 + 高 32），且**起始偏移必须 8 字节对齐**（0x10、0x18、0x20…；如果前一个 arg 是 32-bit scalar，通常会在其后插入一个 pad 到 8 字节对齐）
- **stream 参数**（AXI4-Stream）：不占寄存器，走独立 AXI-Stream 端口

### 5.2 典型布局示例

假设 kernel 签名 `void kernel(int* in0, int* in1, int* out, int size, int mode)`：

| 偏移 | 内容 | 说明 |
|---|---|---|
| 0x00 | CTRL | |
| 0x04 | GIER | |
| 0x08 | IP_IER | |
| 0x0C | IP_ISR | |
| 0x10 | `in0` [31:0] | 64-bit 指针低半 |
| 0x14 | `in0` [63:32] | 64-bit 指针高半 |
| 0x18 | `in1` [31:0] | |
| 0x1C | `in1` [63:32] | |
| 0x20 | `out` [31:0] | |
| 0x24 | `out` [63:32] | |
| 0x28 | `size` | 32-bit scalar |
| 0x2C | (pad) | 保留 |
| 0x30 | `mode` | 32-bit scalar |

**指针占 2 个 32-bit register 是硬性要求**——即使物理上 kernel 只需要 40 位或 48 位地址（HBM 8 GB 只需 33 位）。XRT 传的永远是 64 位设备虚拟地址。

### 5.3 端口关联（`m_axi` port association）

每个「global buffer 指针」参数必须在 `kernel.xml` 里绑到一个 `m_axi` 端口。kernel 内部：

```verilog
// 主机写入的指针值锁存下来，作为 m_axi 的地址基
always @(posedge ap_clk) begin
  if (slv_reg_wren && addr == ARG0_LOW)  arg0_ptr[31:0]  <= WDATA;
  if (slv_reg_wren && addr == ARG0_HIGH) arg0_ptr[63:32] <= WDATA;
end

// arg0_ptr 送去 m_axi_gmem0 的地址生成逻辑（AR 或 AW 通道）
```

**arg 到 m_axi 端口是多对一映射**——多个指针 arg 可以共享同一个 `m_axi_gmem`（通过时序化访问），也可以每个 arg 一个独立端口（并发访问，占更多资源但带宽更好）。

### 5.4 写策略

- **主机在写 arg 之前必须先看到 `ap_idle == 1`**——XRT 自动检查
- **kernel 复位后主机写第一次 arg 之前，arg 寄存器内容未定义**（0 是常见的复位值，但不是契约）
- **kernel 内部使用 arg 值时**：推荐在 `ap_start` 触发的一瞬间把当前 arg 寄存器**锁存到内部影子寄存器**，之后 kernel 内部只用影子寄存器，避免运行中主机修改（虽然 XRT 不会这么干）

---

## 6. AXI4 主端口 `m_axi_<port>`

### 6.1 命名与端口集

命名前缀严格：`m_axi_<port_name>_<CHANNEL>_<SIGNAL>`。`<port_name>` 在 `kernel.xml` 声明。

信号集（**AXI4 full，非 AXI4-Lite**）：

| 通道 | 信号 | 位宽 |
|---|---|---|
| AW | AWID, AWADDR, AWLEN, AWSIZE, AWBURST, AWLOCK, AWCACHE, AWPROT, AWQOS, AWREGION, AWVALID, AWREADY | 见下 |
| W | WDATA, WSTRB, WLAST, WVALID, WREADY | 见下 |
| B | BID, BRESP, BVALID, BREADY | |
| AR | ARID, ARADDR, ARLEN, ARSIZE, ARBURST, ARLOCK, ARCACHE, ARPROT, ARQOS, ARREGION, ARVALID, ARREADY | 见下 |
| R | RID, RDATA, RRESP, RLAST, RVALID, RREADY | |

关键位宽约束：
- **`ARADDR` / `AWADDR` 必须 64 位**——Vitis 强制。即使实际 HBM 只需 33 位，也要走 64 位。
- **`ARID` / `AWID` / `BID` / `RID` 位宽**：1–16 位，同一端口 4 个 ID 位宽必须一致
- **`ARLEN` / `AWLEN` 8 位**：AXI4 支持 1–256 beat burst
- **`ARSIZE` / `AWSIZE` 3 位**：log2(bytes per beat)
- **`ARBURST` / `AWBURST` 2 位**：01 = INCR（最常用）
- **`WSTRB` 宽度 = DATA_WIDTH / 8**

### 6.2 数据位宽 `C_M_AXI_DATA_WIDTH`

|-|-|
| 允许值 | 32、64、128、256、512、1024（HBM 上限 512，DDR 上限 512） |
| 关系 | `WDATA` / `RDATA` 位宽 = `WSTRB * 8`（写通道） |

**性能大杀器**：
- 32 位 → HBM 单 PC 峰值带宽 ~450 MB/s（14.4 GB/s HBM PC 带宽的 3%）
- 256 位 → HBM 单 PC 峰值带宽 ~14.4 GB/s
- 512 位 → 一个 AXI 端口跨两个 PC，需要平台支持

**建议**：数据端口位宽起手 **256 或 512**，除非 kernel 内部数据通路窄且无扩宽收益。

### 6.3 读写方向声明

`kernel.xml` 里通过 `readWriteMode` 告诉 SmartConnect 这个端口只用什么方向：

- `read_only`：只用 AR/R 通道，platform 可省略 AW/W/B 布线
- `write_only`：只用 AW/W/B 通道
- `read_write`（默认）：五通道全用

**声明为单向后**，未使用通道的顶层 output 应该 **从顶层端口清单里删掉**，或者显式 tie-off。悬空 output 在 SmartConnect 侧可能触发 AXI 协议违例。

### 6.4 突发行为要求

- **`AWBURST` / `ARBURST` = INCR (01)**：Vitis 平台不接受 FIXED (00) 或 WRAP (10)
- **突发长度**：1–256 beat（`ARLEN` / `AWLEN` = burst_len - 1）
- **突发不能跨 4KB 边界**：AXI4 硬性规则，kernel 侧生成 AR/AW 时必须自己检查
- **`WLAST` / `RLAST` 精确对齐**：最后一个 beat 拉高，不能提前或延后
- **ID 一致性**：同一 outstanding 事务的 AR/R 或 AW/W/B ID 一致
- **outstanding 深度**：kernel 想并发多少事务由自己决定，SmartConnect 会重排响应；`ARID` / `AWID` 位宽决定最大并发深度（1 位 = 2 个并发事务）

### 6.5 复位后的主机端行为

复位后必须**立即**：
- 所有 valid 信号 = 0
- 没有 outstanding 事务（复位过程中如果有 outstanding，SmartConnect 状态与 kernel 状态不一致，会永久 hang）

**复位释放后**必须等 SmartConnect 稳定（通常几个周期）才能发第一个事务。安全做法：在 `ap_rst_n` 释放到 `ap_start` 触发之间插入一个空闲期。

---

## 7. 中断输出 `interrupt`

### 7.1 电平语义

- **level-sensitive**：拉高持续到主机响应
- **组合逻辑**：不能是 flop 输出（否则会引入一个周期延迟，无害但不 idiomatic）
- **来源**：`GIER & (ISR[0] & IER[0] | ISR[1] & IER[1])`，见 §4.3

### 7.2 无中断怎么办？

**必须实现**，即便你不打算用。原因：
- v++ 会在 kernel 端口清单里查 `interrupt` 存在
- XRT 可以用 polling 模式（`XCL_QUEUE_MODE_POLLING`）代替中断，但 `interrupt` 端口本身仍需存在

如果确实不用中断，把 `GIER` 常置 0 即可，`interrupt` 会一直是 0——功能上主机只能靠 polling `CTRL[1]` 判断 done。

### 7.3 XRT 中断路由

XRT 底层看到 `interrupt` 拉高 → PCIe MSI 触发 → 唤醒 `xrt::run::wait()` 的等待线程。清中断由主机的 ISR W1C 完成，不需要 kernel 主动 deassert。

---

## 8. 时钟与复位

### 8.1 时钟 `ap_clk`

- **单一时钟**：所有 AXI 端口、所有内部逻辑都用它
- **频率由 v++ 决定**：`link.cfg` 里 `[clock] freqHz=<Hz>:<cu>.ap_clk` 设定；平台会实际给出最接近的可用时钟
- **无 CDC**：kernel 内部不允许跨异步时钟域（因为只有一个时钟）
- **多时钟 kernel**：需要 `ap_clk_2`（更慢的辅助时钟）；这是不常用的高级功能，platform 需要显式支持，`kernel.xml` 需要额外声明

### 8.2 复位 `ap_rst_n`

- **同步复位，低电平有效**：由 platform 提供，与 `ap_clk` 同步
- **持续时间**：platform 保证至少 8 个 ap_clk 周期
- **kernel 侧的响应**：所有 flop 与 AXI valid 信号在 `ap_rst_n = 0` 期间被清 0
- **释放后行为**：不能立即发起 AXI 事务，见 §6.5

### 8.3 常见错误

- ❌ 用异步复位 `always @(posedge ap_clk or negedge ap_rst_n)`：SmartConnect 期望同步复位
- ❌ 内部自己造复位 pulse：会与 platform 复位竞争
- ❌ 高电平有效的 `ap_rst`：Vitis 只认低电平有效

---

## 9. `kernel.xml` 元数据

### 9.1 最小完整示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root versionMajor="1" versionMinor="6">
  <kernel name="acc_kernel"
          language="ip_c"
          vlnv="user.org:user:acc_kernel:1.0"
          attributes=""
          preferredWorkGroupSizeMultiple="0"
          workGroupSize="1"
          interrupt="true"
          hwControlProtocol="ap_ctrl_hs">

    <ports>
      <port name="s_axi_control" mode="slave"
            range="0x1000" dataWidth="32"
            portType="addressable" base="0x0"/>

      <port name="m_axi_gmem0" mode="master"
            range="0xFFFFFFFFFFFFFFFF" dataWidth="32"
            portType="addressable" base="0x0"/>

      <port name="m_axi_gmem1" mode="master"
            range="0xFFFFFFFFFFFFFFFF" dataWidth="32"
            portType="addressable" base="0x0"/>

      <port name="m_axi_gmem2" mode="master"
            range="0xFFFFFFFFFFFFFFFF" dataWidth="32"
            portType="addressable" base="0x0"/>
    </ports>

    <args>
      <arg name="in0"    addressQualifier="1" id="0" port="m_axi_gmem0"
           hostOffset="0x0" hostSize="0x8" offset="0x10" size="0x8" type="int*"/>
      <arg name="in1"    addressQualifier="1" id="1" port="m_axi_gmem1"
           hostOffset="0x0" hostSize="0x8" offset="0x18" size="0x8" type="int*"/>
      <arg name="out"    addressQualifier="1" id="2" port="m_axi_gmem2"
           hostOffset="0x0" hostSize="0x8" offset="0x20" size="0x8" type="int*"/>
      <arg name="size"   addressQualifier="0" id="3" port="s_axi_control"
           hostOffset="0x0" hostSize="0x4" offset="0x28" size="0x4" type="int"/>
      <arg name="mode"   addressQualifier="0" id="4" port="s_axi_control"
           hostOffset="0x0" hostSize="0x4" offset="0x30" size="0x4" type="int"/>
    </args>

  </kernel>
</root>
```

### 9.2 关键字段

| 字段 | 含义 |
|---|---|
| `kernel@name` | 顶层 module 名，主机 `xrt::kernel(dev, uuid, name)` 用 |
| `kernel@hwControlProtocol` | 控制协议，**必须 `ap_ctrl_hs`**（Handshake，即 §4 描述的 CTRL/GIER/IER/ISR 集） |
| `kernel@interrupt` | `true` 表示 kernel 有 `interrupt` 端口 |
| `port@name` | 端口前缀名，必须与顶层 signal 前缀严格匹配 |
| `port@mode` | `slave` (AXI-Lite) 或 `master` (AXI4) |
| `port@range` | slave: 寄存器地址空间大小；master: 可访问的地址范围（全部 = 0xFF...） |
| `port@dataWidth` | 位宽 |
| `arg@addressQualifier` | `0` = scalar，`1` = global memory 指针，`4` = stream |
| `arg@port` | 该 arg 关联的端口名（scalar 指向 `s_axi_control`；指针指向对应 m_axi） |
| `arg@offset` | 该 arg 在 s_axi_control 的寄存器偏移（必须与 RTL 内部译码逻辑一致） |
| `arg@size` | 字节数（scalar = 4，pointer = 8） |
| `arg@hostSize` | 主机侧类型的字节数（通常等于 `size`） |

### 9.3 可选：`readWriteMode`

在新版 Vitis 中，`<arg>` 或 `<port>` 里可加：
```xml
<arg name="in0" ... port="m_axi_gmem0" ... readWriteMode="read_only"/>
```
或对端口整体：
```xml
<port name="m_axi_gmem0" mode="master" readWriteMode="read_only" .../>
```

告诉 SmartConnect 优化布线，删掉未使用的通道。见 §6.3。

### 9.4 校验

`kernel.xml` 错一个字节，v++ 就报「参数错位、寄存器不对齐」。**必须**：
- 每个 arg 的 `offset` 与 RTL 内部译码完全一致
- pointer arg 的 `offset` 8 字节对齐
- port 数与顶层 module 端口数、名字一一对应
- `hwControlProtocol` 与 CTRL 寄存器实现一致（HLS 也支持 `ap_ctrl_chain` / `ap_ctrl_none`，RTL kernel 通常用 `ap_ctrl_hs`）

---

## 10. 打包 `.xo` 的物理组织

### 10.1 输入结构

`package_xo` 需要三样东西：

1. **一个 Vivado IP 目录**（含 `component.xml`，可通过 `ipx::package_project` 自动生成，或手工挂载 IP-XACT）
2. **`kernel.xml`**（§9）
3. **可选的 Tcl 脚本**（做额外 IP 配置、寄存器）

调用：
```tcl
package_xo -xo_path acc_kernel.xo \
           -kernel_name acc_kernel \
           -ip_directory ./ip \
           -kernel_xml kernel.xml \
           [-kernel_files rtl_top.v ...]
```

### 10.2 IP 打包的信号识别

Vivado IP packager 会**扫描 module 端口**，按 §2 命名前缀自动推断：
- 前缀 `ap_` → 控制信号
- 前缀 `s_axi_<name>` → AXI-Lite slave 接口
- 前缀 `m_axi_<name>` → AXI4 master 接口
- 端口 `interrupt` → 中断输出

如果识别失败（信号缺失、命名错误、位宽异常），packager 把接口降级为「无关信号集合」，v++ link 报「不能自动连接」。

### 10.3 输出

`acc_kernel.xo` 是一个 zip 容器，内部结构：
```
acc_kernel.xo (zip)
├── kernel.xml
└── ip_repo/
    └── <IP definition + RTL sources>
```

可以 `unzip -l acc_kernel.xo` 查看，或用 `xclbinutil --info` 查看链接后的 xclbin。

---

## 11. XRT 主机侧调用如何映射回 RTL

一段最小 XRT native API 主机代码 + 对应发生在 RTL 的事情：

```cpp
auto device   = xrt::device(0);
auto uuid     = device.load_xclbin("acc.xclbin");
auto kernel   = xrt::kernel(device, uuid, "acc_kernel");
// ↑ RTL 侧：无动作。XRT 把 xclbin 里的 CU 元数据读进来，
//    包括 CU 的基地址（AXI-Lite 的 MMIO 起点）、arg offset 表。

auto bo0 = xrt::bo(device, N*4, kernel.group_id(0));  // arg 0 关联的 m_axi 端口的 bank
auto bo1 = xrt::bo(device, N*4, kernel.group_id(1));  // arg 1 关联的 m_axi 端口的 bank
auto bo2 = xrt::bo(device, N*4, kernel.group_id(2));  // arg 2 关联的 m_axi 端口的 bank
// ↑ RTL 侧：无动作。这一步只在 HBM/DDR bank 分配缓冲区，返回一个物理地址。

bo0.write(host_in0.data());
bo0.sync(XCL_BO_SYNC_BO_TO_DEVICE);
// ↑ RTL 侧：PCIe DMA 把数据从主机内存搬到 bo0 对应的 HBM 位置。
//    与 kernel 无关，kernel 复位后没启动，AR/AW 都是 0。

auto run = kernel(bo0, bo1, bo2, N, 5);
// ↑ RTL 侧：一系列 AXI-Lite 写。XRT 按 kernel.xml 的 offset 表：
//    - MMIO 写 0x10, 0x14 = bo0 的低/高 32 位（in0 指针）
//    - MMIO 写 0x18, 0x1C = bo1 的低/高 32 位（in1 指针）
//    - MMIO 写 0x20, 0x24 = bo2 的低/高 32 位（out 指针）
//    - MMIO 写 0x28       = N（scalar arg 3）
//    - MMIO 写 0x30       = 5（scalar arg 4）
//    - MMIO 写 0x04       = 0x1（GIER）
//    - MMIO 写 0x08       = 0x1（IP_IER）
//    - MMIO 写 0x00       = 0x1（CTRL.ap_start）
// RTL 侧：
//    - AXI-Lite slave 逐个响应写，寄存器更新
//    - 最后一次写触发 ap_start=1，kernel 拉起 ap_idle=0，进入运行
//    - 内部逻辑用锁存的 arg 指针，往 m_axi_gmem0/1 发 AR，往 m_axi_gmem2 发 AW

run.wait();
// ↑ RTL 侧：正在跑。跑完后 ap_done pulse，ISR[0] set，interrupt 拉高。
//    XRT 收到 MSI，读 IP_ISR 确认 done，W1C 清 ISR，run.wait() 返回。

bo2.sync(XCL_BO_SYNC_BO_FROM_DEVICE);
bo2.read(host_out.data());
// ↑ RTL 侧：无动作。PCIe DMA 把 HBM 里的 out buffer 搬回主机。
```

---

## 12. 完整最小示例：CTRL 寄存器参考实现

`vadd_control_s_axi.v` 骨架（HLS 生成的形态，简化版）：

```verilog
module vadd_control_s_axi #(
    parameter C_S_AXI_ADDR_WIDTH = 7,
    parameter C_S_AXI_DATA_WIDTH = 32
)(
    input  wire                              ACLK,
    input  wire                              ARESET,          // 高有效同步复位
    input  wire                              ACLK_EN,

    // AXI-Lite slave（略，标准五通道）
    input  wire [C_S_AXI_ADDR_WIDTH-1:0]     AWADDR,
    input  wire                              AWVALID,
    output wire                              AWREADY,
    // ... (其余 AXI-Lite 信号)

    // 到用户 kernel 逻辑
    output wire                              ap_start,
    input  wire                              ap_done,
    input  wire                              ap_ready,
    input  wire                              ap_idle,
    output wire                              interrupt,

    // 用户参数（示例）
    output wire [63:0]                       arg_in0,
    output wire [63:0]                       arg_in1,
    output wire [63:0]                       arg_out,
    output wire [31:0]                       arg_size
);

// 内部寄存器
reg        int_ap_start;
reg        int_ap_done;
reg        int_auto_restart;
reg        int_gie;                  // GIER[0]
reg  [1:0] int_ier;                  // IP_IER[1:0]
reg  [1:0] int_isr;                  // IP_ISR[1:0]
reg [63:0] int_arg_in0;
reg [63:0] int_arg_in1;
reg [63:0] int_arg_out;
reg [31:0] int_arg_size;

// 寄存器偏移
localparam
    ADDR_AP_CTRL  = 7'h00,
    ADDR_GIE      = 7'h04,
    ADDR_IER      = 7'h08,
    ADDR_ISR      = 7'h0C,
    ADDR_IN0_LO   = 7'h10,
    ADDR_IN0_HI   = 7'h14,
    ADDR_IN1_LO   = 7'h18,
    ADDR_IN1_HI   = 7'h1C,
    ADDR_OUT_LO   = 7'h20,
    ADDR_OUT_HI   = 7'h24,
    ADDR_SIZE     = 7'h28;

wire        w_hs;       // write handshake
wire [6:0]  waddr;      // latched write address
// ... AXI-Lite 五通道 FSM 略（用标准写法）

// ============ ap_start：写触发 + 硬件自清 ============
always @(posedge ACLK) begin
    if (ARESET)
        int_ap_start <= 1'b0;
    else if (ACLK_EN) begin
        if (w_hs && waddr == ADDR_AP_CTRL && WSTRB[0] && WDATA[0])
            int_ap_start <= 1'b1;                     // 主机写 1 触发
        else if (ap_ready)                            // kernel 已消费启动信号
            int_ap_start <= int_auto_restart;         // 除非 auto_restart，否则清 0
    end
end

// ============ ap_done：kernel pulse set，主机读时清 ============
always @(posedge ACLK) begin
    if (ARESET)
        int_ap_done <= 1'b0;
    else if (ACLK_EN) begin
        if (ap_done)
            int_ap_done <= 1'b1;
        else if (ar_hs && araddr == ADDR_AP_CTRL)     // 主机读 CTRL 时清 done
            int_ap_done <= 1'b0;
    end
end

// ============ CTRL 读回值 ============
wire [31:0] ctrl_read = {24'h0, int_auto_restart, 3'b0,
                         ap_ready, ap_idle, int_ap_done, int_ap_start};

// ============ GIER ============
always @(posedge ACLK) begin
    if (ARESET) int_gie <= 1'b0;
    else if (ACLK_EN && w_hs && waddr == ADDR_GIE && WSTRB[0])
        int_gie <= WDATA[0];
end

// ============ IP_IER ============
always @(posedge ACLK) begin
    if (ARESET) int_ier <= 2'b0;
    else if (ACLK_EN && w_hs && waddr == ADDR_IER && WSTRB[0])
        int_ier <= WDATA[1:0];
end

// ============ IP_ISR：kernel pulse set，主机 W1C 清 ============
always @(posedge ACLK) begin
    if (ARESET) int_isr <= 2'b0;
    else if (ACLK_EN) begin
        if (ap_done  && int_ier[0]) int_isr[0] <= 1'b1;
        if (ap_ready && int_ier[1]) int_isr[1] <= 1'b1;
        if (w_hs && waddr == ADDR_ISR && WSTRB[0]) begin
            if (WDATA[0]) int_isr[0] <= 1'b0;         // W1C
            if (WDATA[1]) int_isr[1] <= 1'b0;
        end
    end
end

assign interrupt = int_gie & (|(int_isr & int_ier));

// ============ 用户参数寄存器（略，同样的 slv_reg 模式） ============

assign ap_start = int_ap_start;
assign arg_in0  = int_arg_in0;
assign arg_in1  = int_arg_in1;
assign arg_out  = int_arg_out;
assign arg_size = int_arg_size;

endmodule
```

这个骨架**可以直接被 XRT 驱动**，是所有 RTL kernel 控制层的最小充分实现。真实的 HLS 生成代码更长，主要区别在 AXI-Lite 五通道的握手 FSM 展开和更细的时序处理。

---

## 13. 契约核对清单

在把一份 RTL 交给 `package_xo` 之前，逐条对照：

**顶层端口**
- [ ] 时钟叫 `ap_clk`，一个
- [ ] 复位叫 `ap_rst_n`（不是 `ap_resetn`），一个，低有效同步
- [ ] AXI-Lite slave 前缀 `s_axi_control`，21 个信号齐全（含 AWPROT/ARPROT）
- [ ] AXI4 master 前缀 `m_axi_<name>`，信号齐全（未用通道要么删要么 tie-off）
- [ ] `interrupt` 端口存在（即使不用）
- [ ] 顶层无 inout、无外部 IO、无异步复位、无多时钟

**控制寄存器（0x00–0x0F）**
- [ ] CTRL[0] `ap_start`：W1S 触发 + kernel done 自清
- [ ] CTRL[1] `ap_done`：kernel pulse set + 读清（或下一次 start 清）
- [ ] CTRL[2] `ap_idle`：未运行 = 1，运行中 = 0
- [ ] CTRL[3] `ap_ready`：简单 kernel 可与 ap_done 相同
- [ ] CTRL[7] `auto_restart`：至少可 R/W，内部可选实现
- [ ] GIER[0] R/W
- [ ] IP_IER[1:0] R/W
- [ ] IP_ISR[1:0] pulse set + W1C
- [ ] `interrupt = GIER & (ISR & IER)`

**参数寄存器（0x10 起）**
- [ ] Global buffer 指针占 2 个 32-bit 寄存器，起始偏移 8 字节对齐
- [ ] Scalar 占 1 个 32-bit 寄存器
- [ ] 内部译码地址位宽 == 顶层 `AWADDR`/`ARADDR` 位宽
- [ ] `kernel.xml` 里每个 arg 的 offset 与 RTL 一致

**AXI4 master**
- [ ] `AWADDR`/`ARADDR` 64 位
- [ ] `AWBURST`/`ARBURST` = 01 (INCR)
- [ ] 突发不跨 4KB 边界
- [ ] `WLAST`/`RLAST` 精确对齐
- [ ] 复位时所有 valid = 0，无 outstanding
- [ ] 数据位宽是 32/64/128/256/512 之一

**kernel.xml**
- [ ] `<kernel>` name 与 module 名一致
- [ ] `hwControlProtocol="ap_ctrl_hs"`
- [ ] 每个 port 与顶层前缀一一对应
- [ ] 每个 arg 的 offset / size / port 与 RTL 一致
- [ ] Pointer arg 的 `addressQualifier="1"`，scalar 的 `addressQualifier="0"`

**行为**
- [ ] 复位序列合理（同步、低有效、pulse ≥ 8 clk）
- [ ] Kernel 不主动生成复位
- [ ] Kernel 不假设 arg 寄存器复位值
- [ ] Kernel 在 `ap_start` 上升沿锁存 arg 值到内部影子寄存器

---

## 14. 常见陷阱

### 14.1 打包成功但 `run.wait()` 永不返回

**排查顺序**：
1. `interrupt` 是否真的拉高了？—— ILA 抓一下
2. `ISR` 是否 set 了？—— 是否 `IP_IER` 主机写了 1？
3. `ap_done` pulse 是否只有一个周期？—— 太宽或没出现都会挂
4. `ap_idle` 是否复位后就是 1？—— 若一直是 0，XRT 认为 kernel 忙，不会启动

### 14.2 打包成功但结果全错

**排查顺序**：
1. `kernel.xml` 里 arg offset 与 RTL 译码逻辑是否一致——**最常见**
2. Pointer arg 是否被当成 32 位读？—— 忘记读高 32 位地址
3. `AWBURST`/`ARBURST` 是不是 INCR？—— WRAP 或 FIXED 会读到错数据
4. 突发跨了 4KB？—— 检查 kernel 的地址生成逻辑
5. m_axi 的 `WSTRB` 是不是全 1？—— 有时 kernel 只 assert 部分 strobe，数据被截断

### 14.3 v++ link 报「无法自动连接」

**原因**：接口未被 IP packager 识别为标准接口。
**排查**：
- 检查顶层信号命名，逐字符匹配 §2.1
- 检查 AXI-Lite 是否缺 AWPROT/ARPROT
- 检查 m_axi 是否缺某个通道信号
- 打开 Vivado GUI 打开 IP，看接口列表是否分类正确（应该看到 `s_axi_control` 是 AXI4-Lite，`m_axi_gmem0` 是 AXI4）

### 14.4 时序总收不敛

**通用建议**（与 XRT 契约无关，但常一起出现）：
- Kernel 频率起手 250 MHz，不要一开始就冲 400
- m_axi 端口位宽如果宽（256/512），SmartConnect 需要更多寄存器，`slr` 分配要留余量
- 跨 SLR 布线：`link.cfg` 里 `slr=cu:SLR0` 强制指定
- 复杂组合逻辑（浮点除法、exp、log）后加 pipeline register

### 14.5 主机端 `xrt::kernel(...)` 抛异常「no such kernel」

**原因**：
- `kernel.xml` 里的 `name` 与主机代码字符串不一致
- xclbin 里没这个 kernel（拼接错了 xo）

**排查**：
```bash
xclbinutil --info -i acc.xclbin | grep "Kernel Name"
```

### 14.6 主机端 `xrt::bo` 抛异常「invalid group_id」

**原因**：`kernel.group_id(N)` 中的 N 与 kernel.xml arg 表不匹配，或 arg 不是 global buffer。
**修复**：`kernel.group_id(N)` 只对 pointer arg 有意义，N 是 arg 索引（从 0 数），传给它的是**该 arg 关联的 m_axi 端口对应的 memory bank ID**。

### 14.7 kernel 在 hw_emu 通，上板不通

**常见原因**：
- 时序违例但你没看报告：`v++ -l --save-temps`，找 `*_utilization_placed.rpt` 与 `*_timing_summary_routed.rpt`
- HBM bank 冲突：多个 m_axi 端口挂同一个 HBM PC，仿真时序对了但真硬件抢总线
- 复位序列不同：hw_emu 里复位周期可能更长，掩盖了短复位问题

---

## 15. 参考资料

- **UG1393**：Vitis Unified Software Platform Documentation
  - RTL Kernels 章：packaging、kernel.xml、端口约束
  - Migrating and Optimizing 章：性能调优
- **UG1023**：UltraScale Memory IP（HBM/DDR 接口细节）
- **XRT 源码**：`Xilinx/XRT` GitHub 仓库，`src/runtime_src/core/common/` 目录
- **Vitis Accel Examples**：`Xilinx/Vitis_Accel_Examples` GitHub 仓库，`rtl_kernels/` 子目录里有完整可编译的最小 RTL kernel 参考
- **UG1483**：Vivado IP-XACT 打包指南（`package_xo` 底层依赖）

---

## 一句话总结

**契约的核心**：`ap_ctrl_hs` 协议 + 前 16 字节的 CTRL/GIER/IER/ISR 语义 + `interrupt` 组合逻辑 + AXI-Lite 与 AXI4 端口的完整信号集 + `kernel.xml` 中 arg offset 与 RTL 译码逐字节一致。

其余都是这 5 项的必然推论。
