# Lab 1：多道程序与分时多任务

实验分支：`ch3`。本文依据当前仓库源码、实际修改和本地官方检查器结果整理。

## 一、功能总结（200 字以内）

实现编号为 410 的 sys_trace 系统调用，支持读取指定地址的一个字节、写入 data 的低 8 位，以及查询当前任务的系统调用次数。任务管理器使用以“任务编号、系统调用编号”为键的 BTreeMap 保存计数，在统一分发入口先计数再执行，保证当前查询也计入统计、各任务记录相互独立。未知请求返回 -1，未调用过的系统调用次数返回 0。

## 二、实验环境

| 项目 | 配置 |
| --- | --- |
| 操作系统 | Ubuntu 24.04 |
| Shell | zsh |
| 内核 Rust 工具链 | `nightly-2024-05-02` |
| 目标架构 | `riscv64gc-unknown-none-elf` |
| cargo-binutils | `0.4.0` |
| QEMU | 系统软件包提供的 `8.2.2` |
| SBI 实现 | 仓库附带的 RustSBI Prototyper jump 固件，RustSBI `0.4.1` |

固件信息由启动日志和 `bootloader/rustsbi-qemu.toml` 交叉确认。固件在 M 模式初始化机器，再将控制权交给 `0x80200000` 处的内核，使其在 S 模式执行。固件构建工具链与内核工具链是不同配置，日常实验直接使用仓库提供的固件。

## 三、实现思路

### 1. 字节读写

修改 `os/src/syscall/process.rs` 中的 `sys_trace()`：

- 请求为 0：将 `id` 转换为 `*const u8`，使用 `read_volatile()` 读取一个字节，转换为 `isize` 返回，忽略 `data`。
- 请求为 1：将 `id` 转换为 `*mut u8`，使用 `write_volatile(data as u8)` 写入一个字节，返回 0。
- 请求为 2：读取当前任务对应系统调用的累计次数。
- 其他请求：返回 -1。

将 `data` 转换为 `u8` 会保留最低 8 位。读写使用裸指针，因此需要 `unsafe`；`volatile` 保证实际发出相应访问，但不提供地址验证、内存隔离或并发同步。本章按照练习要求不实现地址安全检查，不能将该接口视为完整的安全内存访问接口。

### 2. 按任务保存计数

在 `os/src/task/mod.rs` 的 `TaskManagerInner` 中增加 `syscall_counts`，类型为 `BTreeMap<(usize, usize), usize>`，初始化为空表。键的第一项是当前任务编号，第二项是系统调用编号，值为调用次数。

新增 `record_syscall()` 与 `get_syscall_count()`，通过 `TASK_MANAGER.inner.exclusive_access()` 访问计数表。累计时使用 `entry(...).or_insert(0)`，查询不存在的记录时返回 0。借用在函数结束时释放，不跨越任务切换。

计数表依赖内核堆分配，当前启动流程已在首次使用任务管理器之前调用 `heap_alloc::init_heap()`。记录使用任务编号区分，任务切换不会导致不同任务的次数混在一起。

### 3. 在分发前累计

在 `os/src/syscall/mod.rs` 的 `syscall()` 中，先调用 `record_syscall(syscall_id)`，再进入原有 `match` 分发。

这个顺序保证查询 `sys_trace` 自身时包含本次调用，也保证 `yield` 切换之前统计到原任务、`exit` 不返回时仍然能够记录。当前任务暂停后恢复时，原调用也不会因此被重复计数。

## 四、简答题

以下行号对应当前 `os/src/trap/trap.S`。

### 1. 三个 bad 测例的出错行为

使用 RustSBI 0.4.1 和 QEMU 8.2.2 运行三个测例，结果如下：

| 测例 | 行为 | 实际输出与处理 |
| --- | --- | --- |
| `ch2b_bad_address` | 在用户态向地址 `0x0` 写入一个字节 | 打印 `PageFault in application`，其中 `bad addr = 0x0`；终止当前任务并调度下一个任务 |
| `ch2b_bad_instructions` | 在用户态执行 `sret` | 打印 `IllegalInstruction in application`，终止当前任务并继续调度 |
| `ch2b_bad_register` | 在用户态执行读取 `sstatus` 的 CSR 指令 | 打印 `IllegalInstruction in application`，终止当前任务并继续调度 |

这些测试没有导致内核停止，后续 Hello、幂运算、时间和任务切换测试仍然运行。

其中，`sret` 是受特权限制的返回指令，`sstatus` 是 S 模式 CSR，U 模式没有执行或读取它们的权限。地址 0 的访问也被当前机器的访问保护拒绝。

`trap_handler()` 将 `StoreFault` 与 `StorePageFault` 合并为相同的 `PageFault` 输出，不能据此判断实际发生的是哪一种，更不能据此推断 ch3 已启用分页。

### 2. L40：刚进入 __restore 时，sp 是什么？有哪些使用情景？

`sp` 指向当前要恢复的任务内核栈上的 `TrapContext` 起始地址，已经预留了 `34 × 8 = 272` 字节的上下文空间。

两种使用情景分别是：

1. **首次启动任务。** `init_app_cx()` 将人为构造的初始 `TrapContext` 放到该任务内核栈上；`TaskContext::goto_restore()` 将任务上下文的 `ra` 设置为 `__restore`，`sp` 设置为上述上下文地址。`__switch` 恢复它们后，通过 `ret` 进入 `__restore`。
2. **处理 Trap 后恢复用户执行。** `__alltraps` 保存用户现场并调用 `trap_handler()`；处理返回后执行 `__restore`。如果中途发生了任务切换，则等待该任务再次被调度，恢复其内核执行流后再走到这里。

注意，当前 ch3 的 `__restore` 直接使用已经准备好的 `sp`，没有 ch2 中用于首次调用的 `mv sp, a0`。

### 3. L43—L48：特殊处理了哪些寄存器？有什么意义？

这段汇编从上下文读取保存值，通过 `t0`、`t1`、`t2` 暂存，随后写入三个 CSR：

| 寄存器 | 恢复内容 | 作用 |
| --- | --- | --- |
| `sstatus` | 上下文中的状态寄存器值 | `SPP` 决定 `sret` 返回的特权级；初始任务上下文将它设置为 User。`SPIE` 等位也参与返回时的中断状态恢复 |
| `sepc` | 应用开始或继续执行的地址 | `sret` 根据此值恢复 PC；首次启动时为应用入口，处理 `ecall` 后通常为下一条指令地址 |
| `sscratch` | 上下文中的用户栈指针 `x[2]` | 暂存用户栈指针，稍后与内核 `sp` 交换，使用户程序恢复自己的栈 |

`t0`、`t1`、`t2` 自身的应用值会在后续通用寄存器恢复阶段恢复，不会因用作临时变量而丢失。

### 4. L50—L56：为什么跳过 x2 和 x4？

`x2` 是 `sp`。恢复其他寄存器时，需要继续用内核栈上的上下文作为读取基址，如果提前将 `sp` 改成用户栈指针，后面的加载就会访问错误位置。因此先把保存的用户栈指针放入 `sscratch`，在上下文恢复完成后再交换。

`x4` 是 `tp`，即线程指针寄存器。当前教学实现约定应用不使用 `tp`，所以保存和恢复时都跳过。它不是恒为零的寄存器；若以后支持线程局部存储等用途，需要相应保存和恢复。

另外，`x0` 恒为零，不需要保存或恢复。

### 5. L60：交换后 sp 和 sscratch 分别表示什么？

执行前，`addi sp, sp, 34*8` 已回收上下文空间，`sp` 指向内核栈顶，`sscratch` 保存用户栈指针。

执行 `csrrw sp, sscratch, sp` 后：

- `sp` 为用户栈指针，供即将恢复的用户程序使用。
- `sscratch` 为当前任务内核栈顶，供下一次用户 Trap 时切换栈使用。

这条交换指令本身不会改变 CPU 特权级。

### 6. __restore 中哪条指令切换状态？为什么进入用户态？

实际特权级切换发生在 L61 的 `sret`。

执行前，内核已恢复 `sstatus` 和 `sepc`。`sret` 根据 `sstatus.SPP` 设置返回特权级，并从 `sepc` 指向的地址恢复执行。首次应用上下文将 `SPP` 设为 User；应用从 U 模式陷入 S 模式时，硬件也会在 `SPP` 中记录此前来自 U 模式。因此这里返回 U 模式。

`sret` 并非无条件进入 U 模式，目标特权级取决于相应状态设置。

### 7. L13：交换后 sp 和 sscratch 分别表示什么？

进入 `__alltraps` 时，CPU 已处于 S 模式，但硬件没有自动切换软件栈：此时 `sp` 仍为用户栈指针，`sscratch` 保存内核栈顶。

执行 `csrrw sp, sscratch, sp` 后：

- `sp` 改为内核栈顶，供内核保存上下文和执行处理函数。
- `sscratch` 暂存原用户栈指针，随后被读取并写入 `TrapContext.x[2]`。

接下来减小 `sp`，为 Trap 上下文预留空间。

### 8. 从 U 态进入 S 态发生在哪条指令？

对于用户系统调用，触发点是用户库中的 `ecall`。CPU 检测到 U 模式环境调用异常后，在当前固件配置下将其交给 S 模式处理，记录 `sepc`、`scause` 等状态，并跳转到 `stvec` 设置的 `__alltraps` 入口。

因此，进入 `__alltraps` 的第一条指令之前，特权级切换已经由硬件完成；L13 只负责交换栈指针。

对于非法指令、非法访存，则是执行对应操作触发异常后进入 S 模式；对于时钟中断，硬件在接收并处理该中断时进入 S 模式，并不要求应用先执行 `ecall`。从 U 模式进入 S 模式并非只能由某一条固定指令触发。

## 五、测试过程与结果

在 `os/` 目录执行 `cargo check`，编译检查通过。在仓库根目录运行官方检查器：

```bash
PATH="/usr/bin:$PATH" make -C ci-user test CHAPTER=3
```

在添加本报告之前，程序部分的实际输出节选为：

```text
string from task trace test

Test trace OK40231!
Test sleep OK40231!
Test passed40231: 7/7
Report for lab1 needed. Add your report to reports/lab1.pdf or reports/lab1.md
```

`OK` 后面的数字由检查器随机生成，每轮可能不同。`7/7` 表示本轮全部输出检查项通过，不表示运行了 7 个应用。此前命令整体失败的原因是缺少报告，不能将该轮称为完整评测成功。添加本报告后的结果见下方复验记录。

实现过程中修正过两处语法错误：将计数语句放到 `match` 分发之前，以及将计数查询写成与其他分支同级的 `2 => ...`，避免在分支内部重复写分支语法。

测试也观察到 A/B/C 程序主动 `yield` 后交替输出，以及不主动 `yield` 的幂运算程序中途被时钟抢占并在后续恢复。模板最后通过 `panic!("All applications completed!")` 结束运行，应结合检查器结果理解该输出。

### 报告添加后的复验

添加本报告后，使用上述命令重新运行，实际输出节选为：

```text
Test trace OK40298!
Test passed40298: 7/7
Report for lab1 found.
```

命令退出码为 0，表明本轮程序检查和报告文件检查均通过。本地检查器验证了报告文件存在；问答覆盖情况由本文与指导书逐项核对，不能将文件检查等同于报告内容的人工评分。

## 六、荣誉准则与协助说明

依据课程[荣誉准则](https://learningos.cn/rCore-Tutorial-Guide/honorcode.html)，本节如实记录本次已知的交流、参考和协助情况；不将受到协助的工作表述为完全独立完成。

1. **交流与协助对象：**使用 Codex AI 助手学习应用加载、特权级切换、Trap、任务调度及 Rust 借用概念。AI 助手提供了字节读写实现示例、基于 BTreeMap 的按任务计数方案、代码修改指导，并直接修正了两处语法错误、执行了编译与测试。本文也由 AI 助手根据实际源码和日志协助起草。
2. **参考资料：**课程提供的实验代码和测试程序；[第三章练习说明](https://learningos.cn/rCore-Tutorial-Guide/chapter3/5exercise.html)；[特权级切换](https://learningos.cn/rCore-Tutorial-Guide/chapter2/4trap-handling.html)；[任务切换](https://learningos.cn/rCore-Tutorial-Guide/chapter3/2task-switching.html)；本仓库 README 与固件来源记录。代码相关位置补充了实现参考与协助注释。
3. **完成方式：**本次通过自行修改代码、与 AI 助手讨论以及接受直接修正的方式完成练习。实现方案、部分代码与报告写作均受到上述协助；不作“全部代码与文档均独立完成”或“未使用任何外部示例代码”的不实声明。
4. **提交内容说明：**本次功能修改用于实现课程要求的字节读写和调用统计，测试使用官方检查器与官方用户程序，未通过修改测试断言或硬编码随机输出通过检查。本报告记录的是本地测试情况，不代表已推送、获得远程评分或上传成绩。
