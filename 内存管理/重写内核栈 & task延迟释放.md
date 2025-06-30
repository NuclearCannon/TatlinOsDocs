# TatlinOS中对内核栈的设计

在TrustOS中，每个线程的内核栈虚拟地址区间是和线程id（或者说任务id，即`tid`）线性相关的，我们认为这对空间浪费太大，不方便管理，为此在TatlinOS中我们重新设计了内核栈。

TatlinOS中，内核控制流可能从一处通过`__switch`函数切换到另一处（以实现wait, yield等系统调用功能），以切换到另一个用户线程，这个过程中必须使用不同的栈才能确保前一处控制流中的函数可以正常返回。因此，每个线程的内核栈必须不同。

但是，如果为了这种不同为每个可能的`tid`都分配一段虚拟地址空间作为内核栈，就太浪费且危险了。我们的方法是，将内核栈视为`task`（线程/任务）的财产，随着`task`的构造和析构而动态地分配。

但是，按照TrustOS中原有的涉及，**当一个task exit时，删去这个task的过程所使用的栈仍然是这个task的内核栈**，如果仅仅是将内核栈视为`task`的财产，则会出现删去task后无栈可用的问题。对此，可能的解决方案包括临时栈、延迟释放等。

对此，我们的解决方案是，延迟释放task对象，这样连带地就实现了内核栈的延迟释放。

在TrustOS的`exit_current_and_run_next`函数的末尾，有这样一段代码：

```rust
    drop(task);
    let mut _unused = TaskContext::zero_init();
    schedule(&mut _unused as *mut _);
}
```

它的作用是，放弃当前控制流，转到`run_tasks()`中的控制流（idle）。

为了实现task的延迟释放，我们可用构造一个全局的`tid->task`映射，并在转到`run_tasks()`中的控制流之后再从映射中删除它。为了告诉之后的控制流要释放哪个task，可以构造一个新的函数：



```rust
pub fn exit_current_and_run_next(exit_code: i32) {
    ... // 上面大部分都一样 
    let tid = task.tid();
    drop(memory_set);
    drop(process);
    drop(task);	// 并不会把task从tid->task映射中删除
    // 启用内核页表，避免task的页表释放后控制流使用不存在的页表
    activate_kernel_space();
    // 将tid传给IDLE控制流，它会负责释放这个线程
    abandon(tid);
}

```

`abandon`是一个新设计的函数。它类似`schedule`，会转到该处理器的idle控制流，但是，它会使得转到目的位置之后a0寄存器等于tid。

```asm
.altmacro
.macro SAVE_SN n
    sd s\n, (\n+2)*8(a0)
.endm
.macro LOAD_SN n
    ld s\n, (\n+2)*8(a1)
.endm
    .section .text
    .globl __switch
__switch:
    # save kernel stack of current task
    sd sp, 8(a0)
    # save ra & s0~s11 of current execution
    sd ra, 0(a0)
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr
    # restore ra & s0~s11 of next execution
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # restore kernel stack of next task
    ld sp, 8(a1)
    # 将a0设置为0以告诉调用者：这是从switch来的，不需要free stack
    li a0, 0
    ret

.section .text
.globl __abandon
__abandon:
    # 该函数的作用是
    # 将a1的状态写入到CPU上。CPU原先的状态将被丢弃
    # 同时，不改变a0的值
    # 第一个参数会作为返回值被返回回去
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    ld sp, 8(a1)
    ret

```



```rust
use super::TaskContext;
use crate::task::tid_to_task;

extern "C" {
    /// 该汇编函数的两个参数a0, a1是两个struct TaskContext*，
    /// 该函数是将当前CPU上正在运行的控制流上下文
    /// （此处上下文指的是栈、pc（对pc的保存实际上通过保存ra实现），s系列寄存器）
    /// 保存到a0指向的位置，并从a1处获取新的上下文并运行之。
    /// 因此，该函数不会正常地返回到调用者手中
    /// 如果return 0，说明next_task_cx_ptr是从另一个switch中返回的
    /// 如果return >0，则return的是tid，需要释放tid指向的内核栈
    fn __switch(
        current_task_cx_ptr: *mut TaskContext,
        next_task_cx_ptr: *const TaskContext,
    ) -> usize;

    pub fn __abandon(tid: usize, next_task_cx_ptr: *const TaskContext) -> usize;
}

/// 对汇编函数__switch的包装
/// 会在__switch之后检查返回值，如果不为0则释放那个页
pub fn switch(current_task_cx_ptr: *mut TaskContext, next_task_cx_ptr: *const TaskContext) {
    let tid = unsafe { __switch(current_task_cx_ptr, next_task_cx_ptr) };
    if tid != 0 {
        tid_to_task::remove(tid);
    }
}

// 在另一处：
pub fn abandon(tid: usize) {
    let processor = get_proc_by_hartid(hart_id());

    let idle_task_cx_ptr = processor.get_idle_task_cx_ptr();
    unsafe {
        __abandon(tid, idle_task_cx_ptr);
    }
}

```

这样目的位置就可以根据a0寄存器的值来判断是从switch转来的还是从abandon转来的，如果是从abandon转来的，则需要释放task。

