# loongarch64迁移

trust_os 是只支持 riscv64 架构的，需要我们完成向 loongarch64 的迁移。本文档只记录迁移过程中和 riscv64 架构不同的部分。

## 内核初始化流程

在启动qemu评测命令后，qemu 执行默认的 bios 后，将跳转到我们设置的内核入口点，即 os/src/arch/loongarch64/qemu/asms/entry.asm 文件中的 \_start 函数，在 \_start 函数中，只进行栈空间初始化

\_start 函数执行后跳转到 init_csr_regs 函数，该函数负责初始化 csr 寄存器，完成设置直接映射窗口、设置中断和异常配置、设置映射翻译模式以及设置页表相关参数，随后跳转到多架构通用的 rust_main 函数，执行一般的初始化。

## 地址空间

龙芯MMU支持两种地址翻译模式（通过设置CSR.CRMD的DA和PG字段）

- 直接地址翻译模式

  - CPU复位后默认模式，此时整个虚拟地址空间可用，任何地址在某位截断后，低位当做物理地址使用

- 映射地址翻译模式

  - 直接映射模式

    > 举例来说，在 PALEN 等于 48 的情况下，通过将 DMW0 配置为 0x11，那么在 PLV0级下，0x9000_0000_0000_0000~0x9000_FFFF_FFFF_FFFF 这段原本在页映射模式下不合法的虚地址空间，将被映射到物理地址空间 0x0~0xFFFF_FFFF_FFFF 上，且存储访问类型是一致可缓存的。

  - 页表映射模式

    > 翻译地址时将优先看其能否按照直接映射模式进行翻译，无法进行后再按照页表映射模式进行翻译。


![image-20250603094644880](./assets/image-20250603094644880.png)

在翻译模式下，内核地址空间如下

- 0x9000_0000_0000_0000 ~ 0x9000_ffff_ffff_ffff：本段为直接映射窗口，内核地址0x9000_xxxx_xxxx_xxxx对应物理地址 0Xxxxx_xxxx_xxxx
- 0xffff_ffff_0000_0000：本段为内核的页表映射，其中包含了 MMIO 映射（这是为了单独将 MMIO 地址设为非缓存）、用户态信号跳板函数（为了单独将该页设为用户可访问）

用户地址空间和 riscv 一致，均为：0 ~ 0x2f_ffff_ffff

## 页表实现

根据 loongarch 基础架构手册，多级页表的结构如下图所示（图中为四级页表，DirX_width和DirX_base需要手动设置相关寄存器）

<img src="./assets/image-20250611214731605.png" alt="image-20250611214731605" style="zoom: 50%;" />

为了和 riscv 架构下尽量统一，loongarch64 的页表同样为三级页表，并且每个页的大小为 4096 字节。页表相关的 csr 寄存器初始化设置如下。

```rust
/// 初始化csr寄存器
#[no_mangle]
pub fn init_csr_regs() {
    ...
    // 设置当前模式寄存器
    // 设置特权级，启用分页
    crmd::set_plv(CpuMode::Ring0);
    crmd::set_da(false); // 启用分页（和set_pg一起发挥作用）
    crmd::set_pg(true);
    crmd::set_datf(MemoryAccessType::CoherentCached); // 直接翻译的取指访存类型
    crmd::set_datm(MemoryAccessType::CoherentCached); // 直接翻译的load/store访存类型
    crmd::set_we(true);

    ...
    
    // 设置页表参数
    tlbidx::set_ps(PAGE_SIZE_BITS);
    stlbps::set_ps(PAGE_SIZE_BITS);
    tlbrehi::set_ps(PAGE_SIZE_BITS);
    pwcl::set_pte_width(8);

    // 页表结构配置
    pwcl::set_ptbase(12);
    pwcl::set_ptwidth(9);
    pwcl::set_dir1_base(12 + 9);
    pwcl::set_dir1_width(9);
    pwcl::set_dir2_base(12 + 9 + 9);
    pwcl::set_dir2_width(9);
    pwch::set_dir3_base(0);
    pwch::set_dir3_width(0);
    pwch::set_dir4_base(0);
    pwch::set_dir4_width(0);
    
    ...
}
```

根据基础手册可知 loongarch64 的基本页表项格式如下（各个标志位的含义可见基础架构手册）。

<img src="./assets/image-20250611215028685.png" alt="image-20250611215028685" style="zoom:50%;" />

我们默认页表映射的物理页是经过缓存的，所以页表项中的 MAT 标志位总是“一致可缓存”。P 标志位（表示是否存在对应的物理页）是由软件管理的，我们将其置为恒1。D 标志位在 tlb 中表示该页是否被修改过，和 riscv 不同，D 标志位是由软件管理的，需要在触发 PageModifyFault 例外时修改。页表项第9个比特位（从0计数）是空闲位，和 riscv 一致，我们将其设为 COW 位，用于写时复制。

由于 loongarch64 和 riscv64 的页表项标志位不完全相同，需要设计二者统一的页表项标志位，以便给页表调用者提供统一的接口。统一的标志位接口表示如下。

```rust
bitflags! {
    // 页表项标志位
    pub struct PTEFlags: usize {
        const VALID = 1 << 0;       // 有效位
        const READABLE = 1 << 1;    // 可读
        const WRITEABLE = 1 << 2;   // 可写
        const EXECUTABLE = 1 << 3;  // 可执行
        const USER = 1 << 4;        // 用户态访问
        const COW = 1 << 9;         // 写时复制
    }
}
```

目录项结构和 riscv 不同，loongarch64 的目录项的值仅为子一级页表项/目录项所在的物理页的页首地址，实现的 find_pte 如下。

```rust
fn find_pte(&self, vpn: VirtPageNum) -> Option<&mut PageTableEntry> {
    let idxs = vpn.indexes();
    let mut ppn = self.root_ppn;
    for (i, idx) in idxs.iter().enumerate() {
        if ppn.0 == 0 {
            return None;
        }
        if i == 2 {
            let pte = &mut ppn.as_array::<PageTableEntry>()[*idx];
            return Some(pte);
        } else {
            let next = ppn.as_array::<usize>()[*idx];
            ppn = PhysAddr::from(next).floor();
        }
    }
    return None;
}
```

设置和切换页表需要关注的寄存器为 pgdl 和 pgdh，由于 tatlin_os 的内核和进程共用页表，所以可以将这两个寄存器视为一个寄存器，存放的是当前页表的根物理地址。当启用某个页表时，将 pgdl 和 pgdh 同时设为该页表的根物理地址，并刷新 tlb 缓存即可。

## 异常处理

在本文中，中断（Interrupt）是指外部事件（如时钟信号）引发的，例外（Exception）是指在指令执行过程中发生的异常情况引发的，我们统一将中断和例外称为 “异常” 。

loongarch64 中发生的异常归类为三种，tlb重填异常、机器错误异常、其他普通异常。

首先介绍 loongarch64 特有的 tlb 重填异常，当访存操作的虚地址在 TLB 中查找没有匹配项时，触发该异常，通知系统软件进行 TLB 重填工作，该异常允许在其他异常处理过程中被触发。而 riscv64 中，tlb 重填是完全由硬件负责的。

tlb重填发生时，会跳转到 tlbrentry 寄存器存放的、单独的处理函数跳转到相应地、单独地处理函数，

```rust
        .globl __tlb_rfill
        .section .text.__rfill
        .balign 4096
__tlb_rfill:
        csrwr $t0, 0x8B
        csrrd $t0, 0x1B
        lddir $t0, $t0, 2
        lddir $t0, $t0, 1
        ldpte $t0, 0
        #取回偶数号页表项
        ldpte $t0, 1
        #取回奇数号页表项
        tlbfill
        csrrd $t0, 0x8B
        #jr $ra
        ertn
```









riscv64 和 loongarch64 都有类似的异常处理流程，但是，二者的异常类型并不完全相同。

为了使得系统更容易维护和扩展，我们实现了两个架构中关于异常的统一处理流程，并在合适地地方调用两个架构分别实现的具体处理函数。

当异常发生时，我们需要将两个架构中具体的异常类型转换为统一的异常类型，定义统一异常类型如下。

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Trap {
    Exception(Exception),
    Interrupt(Interrupt),
    Unknown,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Exception {
    LoadPageFault,
    StorePageFault,
    FetchInstructionPageFault,
    /// 龙芯特有的PageModifyFault，发生时需要内核将这一页的dirty置为1
    PageModifyFault,
    Syscall,
}

/// The interrupt type.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(usize)]
pub enum Interrupt {
    Timer,
}
```





## devices

TODO：

## 其他工作

TODO

时钟、串口输入输出

## 参考

TODO

#### 条件编译中存在的互斥条件

每个项目（比如os和user）的`Cargo.toml`文件，在 `[features]` 选项可能存在 `default`，这是为了让vscode不产生红色波浪线报错，如果编译出错，请检查`default`中的feature与cargo build传入的feature是否冲突（例如，default有`riscv64`而cargo build传入`--feature=loongarch64`，则会报错

kernel 的 debug_level 之间也存在互斥

#### 在用户程序实现了la64的编译

- 在user/dotcargo/config文件中添加：（参考NPUcore-IMPACT）

  ```
      "-Clink-arg=-nostdlib",
      "-Clink-arg=-static",
      "-Cpanic=abort"
  ```

  并且由于要配套loongarch64-unknown-linux-gnu编译工具，在文件中还指定了linker = "loongarch64-linux-gnu-gcc，以支持la64

- 由于指定`-Clink-arg=-nostdlib`，所以在user/arch文件夹下给链接器提供了一些函数

- 修改了system_call的调用处代码

#### la64的gdb

来自：https://github.com/LoongsonLab/oscomp-toolchains-for-oskernel

#### 关于vendor

vendor目录不能手动修改，在os（或者user）下运行cargo vendor会将Cargo.toml中的依赖项复制（或者下载）到vendor目录下，覆盖修改。如果想要定制vendor中的库，请将其放到项目外的目录A下（即os或者user目录之外），并修改os或者user的Cargo.toml中的对应依赖的path为目录A

为了解决lwext4_rust库间接引用的cty库不支持la64的问题，将cty修改后放下lwext4_rust文件夹中，并修改了os/Cargo.toml让cty的引用重定向

在os/vendor下引用库loongArch64（https://github.com/Godones/loongArch64.git），提供了读写寄存器等辅助函数

#### 特权级

la64有四个特权级，PLV0~PLV3。只有PLV0可以使用特权指令（如CSR指令）。所处特权级由CSR.CRMD（当前模式信息寄存器）中的PLV域决定。

由上，设置内核态位于PLV0，用户态位于PLV3

#### 地址空间

这部分参考了龙芯架构参考手册（LoongArch-Vol1-v1.10-CN）第五章，NPUcore的mm.pdf文档

物理地址空间范围为 $0 \sim 2^{PALEN}-1$，PALEN由CPUCFG指令获得，tatlin_os中该值为48

龙芯MMU支持两种地址翻译模式（通过设置CSR.CRMD的DA和PG字段）

- 直接地址翻译模式

  - CPU复位后默认模式，此时整个虚拟地址空间可用，任何地址被低位截断后当做物理地址使用

- 映射地址翻译模式（如下图所示）

  - 直接映射模式

    - 参考手册原文：

      > 举例来说，在 PALEN 等于 48 的情况下，通过将 DMWO 配置为 0x11，那么在 PLV0级下，0x9000_0000_0000_0000~0x9000_FFFF_FFFF_FFFF 这段原本在页映射模式下不合法的虚地址空间，将被映射到物理地址空间 0x0~0xFFFF_FFFF_FFFF 上，且存储访问类型是一致可缓存的。

  - 页表映射模式

![image-20250603094644880](./assets/image-20250603094644880.png)

由图可知，如果访问的虚拟地址的PALEN位是0，则从PGDH寄存器读出页表的ppn到PGD，然后查询；如果是1，则读出PGDL

> 采用页表映射模式时，合法虚拟地址的 [63, VALEN] 位必须与 [VALEN-1] 位相同，否则出发地址错（ADE）例外

#### 存储访存类型

由页表项的MAT域（具体可见架构手册2.1.7节）

| 类型       | 是否进入缓存 | 是否保证先后顺序   | 适用范围                                                   |
| ---------- | ------------ | ------------------ | ---------------------------------------------------------- |
| 一致可缓存 | 是           | 否（允许一定重排） | 代码、堆栈等，一般都用这个                                 |
| 强序非缓存 | 否           | 是（强顺序）       | MMIO、外设寄存器                                           |
| 弱序非缓存 | 否           | 否（允许重排）     | DMA 缓冲区、与设备共享的数据结构（需要内存一定的内存屏障） |

#### 页表

请参考架构手册5.4.5

> CSR.PWCL和 CSR.PWCH用来配置 LDDIR和 LDPTE指令所遍历页表的规格参数信息

查询页表的过程（我们设置了三级页表，图中为四级），图中的DirX_width和DirX_base需要手动设置相关寄存器

![image-20250611214731605](./assets/image-20250611214731605.png)

![image-20250611215028685](./assets/image-20250611215028685.png)

**使用sv39方案**

qemu模拟的la64CPU的 VALEN=48，PALEN=48

开启虚地址缩减模式

> 为了在某些应用场合减少页表级数，LA64架构下还提供了一种虚地址缩减模式。当系统软件将CSR.RVACFG 寄存器中的 RDVA 配置为 1~8中的一个值时,映射地址翻译模式下虚拟地址的有效位将按照(VALEN-RDVA)这么多位来处理。例如，在一台 VALEN=48 的处理器上，当 RDVA 配置为8时，合法地址的[63:40]位需要是第[39]位的符号扩展。

以第38位区别内核和用户地址空间

内核地址空间：0xffff_ffc0_0000_0000 ~ 0xffff_ff....ff

用户地址空间：0x0 ~ 0x0000_003F_FFFF_FFFF

**关于tlb**

>  若未在 TLB 中命中，则需要将页表内容从内存中取出并填入 TLB 中，这一过程通常称为 TLB 重填（TLB Refill）。 TLB 重填可由硬件或软件进行，riscv64，即由硬件完成页表遍历 （Page Table Walker），将所需的页表项填入 TLB 中；而 LoongArch 默认采用软件 TLB 重填，即查找 TLB 发现不命中时，将触发 TLB 重填异常，由异常处理程序进行页表遍历并进行 TLB 填入。

**关于全局标志位(G)**

>  当该位为 1 时，查找时不进行 ASID 是否一致性的检查。当操作系统需要在所有进程间共享同一虚拟地址时，可以设置 TLB 页表项中的 G 位置为 1。

riscv64中并未涉及到这一方面，未来可以关注G位设置

#### MMIO和串口通信：实现控制台输入输出、关机

使用以下命令得到设备树文件（和CPU相关的信息）

```
qemu-system-loongarch64 -M virt -cpu la464 -nographic -kernel kernel-la -machine dumpdtb=ladtb.dtb
dtc -I dtb -O dts ladtb.dtb -o ladtb.dts
```

由设备树可以得到

- 关机：向地址 `0x100e001c` 写入 `0x34`
- 控制台输入输出的串口地址：`0x1fe001e0`，使用标准的 NS16550 UART 控制器

我们启用了0x9000起始的直接映射窗口，所以上面的两个物理地址需要加上0x9000_0000_0000_0000实现访问

注意：

**MMIO 区域会导致物理内存地址空间中的“空洞”（Hole）** ，也就是说：

- 虽然物理地址是连续的，但某些地址范围被外设占用，不能用来存放数据或代码
- 实际可用 RAM 是多个不连续的地址块

在将物理内存交给CMA管理时需要考虑这一部分（不过初赛的内存为128MB，而上面的串口和关机地址都大于128MB）

#### 异常

la64中三类异常处理

- 用户态异常
  -  __alltraps -> trap_handler -> trap_tretunr 
- tlb重填异常
  - __tlb_rfill
- 内核态其他异常



？？？TODO：整合arch模块对外的异常类型

#### 改进

是不是可以不使用直接地址翻译模式，即不使用0x9000开头的地址？如果不使用直接翻译，则需要再启用页表之时，保证启用前后的指令均能被CPU访问到（开启了页表后的下一条指令地址，即PC+4就不合法了）

#### 参考

[1. LoongArch介绍 — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/translations/zh_CN/arch/loongarch/introduction.html)

[多级页表硬件机制 - rCoreloongArch-tutorial](https://godones.github.io/rCoreloongArch/mmu.html)

------

[Fediory/NPUcore-IMPACT: “全国大学生计算机系统能力大赛 - 操作系统设计赛(全国)- OS内核实现赛道龙芯LA2K1000分赛道” 一等奖参赛作品](https://github.com/Fediory/NPUcore-IMPACT/tree/NPUcore-FF)

[LoongsonLab/StarryOS-LoongArch: Port Starry OS to LoongArch64 Platform](https://github.com/LoongsonLab/StarryOS-LoongArch)

[Junkher/xv6-loongarch: OS2022-Proj95](https://github.com/Junkher/xv6-loongarch/tree/main)