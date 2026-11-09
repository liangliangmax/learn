# ARM 平台开机全链路 · 代码级深度解析

> 以树莓派 4B（BCM2711，Cortex-A72，arm64）为主线，从上电到登录界面，
> 每一站执行什么代码、源文件在哪、逐行注释。并与 x86 链路对照。
>
> 源码全部本地可读：
> - U-Boot 2025.10（引导器）：`~/workspace/boot-code-reading/u-boot-2025.10/`
> - Linux 7.3.0 内核 arm64 部分：`~/workspace/kernel-build/src/linux-7.3/arch/arm64/`
> - ARM64 Boot Protocol 文档：内核源码 `Documentation/arch/arm64/booting.rst`
>
> 闭源部分（说清楚）：树莓派 GPU 固件（bootcode.bin/start4.elf）、SoC Boot ROM、
> TrustZone 固件(ATF 的部分二进制由芯片厂提供)。其余全开源。

---

## 目录

- 第 0 站：上电与 SoC Boot ROM（片内 ROM，无代码）
- 第 1 站：树莓派 GPU 固件三段链（闭源但行为透明）
- 第 2 站：U-Boot 的一生（开源，重点站）
- 第 3 站：Linux arm64 内核入口（head.S 精读）
- 第 4 站：start_kernel 之后（与 x86 汇合）
- 第 5 站：systemd（与 x86 相同）
- 附录 A：ARM64 vs x86 开机链路对照总表
- 附录 B：ARM64 Boot Protocol（64 字节 Image 头字段表）
- 附录 C：树莓派 config.txt 引导配置对照
- 附录 D：qemu 模拟实验命令
- 附录 E：阅读路线

---

## 全链一图（树莓派 4B 实际路径）

```
上电(电路) ──→ BCM2711 Boot ROM(片内只读代码,首块程序在SPI EEPROM)
   │
   ▼ ①GPU固件链: bootcode.bin(SPI) → start4.elf(读SD卡FAT32根目录!)
   │        fixup*.dat → config.txt(你的开机参数在这改)
   │        GPU固件加载: kernel8.img(arm64内核) + bcm2711-rpi-4-b.dtb
   │        然后置起CPU(核心0), 跳内核入口
   ▼ ②Linux内核 primary_entry(arch/arm64/kernel/head.S:85)
   │        (树莓派官方镜像: GPU固件直接跳内核, U-Boot是"可插入选项")
   │        [通用ARM板: BootROM→ATF(ARM Trusted Firmware)→U-Boot→内核]
   │
   ▼ ③U-Boot路线(树莓派装了uboot时): SPL→U-Boot proper→bootcmd扫描
   │        → 从SD/extlinux找到Image+dtb → boot_jump_linux()跳内核
   ▼ ④arm64内核: _head(64字节头) → primary_entry → 建初始页表
   │        __primary_switch(开MMU!) → __primary_switched → start_kernel
   ▼ ⑤start_kernel()(与x86同一份代码!) → rest_init → PID1
   ▼ ⑥systemd → 登录
```

**和 x86 最大的三个结构差异**（先记住这个再读代码）：

| 差异 | x86 | ARM/树莓派 |
|---|---|---|
| 第一段代码 | BIOS 固件（CPU 从 0xFFFFFFF0 取指） | **SoC Boot ROM**（芯片内部，地址 0） |
| 引导配置 | GRUB 写死在 MBR→grub.cfg | **GPU 固件读 FAT32 明文文件**（改文件=改引导） |
| 设备描述 | ACPI 表（固件给） | **设备树 DTB**（一个描述硬件的数据文件，传给内核） |

---

## 第 0 站：上电与 SoC Boot ROM

按电源 → PMIC 电源就绪 → SoC 复位：
1. Cortex-A72 核心保持复位，**先唤醒 GPU**（VideoCore VI）——树莓派的启动哲学：GPU 是"主引导 CPU"
2. GPU 从片内 **Boot ROM**（掩膜在硅片里的只读代码）执行：
   - 读 **SPI EEPROM**（树莓派4焊在板上的小存储，bootloader 可升级）
   - EEPROM 里第一阶段 `bootcode.bin` 被加载执行
3. Boot ROM/EEPROM 阶段：初始化 SDRAM（训练内存时序），点亮 SD 卡接口

**无源码**：Boot ROM 是芯片制造时掩膜的；EEPROM 里的 bootloader 是树莓派基金会的闭源固件。

> x86 对照：x86 是 CPU 自己从 0xFFFFFFF0 取第一条指令；
> 树莓派是 **GPU 先当 CPU 用**，把系统带到能读 SD 卡的程度，再叫醒 ARM 核。

---

## 第 1 站：树莓派 GPU 固件三段链（闭源，但行为透明）

```
bootcode.bin(SPI EEPROM) 
   → start4.elf        ← GPU固件主体, 从SD卡第一个FAT32分区根目录读取
   → fixup4.dat        ← 内核/GPU内存划分表(arm_mem/gpu_mem参数的载体)
   → config.txt        ← ★ 你的"BIOS设置页": 明文配置,改了立即生效
```

**这一站没有源码，但有完整文档**（树莓派官方 firmware 文档），行为完全透明：

`config.txt`（你以后裁剪/定制树莓派系统第一个要碰的文件）关键条目：
```
arm_64bit=1              # 加载64位内核(kernel8.img)
kernel=kernel8.img       # 内核文件名(不设就按规则找:8→7→img)
device_tree_address=0x1f00000   # dtb 放的内存地址
initramfs followkernel   # initramfs 跟在内核后面
enable_uart=1            # 打开串口(无屏调试神器,相当于x86的console=ttyS0)
dtparam=audio=on         # 设备树覆盖参数(改硬件描述!)
```

GPU 固件干完的事（相当于 x86 里 BIOS+GRUB 的合体）：
1. 读 `config.txt` + `cmdline.txt`（内核命令行，相当于 grub.cfg 的 linux 行参数）
2. 从 FAT32 分区加载 `kernel8.img` 到内存（**arm64 内核不压缩**——这是与 x86 的一大差别，见下文）
3. 加载 `bcm2711-rpi-4-b.dtb`（设备树）到 `device_tree_address`
4. 把 dtb 指针放进 **x0 寄存器**、内核入口地址放进 PC，唤醒 ARM 核心

> **为什么 arm64 内核 Image 不压缩（不像 x86 的 bzImage）**？
> ARM64 Boot Protocol 的设计哲学：解压由引导器负责（U-Boot 有 gzip 支持）；
> 内核 Image 保持"可执行+自描述"，64 字节头既是数据又是代码（见第 3 站）——
> 没有 x86 那种"解压器住在内核里"的奇技淫巧，分工更干净。

---

## 第 2 站：U-Boot 的一生（开源重点站）

树莓派官方镜像里 GPU 固件直接跳内核；但**通用 ARM 板**和"想要更多控制"的场景用 U-Boot
（你自己交叉编译 ARM 内核做自制系统时，U-Boot 是最标准的引导器）。
文件：`~/workspace/boot-code-reading/u-boot-2025.10/`

U-Boot 自己也分两段（因为 DDR 内存初始化前代码只能在 SRAM 里跑）：

```
SPL(Secondary Program Loader, ~几十KB) → 初始化DDR → 加载U-Boot主体(几百KB)
```

### 2.1 crt0_64.S —— U-Boot 的 _main（进入 C 世界的仪式）

**文件：`u-boot-2025.10/arch/arm/lib/crt0_64.S`**

这个文件开头的注释就是 U-Boot 启动哲学的完整自白（节选）：

```
/*
 * 1. Set up initial C runtime environment ...
 *    This environment only provides a stack and a place to store
 *    the GD ('global data') structure, both located in some readily
 *    available RAM (SRAM, locked cache...). In this context, VARIABLE
 *    global data, initialized or not (BSS), are UNAVAILABLE; only
 *    CONSTANT initialized data are available. GD should be zeroed
 *    before board_init_f() is called.
 *
 * 2. Call board_init_f(). This function prepares the hardware for
 *    execution from system RAM (DRAM, DDR...) As system RAM may not
 *    be available yet, board_init_f() must use the current GD to
 *    store any data which must be passed on to later stages. These
 *    data include the relocation destination, the future stack, and
 *    the future GD location.
 *
 * 4a. For U-Boot proper (not SPL), call relocate_code(). This function
 *    relocates U-Boot from its current location into the relocation
 *    destination computed by board_init_f().
 *
 * 7. Branch to board_init_r().
 */
```

**解读**——U-Boot 的七步舞，本质是"在内存荒漠里逐步搭绿洲"：
1. 只有 SRAM/锁定的 cache 当"内存"（几 KB 级），全局变量都放不了 → 先攒一个 `GD`（global data）结构存家当
2. `board_init_f`：初始化 DDR（这段是每块板最硬的部分，时序参数来自设备树）
3-4. **把自己搬到 DDR 高地址**（relocate：闪存/SRAM 里挤着住 → 搬进大房子）
5-7. 在 DDR 里重建栈/全局变量，进 `board_init_r` 永驻

```asm
ENTRY(_main)
#if CONFIG_IS_ENABLED(HAVE_INIT_STACK)
        ldr     x0, =CONFIG_VAL(STACK)         // 栈先落在 SRAM 固定地址
#else
        adrp    x0, __bss_start                // 或 BSS 起点相对寻址(位置无关)
        add     x0, x0, #:lo12:__bss_start
#endif
        ...
        bl      board_init_f                    // ← 第一棒: 初始化 DDR, 算搬家地址
```

### 2.2 board_r.c —— init_sequence_r 接力表（U-Boot 的"总装配流水线"）

**文件：`u-boot-2025.10/common/board_r.c`**

和内核 `start_kernel()` 同一个设计思想——一张初始化函数表按序执行：

```c
static init_fnc_t init_sequence_r[] = {
        initr_trace,              /* 跟踪初始化 */
        initr_reloc,              /* 告诉系统"我已搬家完成" */
        initr_caches,             /* 打开 I/D cache */
        initr_malloc,             /* 堆管理器上线(从此可用 malloc!) */
        initr_dm,                 /* 设备模型(Driver Model)——U-Boot 自己的驱动框架 */
        initr_dm_devices,         /* 枚举设备( mmc/usb/eth ... ) */
        initr_mmc,                /* ← SD 卡驱动在这站起来! */
        initr_env,                /* 环境变量(bootcmd 就存这) */
        initr_console,            /* 串口控制台(U-Boot shell 的输出通道) */
        ...
        run_main_loop,            /* ← 进 U-Boot 命令行/自动启动 */
};
```

到 `run_main_loop` 时，屏幕上出现那行著名提示（树莓派串口里见过）：
```
U-Boot 2025.10 (...)
Hit any key to stop autoboot: 2  ← 2秒内按键进U-Boot shell
```

### 2.3 bootcmd —— U-Boot 自动启动内核（相当于 GRUB 的 grub.cfg）

不按键则执行环境变量 `bootcmd`（标准发行版通常走 `extlinux`/`bootefi`）：

```
# U-Boot shell 里查看(你的系统也能看到):
=> printenv bootcmd
bootcmd=... load mmc 0:1 ${kernel_addr_r} kernel8.img
          load mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb
          booti ${kernel_addr_r} - ${fdt_addr_r}    ← booti=arm64专用"启动Image"命令
```

`booti` → `cmd/bootm.c` → **`arch/arm/lib/bootm.c: do_bootm_linux()`** → 最终跳转：

### 2.4 boot_jump_linux() —— U-Boot 的最后一跳（原文+注释）

**文件：`u-boot-2025.10/arch/arm/lib/bootm.c` 第 286 行**

```c
static void boot_jump_linux(struct bootm_headers *images, int flag)
{
#ifdef CONFIG_ARM64
        /* 函数指针: 签名就是 ARM64 Boot Protocol 的寄存器约定!
           void kernel_entry(fdt_addr, res0, res1, res2)
           —— 第一个参数 = 设备树地址(它将成为内核的 x0) */
        void (*kernel_entry)(void *fdt_addr, void *res0, void *res1,
                             void *res2);
        int fake = (flag & BOOTM_STATE_OS_FAKE_GO);

        kernel_entry = (void (*)(void *fdt_addr, void *res0, void *res1,
                                void *res2))images->ep;   // ep = 内核入口(Image头算出的地址)

        debug("## Transferring control to Linux (at address %lx)...\n",
              (ulong) kernel_entry);
        bootstage_mark(BOOTSTAGE_ID_RUN_OS);

        announce_and_cleanup(fake);    // 打印"Starting kernel..." + 关闭U-Boot自己的驱动/串口
                                       // (内核期望一个"干净"的机器, U-Boot全身而退)

        if (!fake) {
#ifdef CONFIG_ARMV8_PSCI
                armv8_setup_psci();    // 配置PSCI(CPU电源管理接口,内核 SMP 启动要用)
#endif
                do_nonsec_virt_switch();  // ★ 安全世界切换: EL3/安全态 → EL2/非安全态
                                          //   ARM特有: 异常等级(exception level)机制

                update_os_arch_secondary_cores(images->os.arch);

                /* 真正的跳转:
                   x0 = fdt_addr(设备树)
                   PC = kernel_entry
                   —— U-Boot 使命终结 */
                kernel_entry(images->ft_addr, 0, 0, 0);
        }
#endif
...
}
```

**U-Boot 一生总结**：从 SRAM 里 30KB 的 SPL 起家 → 初始化 DDR → 搬家 →
驱动框架起来认 SD 卡/网卡 → 扫描文件系统找到 Image+dtb → 按 Boot Protocol 摆好内存
→ 退出时清理自己 → 把 dtb 地址放 x0 跳内核。比 GRUB 干的活多得多
（GRUB 站在 BIOS 肩膀上，U-Boot 从"裸芯片"开始）。

> **通用 ARM 服务器路线补充**：实际服务器/手机的链条比树莓派多一层——
> Boot ROM → **ATF（ARM Trusted Firmware，开源！github.com/ARM-software/arm-trusted-firmware）**
> → U-Boot → 内核。ATF 管安全世界(EL3)和 TrustZone，是 ARM 的"BIOS 安全层"。
> 树莓派简化掉了 ATF，直接 U-Boot/固件→内核。

---

## 第 3 站：Linux arm64 内核入口（head.S 精读）

文件：`~/workspace/kernel-build/src/linux-7.3/arch/arm64/kernel/head.S`

### 3.1 _head —— 64 字节 Image 头（数据即代码！）

**head.S 第 59 行起（原文+注释）**：

```asm
        __HEAD
        /*
         * DO NOT MODIFY. Image header expected by Linux boot-loaders.
         */
        efi_signature_nop       // 【8字节特殊NOP】让这个文件看起来是PE/COFF格式!
                                // ——"魔法": UEFI固件直接把这文件当.efi执行(EFI Stub),
                                //    先走efi_stub_entry做EFI初始化,完成后跳回正常流程
        b   primary_entry       // 【可执行指令!】普通引导器(U-Boot/GPU固件)直接跳到这,
                                //    一条b指令进入 primary_entry
        .quad   0               // text_offset: 内核在RAM中的装载偏移(0=按协议就近放)
        le64sym _kernel_size_le // 内核有效大小(引导器据此预留内存)
        le64sym _kernel_flags_le// 标志位(大小端/页大小/物理基址偏好)
        .quad   0               // reserved ×3
        .quad   0
        .quad   0
        .ascii  ARM64_IMAGE_MAGIC  // ★ 魔数 0x644d5241 = "ARM\x64" (U-Boot校验的就是它)
        .long   .Lpe_header_offset // PE头偏移(EFI路线用)

        __EFI_PE_HEADER             // PE/COFF 结构(EFI世界认的部分)
```

**这是全链条最优雅的设计**——arm64 内核 Image 的前 64 字节**同时是三种东西**：
1. **数据表**（text_offset/size/flags——U-Boot 读它布局内存）
2. **代码**（`b primary_entry`——直接可执行）
3. **PE/COFF 可执行文件头**（UEFI 固件把它当 .efi 直接加载执行！）

一个文件三种身份 = 不需要单独的引导扇区/解压器——对比 x86 bzImage 的三段结构
（setup区+解压器+压缩球），arm64 用 64 字节头+不压缩 Image 达成了同样的自举。
协议文档 `Documentation/arch/arm64/booting.rst`（前面已全文展示 64 字节字段表）。

### 3.2 primary_entry —— CPU 进入内核的第一批指令

**head.S 第 85 行（原文+注释）**：

```asm
/*
 * 寄存器约定(注释原文):
 *   x19  primary_entry() .. start_kernel()   启动时MMU是否已开
 *   x20  primary_entry() .. __primary_switch()  CPU启动模式
 *   x21  primary_entry() .. start_kernel()   ★ FDT(设备树)指针,引导器放在x0传进来
 */
SYM_CODE_START(primary_entry)
        bl  record_mmu_state        // 记录"进来时MMU是开是关"(ARM允许带MMU进内核!)

        bl  preserve_boot_args      // ★ 把 x0..x3(引导器传的4个参数)存进 boot_args 数组
                                    //   x0=FDT指针 → 存为 x21, 后面 start_kernel 要用!

        adrp    x1, early_init_stack
        mov     sp, x1              // 设初期栈
        mov     x29, xzr            // 清帧指针
        adrp    x0, __pi_init_idmap_pg_dir
        mov     x1, xzr
        bl  __pi_create_init_idmap  // ★ 手工建"恒等映射页表"(物理地址=虚拟地址)
                                    //   ——开MMU前的必要准备(ARM开MMU必须先有页表,
                                    //      而建页表代码本身又要在MMU关时能跑:自举)

        /* MMU 关着建的页表, 先清缓存保证一致性 */
        dmb     sy
        ...
0:      adrp    x0, __idmap_text_start
        ...
        bl  __cpu_setup             // 初始化处理器(SCTLR等系统寄存器)
        b   __primary_switch        // → 切换!
```

**preserve_boot_args 原文**（head.S:170，引导参数交接的现场）：

```asm
SYM_CODE_START_LOCAL(preserve_boot_args)
        mov     x21, x0             // x21 = FDT(设备树)——x86是ESI传参数块,ARM是x0传DTB

        adr_l   x0, boot_args       // 把 x0..x3 四个寄存器完整存进 boot_args 数组
        stp     x21, x1, [x0]       // (调试时能打印"引导器到底传了什么")
        stp     x2, x3, [x0, #16]

        dmb     sy                  // 屏障(内存序)
        ...
        ret
SYM_CODE_END(preserve_boot_args)
```

### 3.3 __primary_switch —— 点火开 MMU

**head.S 第 508 行（原文+注释）**：

```asm
SYM_FUNC_START_LOCAL(__primary_switch)
        adrp    x1, reserved_pg_dir
        adrp    x2, __pi_init_idmap_pg_dir
        bl  __enable_mmu             // ★★ 点火! SCTLR_EL1.M=1, MMU正式开启
                                     //    此后 CPU 看到的是虚拟地址世界
                                     //    (x86对应物: head_64.S里 movl %eax,%cr0 置PE位)

        adrp    x1, early_init_stack
        mov     sp, x1
        mov     x29, xzr
        mov     x0, x20              // 传启动模式
        mov     x1, x21              // 传FDT指针
        bl  __pi_early_map_kernel    // 把内核映射到最终的虚拟地址位置(重定位/KASLR)

        ldr     x8, =__primary_switched
        adrp    x0, KERNEL_START     // 内核物理基址
        br  x8                       // → __primary_switched
SYM_FUNC_END(__primary_switch)
```

### 3.4 __primary_switched —— 通往 start_kernel 的最后一段

**head.S 第 220 行（原文+注释）**：

```asm
SYM_FUNC_START_LOCAL(__primary_switched)
        adr_l   x4, init_task        // 0号进程(和x86的init_task同一个!)
        init_cpu_task x4, x5, x6     // 设当前任务栈(switch到正式内核栈)

        adr_l   x8, vectors
        msr     vbar_el1, x8         // 装异常向量表(ARM版的IDT)
        isb

        stp     x29, x30, [sp, #-16]!
        mov     x29, sp

        str_l   x21, __fdt_pointer, x5   // ★ 全局保存FDT指针——之后所有代码
                                          //   靠它找到设备树(硬件说明书)

        adrp    x4, _text
        sub     x4, x4, x0
        str_l   x4, kimage_voffset, x5   // 记下 虚拟地址-物理地址 偏移(ARM64的
                                          //   linear mapping需要)

        bl  set_cpu_boot_mode_flag
#if defined(CONFIG_KASAN_...)
        bl  kasan_early_init
#endif
        bl  finalise_el2             // ★ ARM特有: 尽量留在EL2(Virtualization Host
                                     //   Extension)——内核当hypervisor跑,虚拟化免切换
        ldp     x29, x30, [sp], #16
        bl  start_kernel             // ★★ 进入 start_kernel!从此与x86合流
        ASM_BUG()
SYM_FUNC_END(__primary_switched)
```

---

## 第 4 站：start_kernel 之后（与 x86 汇合）

`start_kernel()` 之后的所有代码**与 x86 完全共享**（同一个函数！`init/main.c:1002`）：

```
共用: start_kernel → sched_init/mm_init/... → rest_init → kernel_init → systemd
   │
   └─ 架构差异只藏在三个"钩子"里:
      • setup_arch()     x86版读ACPI/EBDA, arm64版读设备树(unflatten_device_tree
                         把DTB解析成内核数据结构)
      • 硬件驱动         设备树里每个节点被 platform 总线匹配到驱动
      • 中断/定时器      GIC中断控制器(ARM) vs APIC(x86)——都藏在arch/下
```

**这是 Linux 内核工程的精髓**：900 万行代码里 `arch/` 只占不到 10%，
开机这种最底层的东西做到了"95% 共享 + 5% 插拔"。你今天 x86 之旅读过的
`init/main.c`、`start_kernel`、`rest_init`，在树莓派上**逐字节同样执行**——
这就是为什么先在 x86 上学内核完全不亏。

---

## 第 5 站：systemd（与 x86 相同）

PID 1 = systemd，同样拉起 getty 登录。唯一区别：树莓派官方镜像默认
`systemd-boot` 之外还可能有 UART console（串口登录，无屏服务器场景）。

---

## 附录 A：ARM64 vs x86 全方位对比（开机链路 + 体系结构）

### A.1 开机链路对比（12 站逐环节）

| 环节 | x86 | ARM64（树莓派/通用板） |
|---|---|---|
| 复位后第一条指令 | CPU 跳 0xFFFFFFF0（BIOS 闪存） | **GPU/Boot ROM** 执行（片内 ROM） |
| 固件初始化 | BIOS POST（SeaBIOS 可读） | GPU 固件链 / Boot ROM + ATF（闭源+ATF开源） |
| 内存初始化 | BIOS 已做好（GRUB 查 E820） | **引导器自己做**（U-Boot board_init_f，最难的部分） |
| 引导器 | GRUB（站在 BIOS 肩上） | U-Boot（从裸芯片开始，含 SPL 两段式） |
| 引导配置 | grub.cfg（GRUB 自有格式） | config.txt + cmdline.txt（FAT 明文）/ extlinux.conf |
| 设备描述 | ACPI 表 | **设备树 DTB**（独立数据文件，可读可改） |
| 内核形态 | bzImage：setup+解压器+压缩球 三段 | **Image 不压缩**：64 字节头即数据即代码 |
| 内核↔引导协议 | Linux Boot Protocol（HdrS 魔数） | **ARM64 Boot Protocol**（"ARM\x64" 魔数 + x0=dtb） |
| 内核入口跳转参数 | ESI=参数块地址 | **x0=设备树指针** |
| 开 MMU/保护模式 | head_64.S 置 CR0.PE | __primary_switch 开 SCTLR.M |
| 0号进程/1号进程 | init_task / kernel_init | **同名同函数**（arch 无关代码） |
| start_kernel 之后 | — | **100% 相同** |

### A.2 指令集与寄存器对比（读代码时速查）

| 项 | x86 | ARM64 |
|---|---|---|
| 通用寄存器 | 16 个（RAX/RBX/RCX/RDX/RSI/RDI/R8-R15） | 31 个（X0-X30，X31=SPZR） |
| 栈指针 | RSP 专用 | SP（X31 的别名用法） |
| 函数传参 | RDI/RSI/RDX/RCX/R8/R9（System V ABI） | **X0-X7**（所以"dtb 在 x0"是约定第一参数） |
| 返回值 | RAX | X0 |
| 链接寄存器 | —（返回地址压栈） | **LR=X30**（返回地址存寄存器, bl 指令自动填） |
| 取指针技巧 | `lea rax, [rip+X]`（RIP 相对寻址） | `adrp x0, X` + `add x0, x0, #:lo12:X`（4KB页对齐两步走） |
| 16位实模式 | 有（BIOS 时代遗产，boot.S 用的） | **无**（ARM 从第一天就是干净的 32/64 位） |
| 指令定长 | 否（1-15 字节变长） | **是**（一律 4 字节——解码简单,反汇编好读） |

### A.3 开机汇编代码风格对比（同款任务两写法）

**x86（grub boot.S，16 位实模式）：**
```asm
        .code16                    // 还要声明"这是16位代码"
        cli
        xorw %ax, %ax              // AT&T 语法: 寄存器带%前缀, 操作数方向 反
        movw %ax, %ds              // 16位指令带w后缀
        ljmp $0, $real_start       // 段:偏移 远跳转(实模式特有)
```

**ARM64（内核 head.S）：**
```asm
        bl  record_mmu_state       // 一条指令=调用+存返回地址到LR, 干净
        adrp x1, early_init_stack  // 取符号地址(页对齐), ARM64标准姿势
        mov sp, x1                 // 无前缀无后缀, 操作数方向 直观
        b   __primary_switch       // 无条件跳(b) vs 带_LR的调用(bl)
```

### A.4 内存与安全机制对比

| 项 | x86-64 | ARM64 |
|---|---|---|
| 分页模式 | 4级/5级页表（CR4.LA57） | 3级/4级（页大小可选 **4K/16K/64K**，Image 头 flags 申明） |
| 特权级 | Ring 0-3（实际用 0 和 3） | **EL0-EL3**（异常等级:EL0用户/EL1内核/EL2虚拟化/EL3安全监控） |
| 虚拟化 | VT-x（EPT 二级地址翻译） | EL2（内核可直接住 EL2 = VHE，finalise_el2 的选择） |
| 可信执行 | TPM 外挂模块 | **TrustZone**（硬件隔离安全/非安全世界，ATF 把守 EL3） |
| 入口一致性 | 引导器只需管好自己跳走 | 同左+ 可选经 ATF 安全世界切换（do_nonsec_virt_switch） |

### A.5 一表流：两个世界的哲学差异

| 哲学维度 | x86 | ARM |
|---|---|---|
| 血统 | 40 年向后兼容（8086→…→x86-64），每一代背着所有祖先的包袱 | 定期推倒重来（ARMv7→v8 丢弃 26位模式等），包袱少 |
| 引导思想 | "约定最小化"（512字节+魔数,能力逐跳升级） | "数据即代码"（64字节头三重身份,文件自描述） |
| 硬件描述 | ACPI（固件生成,启动时才可见） | **设备树**（独立源文件 dts,编译期可见可 diff 可改） |
| 生态风格 | 一家厂商(Intel/AMD)定标准 | IP 授权模式(ARM定架构,高通/苹果/博通各自实现) |
| 读码体验 | 汇编变长+历史分支多,烧脑 | 定长4字节+LR寄存器,**新手汇编友好得多** |

> **学内核建议**（顺带）：ARM64 汇编比 x86 汇编好读得多——
> 指令定长、传参即 X0-X7、没有 16 位实模式的历史包袱。
> 先读 ARM64 的 head.S 建立"开机汇编"直觉，再回头看 x86 的 boot.S
> 会觉得"原来那些别扭都是历史包袱"，而不是"汇编本来就难"。


## 附录 B：ARM64 Boot Protocol 字段表（Image 头 64 字节）

| 偏移 | 字段 | 谁读 | 说明 |
|---|---|---|---|
| 0 | code0（NOP+`b primary_entry`） | 全部 | 可执行；UEFI 当 PE 头认 |
| 8 | code1 | — | 保留 |
| 16 | text_offset | 引导器 | 内核期望的装载偏移 |
| 24 | image_size | 引导器 | 预留内存大小 |
| 32 | flags | 引导器 | 大小端/页大小(4K/16K/64K)/物理基址策略 |
| 40-56 | res2/3/4 | — | 保留 0 |
| 60 | magic 0x644d5241 | 引导器 | **"ARM\x64" 校验**（U-Boot 的暗号） |
| 63 | res5=PE offset | UEFI | EFI Stub 入口 |

引导器必须做（booting.rst 原文）：初始化 RAM → 放好 DTB（8字节对齐,≤2MB）→ 解压（若压缩）→ x0=dtb, x1..x3=0, PC=Image 入口。

## 附录 C：树莓派 config.txt 对照 grub.cfg

| 用途 | x86 (grub.cfg) | 树莓派 (config.txt+cmdline.txt) |
|---|---|---|
| 换内核 | `linux /boot/vmlinuz-xxx` | `kernel=kernel8.img`（或直接改文件名） |
| 内核参数 | 同一行 root=... | cmdline.txt（整文件就是一行参数） |
| 指定根分区 | `root=UUID=...` | `root=/dev/mmcblk0p2`（cmdline.txt 里） |
| 串口控制台 | `console=ttyS0` | 同款写法 + config.txt `enable_uart=1` |
| 传设备树 | （x86 无 DTB，用 ACPI） | `device_tree_address=` / `dtoverlay=` |

## 附录 D：qemu 模拟 ARM 实验（不买板子也能走这条链）

```bash
# 模拟一台通用 ARM64 虚机(qemu 的 virt 机型), U-Boot→内核:
qemu-system-aarch64 -M virt -cpu cortex-a72 -m 1G -nographic \
  -kernel ~/workspace/kernel-build/src/linux-7.3/arch/arm64/boot/Image \
  -bios u-boot.bin \
  -append "console=ttyAMA0"
# 屏幕输出对应: "U-Boot 2025.10"→board_r.c / "Starting kernel..."→bootm.c /
#              内核滚日志→head.S→start_kernel

# 亲手编一版 arm64 内核(源码已在手,只是加个交叉工具链):
#   sudo apt install gcc-aarch64-linux-gnu
#   make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
#   make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j8
#   产物: arch/arm64/boot/Image  ← 就能进上面 qemu 跑
```

## 附录 E：阅读路线（半天量）

```
1. 内核 Documentation/arch/arm64/booting.rst     15分钟  协议全文(不长,官方文档典范)
2. 内核 arch/arm64/kernel/head.S: 59行起         20分钟  64字节头三重身份
3. 内核 head.S: 85,170,508,220 行               45分钟  primary_entry→开MMU→start_kernel
4. u-boot-2025.10/arch/arm/lib/crt0_64.S        20分钟  注释就是启动哲学
5. u-boot common/board_r.c init_sequence_r       15分钟  接力表(对照start_kernel)
6. u-boot arch/arm/lib/bootm.c: 286行           20分钟  最后一跳(x0=dtb)
7. 对照读: bootm.c(引导器侧) vs head.S(内核侧)    ——和x86那对linux.c/header.S同款玩法
```

---

*生成：2026-11-09 · 代码取自本地真实源码（U-Boot 2025.10 / Linux 7.3.0 arm64）*
*配套 x86 版：`x86开机全链路代码详解.md`（建议两篇并排读，对照表在附录 A）*
