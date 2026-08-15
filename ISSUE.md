# [Bug] group_token_vec_mt 无法仿真：addtpc 基址错位 → 标量 store 写入 .text（gfrun AssertNotTextStore / gfsim SIGABRT）—— addtpc page-rounding 缺陷的标量 store 路径实例

## 摘要

`SuperNPUBench/benchmark/one-level-arch/kernels/group_token_vec_mt`（4-PE SPMD 多线程
MoE Token 分组算子，TLOAD + 标量计数）编译通过，但在当前模型下**两条仿真链路均失败**：

- **gfrun**：main 执行到 Phase-1 清零段时，`addtpc` 推出的基址错位（0x1127e），
  `sdi` 清零 store 命中 `.text` 段（0x112b6），触发 `AssertNotTextStore` 中止；
- **gfsim**：SIGABRT（signal 6）。

根因定位：崩溃指令使用 **`addtpc`（PC-relative 基址）+ `subi`** 生成数据对象地址，
其值按"text 紧接上一段"的布局假设预编码（`addtpc 19` → pc+42 = 0x1148c → −522 =
0x1127e）；实际链接布局把 `.text` 页对齐到 **0x112a4**（上一段内容止于 0x102a4，
中间 4KB 页对齐空洞），预编码位移未回填 → 基址落在空洞上 → store 写进 text。

**非新问题**：与既存模型侧缺陷同族，本报告为【标量 store 基址】路径的独立可复现实例：
- #191（TLOAD 读错地址：B.IOR 操作数顺序 + **addtpc page rounding 仍按 PTO v0.2 假设**）
  —— 同一 addtpc 解码契约缺陷；#191 已覆盖 TLOAD 面，本报告补充标量访存面；
- #186（deepseek 11 ELF 写 text 段被拒，`gAllowTextStore` 断言）—— 同一断言族；
- #195（PC-relative 常量读取返回 0）—— 同一 addtpc/PC-rel 家族第三表现；
- 工具链侧 linx-toolchain-build#6（`addtpc+lwi.u` 常量池位移 vs 0x1000 对齐）——
  同一契约的编译侧表现。

## 环境

| 项 | 值 |
|---|---|
| 算子仓 | `SuperNPUBench`（v300_tag 工作区），`benchmark/one-level-arch/kernels/group_token_vec_mt` + `test/kernel/multi_thread/group_token_vec` |
| 工具链 | `linx-toolchain-build/output/linx_blockisa_llvm_musl`（clang 15.0.4，llvm-project `temp/shared-tload-integration-20260811 @ eb64de8af`） |
| 仿真器 | `SuperScalarModel` @ `feat/pto-v058-adaptation` `319294ff`：`bin/gfrun` / `bin/gfsim` |
| 运行参数 | `-s softcore.multiThreadNum=4`（4-PE SPMD 必需） |
| 日期 | 2026-08-15 |

## 复现步骤

```bash
# 免编译（材料包 elfs/ + sim/ 已内置）：
cd issue_repro_group_token_vec_mt && bash repro.sh

# 或源码编译：
cd <ws>/SuperNPUBench/benchmark/one-level-arch/test/kernel/multi_thread/group_token_vec
make TESTCASE=group_token_vec_mt COMPILER_DIR=<linx 工具链 bin>   # 编译通过, ELF 6.4KB
<ws>/SuperScalarModel/bin/gfrun -f <elf> -s softcore.multiThreadNum=4
<ws>/SuperScalarModel/bin/gfsim -f <elf> -s softcore.multiThreadNum=4
```

### 期望

gfrun 正常跑完（`Success to Reach the End of Benchmark!`），gfsim 打印完 SuperScalar Report。

### 实际

gfrun：

```text
gfrun: illegal instruction: ASSERTION FAILED: false
SoftCore store targets ELF text pc=0x1146e addr=0x112b6 size=8 data=0x0
M68626  |TPC:0x1146e  I4  T0 L0 |zero:0x0 |t#1:0x1127e |56
        |sdi zero, t#1, 56 |bin: 0xf803059 AccMemAddr 0x112b6,
        func AssertNotTextStore, file emulator/engine/AaccelssMemoryEngine.cpp:12
```

gfsim：`FATAL: gfsim received signal 6`（SIGABRT，std::terminate）。

## 定位分析

### 1. 崩溃指令链（Phase-1 cntLocal 清零段，main/内核序言）

```asm
11462: addtpc 19, ->t            ; 模型按 pc+imm 解码: t = 0x11462+0x2a = 0x1148c
11466: sdi.u  zero, [t#1, -522]  ; 首条清零 store（同样落空洞/越界方向）
1146a: subi   t#1, 522, ->t      ; t = 0x1148c - 522 = 0x1127e  ← 期望的数据对象基址
1146e: sdi    zero, [t#1, 56]    ; ★ 崩溃: 0x1127e + 56 = 0x112b6 = .text 段内
11472..1148a: sdi × 7（连续 64B 清零，全部指向 text）
```

- 段布局：`.text` 起点 **0x112a4**；上一段内容止于 0x102a4（.eh_frame），两者之间为
  **4KB 页对齐空洞**；崩溃基址 0x1127e 恰在空洞内（text 起点下方 0x26）。
- 编译器假设"text 紧接上一段（≈0x102a4）"时，`pc+偏移-位移` 恰好指向目标数据对象
  （零初始化区域）；lld 页对齐把 text 推到 0x112a4，预编码位移未回填。

### 2. 判别实验（排除 .uw / ssrget 家族干扰）

| 实验 | 结果 |
|---|---|
| 158 修复版（`02198ac5`，`.uw`/`.not` 解码）构建 gfrun 重跑 | **同样 AssertNotTextStore（崩溃点逐字节一致）** → 与 issue 158 机制无关 |
| 分支最新（`e3f9c13e`，含 ADDTPC 解码修复）构建 gfrun 重跑 | 行为变化：AssertNotTextStore → **SIGSEGV** → addtpc 解码参与该地址链（与 #191 同源） |
| 本 ELF 同时含 ssrget×3（@0x1148e/0x11566/0x11726，issue 179 族）与 `.uw`（@0x1154a/0x11764，issue 158 族） | 均位于崩溃点之后，非本次引爆点；修复本问题后需继续面对（multi_thread 套件 179 已登记） |

### 3. 结论

`addtpc` 解码/布局契约未适配 v0.58 工具链产物（同 #191/#195），其在**标量 store
基址**路径上的体现使任何配了 0x1000 对齐段布局的 ELF 都可能把清零/初始化 store
打进 .text。模型当前实现（319294ff）未含 #191 与 `addtpc page offset`（e3f9c13e）
修复。

## 修复建议

1. 按 #191 的 addtpc page-rounding 语义精确复算（覆盖**标量访存地址**面，不止 TLOAD）；
2. 修复后回归：multi_thread 套件全部成员（含 179 D1/D2 / 158 `.uw` / 本报告三路叠加）；
3. 工具链侧（linx-toolchain-build#6）同步核对 PC-relative 常量池/位移的段对齐回填。

## 复现材料

随包：预编译 ELF、内置 gfrun/gfsim、`repro.sh`、源码（kernel hpp + harness）、
证据（build/gfrun/gfsim 日志、反汇编、热循环 trace、段表、叠加缺陷扫描）。
解压后 `bash repro.sh` 单目录复现，无需工具链。