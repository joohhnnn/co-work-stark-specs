# Cairo-VM（执行层客户端）

## 1. Cairo-VM 是什么？

* **定位** ：StarkWare 为 Cairo 语言实现的虚拟机 / 解释器。
* **功能** ：负责加载 Cairo 字节码、初始化内存与内置（builtins），解析并执行 hint，驱动指令循环并给出执行结果（寄存器、内存、trace、pie 等）。
* **依赖信息** ：在 sequencer 的 `Cargo.toml`（约第 170 行）中可见 `cairo-vm = "2.0.1"`，说明 VM 作为普通 Crate 依赖被直接链接到二进制中。

## 2. 在 sequencer 中的两大使用场景

| 场景 | 代码位置 | 作用 |
|-----|---------|------|
| Starknet-OS | `crates/starknet_os/src/runner.rs` | 区块结束时执行 "操作系统" 程序，完成状态承诺、L1 交互等 |
| Blockifier | `crates/blockifier/src/execution/entry_point_execution.rs` | 区块内逐笔交易执行合约入口点，收集 state diff / events |

### 2.1 Starknet-OS 层面

[runner.rs:30-60](https://github.com/starkware-libs/sequencer/blob/8cd9dd20/crates/starknet_os/src/runner.rs#L30-L60)

```rust
// (摘自 runner.rs 片段)
let os_program = Program::from_bytes(compiled_os, Some(cairo_run_config.entrypoint))?;
let mut cairo_runner = CairoRunner::new(&os_program, …)?;
let end = cairo_runner.initialize(allow_missing_builtins)?;
cairo_runner.run_until_pc(end, &mut snos_hint_processor)?;
```

**关键点解析（runner.rs::run_os）**

* `Program::from_bytes` 将 **OS 字节码** 解析为 Cairo 程序结构。
* `CairoRunner::new` 按 `LayoutName` 创建运行器，设置 trace/proof 等运行模式。
* `initialize` 在 VM 内部分配内存段并返回 **结束 PC**，供后续 `run_until_pc` 使用。
* `run_until_pc` 驱动 Cairo-VM 指令循环直到到达结束 PC，过程中 Hint 由 `SnosHintProcessor` 处理。
* `end_run` / `finalize_segments` 完成收尾与证明相关段落，生成 `CairoPie` 等输出数据。

流程：加载 OS 字节码 → 初始化 Cairo-VM → 运行直至结束 → 输出结果（commitment 等）。

### 2.2 Blockifier 层面

[entry_point_execution.rs:30-55](https://github.com/starkware-libs/sequencer/blob/8cd9dd20/crates/blockifier/src/execution/entry_point_execution.rs#L30-L55)

```rust
// (摘自 entry_point_execution.rs 片段)
pub struct VmExecutionContext<'a> {
    pub runner: CairoRunner,
    pub syscall_handler: SyscallHintProcessor<'a>,
    …
}
```

**关键点解析（entry_point_execution.rs::VmExecutionContext）**

* `VmExecutionContext` 聚合 **CairoRunner + SyscallHintProcessor**，并持有 `Relocatable` 指向首个 syscall 区段，作为一次合约调用的完整上下文。
* 构造时，会按合约的 `builtins` 调用 `runner.initialize_function_runner_cairo_1`，确保内置顺序与字节码一致。
* `SyscallHintProcessor` 维护对 sequencer 状态树的可变引用，实现 **storage_read / write、emit_event** 等系统调用。
* 调用完成后通过 `finalize_execution` 汇总资源消耗（`ExecutionResources`）、gas 统计以及返回数据。

### 2.3 Execution Utils 中的 VM 辅助函数

[execution_utils.rs:160-180](https://github.com/starkware-libs/sequencer/blob/8cd9dd20/crates/blockifier/src/execution/execution_utils.rs#L160-L180)

```rust
pub fn relocatable_from_ptr(vm: &VirtualMachine, ptr: &mut Relocatable) -> Result<Relocatable, VirtualMachineError> {
    let value = vm.get_relocatable(*ptr)?;
    *ptr = (*ptr + 1)?;
    Ok(value)
}

pub fn felt_from_ptr(vm: &VirtualMachine, ptr: &mut Relocatable) -> Result<Felt, VirtualMachineError> {
    let felt = vm.get_integer(*ptr)?.into_owned();
    *ptr = (*ptr + 1)?;
    Ok(felt)
}
```

**关键点解析（execution_utils.rs）**

* 这些工具函数为 Hint / Syscall 解析提供 **安全且简洁** 的指针解引用能力，负责移动 `ptr` 并返回对应类型。
* 抽象出 VM 原生 API 里的错误处理，统一转化为 `VirtualMachineError`，便于在上层进行错误追踪。
* 通过 `felt_from_ptr` / `relocatable_from_ptr` 保持 **指针自增** 语义，简化解析循环逻辑。

## 3. 关键组件速览

* **CairoRunner**：高级运行器，封装了程序装载、段管理、trace 生成等（示例见 `runner.rs:6`）。
* **VirtualMachine**：底层执行引擎，管理寄存器、内存及指令循环（`execution_utils.rs:174-188` 有直接调用）。
* **SyscallHintProcessor**：系统调用处理器，将 VM 内部 syscall 与 sequencer 业务逻辑（状态读写、事件、L1 消息等）对接（`entry_point_execution.rs:42`）。

## 4. 依赖关系与引用概览

* `cairo_vm` 自身是独立 Crate，被 sequencer 直接依赖；并非独立进程。
* 通过 `grep "use cairo_vm"` 可在 80+ 个源文件中发现引用，覆盖 `starknet_os`、`blockifier`、`apollo_rpc_execution` 等子 crate。

## 5. 小结

Cairo-VM 并不单独运行，而是作为 Rust 库嵌入到 sequencer 的各个组件中：

```
交易 → Blockifier (调用 Cairo-VM 执行入口点) → 收集 state diff / events
   └─ 区块结束 → Starknet-OS (再次调用 Cairo-VM 做状态承诺)
```

这一执行层客户端负责所有 Cairo 字节码的实际执行工作，是 sequencer 架构的计算核心。