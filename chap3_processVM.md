# Chapter 3

* 允許執行不同系統(OS, ISA)的程式, ABI層VM

* e.g. IA-32 EL/Windows 讓 Itanium 64/Windows 執行 x86-32/Windows

* Runtime 程式 包覆 guest process


## 3.1 Virtual Machine Implementation

* Loader
 - 配置記憶體，載入 guest memory image
 - 該記憶體內容用來 interprete & BT
 - 載入執行 runtime code

* Initialization
 - 配置 code cache、初始化 tables
 - 向 host OS 註冊 interupt handler

* Emulation Engine
 - 執行 interpretation / BT (回到第二章內容)

* Code Cache Manager
 - 在 code cache 有限的情況下，做 memory 回收

* Profile DB
 - 動態 guest program 資訊，用來optimize
 
* OS Call Emulator
 - 攔截 syscall，模擬syscall 或 往下傳

* Exception Emulator
 - 來自 OS 的 Interrupt
 - 傳給 Host OS 的 Guest Program trap

* Exception side tables
 - 支撐 Exception Emulator 讓模擬準確


## 3.2 Compatibility

* Compatible 定義：在 Host 平台上模擬的 Guest Program 和在 native 平台上 Guest Program 的行為差異
* 理論上應該完全兼容，不過實際情況顯示並非需要完全兼容


### 3.2.1 Levels of Compatibility

* intrinsic compatibility = transparent compatibility
 - 無論 guest code 長得如何，都能與在 native 平台上行為一樣
 - VM上多了 runtime，一些情況難實現 icomp，比如：
   checksum memory、
   guest process佔用完virtual memory(排擠到runtime)、
   高準確度的浮點運算
  
* extrinsic compatibility
 - 官方規範說哪些 guest code、(使用)library、toolchains 支援兼容


### 3.2.2 A Compatibility Framework

* 很難嚴謹證明 compatibility
  * 在框架上討論是否滿足 compatibility 需求
* 框架分為
  1. State Mapping: S -> S'
  2. Op transform: e(S) -> e'(S')

````
Guest
          User             gOS              User
--> {Si} ---e(Si)---> {Sj} ---e(Sj)---> {Sk} -->
    |                 |                 |
    V(Si)             V(Sj)             V(Sk)
    |                 |                 |
--> {Si'} -e'(Si')--> {Sj'} -e'(Sj')--> {Sk'} ->
          Emulate          Runtime, hOS     Emulate
Host
````
  
* State 可分為 
  * User Managed State (User lvl op)
  * OS Managed State (I/O op, privilledge op)

> Much effort on maintaining precise architecture state (out-of-order ins)


#### State Mapping

* 對於 Host 和 Guest 不一樣的機器 (Guest RegN > Host RegN)
 * Map 到 Host 的 MM上

#### Operations

* 從 trap 到 OS/interrupt (vice versa) 時需要 state transfer
 * Si, Sj, Sk 時


#### Sufficient Compatibility Conditions

* 從 UMS 轉 OMS:
 * UMS 不用 ins granularity，而是在 visible point (bb/trap etc) 時才 expose 到外面
 * 增加 translated code 的彈性(做進一步的opt)

* 從 OMS 轉 UMS: 
 * 完整的 Mapping (SPC-TPC etc) 資訊，讓 Runtime/OS 可以直接修改 UMS


#### Discussion

* Compatibility Framework: Formal conditions to discuss compatibility issues

* **Instrinsic** compatible is hard:
 * State mapping: guest virtual mem will always smaller than it runs on native
 * Ctrl transfer mapping: some trap in interrupt emu will be removed (extrinsic then)
 * User level ins compatible: FP ALU accuracy
 * Guest OS != Host OS feature: May avoid it


### 3.2.3 Implementation Dependences

* Architecture and Implementation (aka MicroArch)
 * Difference betw. MicroArch makes functionality different
 * e.g. caches: some will not automatic modified cache contents (passive cache flush needed)

* Logical SMC: 保證執行最新，以下會讓Logical失效
 * 程式用 cache的操作 和 SMC 來判斷底下的 MicroArch 的類型
 * 有些ISA需要flush cache才能執行最新的 SMC
  
> Emulate OS will always trigger self-mod code (when start new/halt prog)


##### Trap Compatibility (except page fault)

* OS dependent
* optimization will remove dead code (trapped code)

````
...
r1 = r2+r3； this will trap
r1 = r4+r5 
...

optimized
...
r1 = r4+r5
...
````

#### Register State Compatible

* Code reorder will reorder the trapped and non-trap ins, make the eliminate registers presice state

````
R1 <- R2 + R3
R9 <- R1 + R5 -- 1
R6 <- R1 * R7 -- 2
R3 <- R6 + 1

1 and 2 re-schedule, what if 2 will invoke trap?
````

#### Memory state compatible

* as reg state compatible, reorder memory protection fault ins with non-fault ins will make memory state not precise


#### Memory ordering compatible

* prod/coms problems: prod update memA, memB; coms read memB, memA; reorder coms read order make memory order non-compat


#### Undefined cases

* ISA
  * register 0 always value 0
* behavioud
  * short-circuit condition


## 3.3 State Mapping

* aka Resources Mapping (Register + Memory)
* Process VM refer to Logical Memory not real mem
* Guest registers mapped into host registers and memory (runtime data)

### 3.3.1 Register Mapping

* Guest reg N < Host reg N: MAY possible to map all to host regs
  * Runtime may use most/all registers for 
   Intepreting, Dynamic Opt, LW/SW Context Block Register

### 3.3.2 Memory Address Space Mapping

* Runtime Space + Guest Space <= Host*(Logical)* Space
  * Guest address 0 <=> Host address 0 (Let hw do the work)

* Guest Space > Host Space 
  * Software Translation (LW/SW overhead)

#### Runtime software-supported Translation Tables

* similiar to Page table

* slow (8 ins overhead in PowerPC), always correct (backup plan)

* Guest virt addr > Runtime disk file + Host virt addr > Swap + Host real addr

* Translation Table Valid bit
  * guest memblock available in host addr space?

* Can used by both Intepreter/BT
  * Unavoidable if Simantic mismatch (64bit simulated on 32bit machine)


#### Direct Translation Methods

* Methods:
  * Guest Address + Base Address -> Host Address
  * Guest Address -> Host Address
* Issue: insufficient space for runtime


#### Compatibility Issues

* SWTT slow, DT fails on ````Guest > Host + Runtime````
* Strategic placement (DT methods)
  * Guest will not always used up Host space
  * Runtime put in region not used by Guest


## 3.4 Memory Architecture Emulation

* Important features of ABI Memory Arch
  1. Overall structure of address space: segments, flat, linear?
  2. Access Privilege supported: R,W,E 
  3. Granularity: Smallest size for allocation and protection

### 3.4.1 Memory Protection

* by SWTT: slow, correct
* by DT: need to implement a protection checking SW table, slow down DT

#### Host-Supported Memory Protection

* Methods:
  1. By syscall to set a page privilege by runtime
  2. Register page fault signal to runtime
 
* simOS: mmap guest address region to serveral files with privilege set  

> * Imple: if guest registered same mem protection handler
> * Pass to interrupt emulator first, check if guest registered? pass to guest: interrupt emu do the job

#### Page Size Issues

* Larger Host page size: 
  * e.g. 2 different privilege guest page fit in single host page
  * but host page can only support one privilege setting

* Possible Sol:
  * realign guest page - must ensure correctness, portability reduced
  * set host page to lesser privilege

* Privilege level mismatch:
  * Host support only R,W but Guest need R,W,E
  * Intepretation
  * BT - checks only runtime read and translate guest code
  * Detecting Guest change protection?

> * new arch support var page sizes


### 3.4.2 Self-Referencing and Self-Modifying Code

#### Basic Method

* or rely on architecture behavior (SPARC flush / PA-RISC cache purge after code modified)


#### Pseudo-SMC

- If data & code mixed (in a same page), pseudo SMC will cause false traps

* Frequent traps cause performance loss, Dynamic Check
  * link Source src to prolog, turn off write protection 
 	when write protection faults occured
 * check if side table == guest memory 
 * if same (SMC not occur) turn on wp, unlink prolog;

#### Fine-Grain Write Protection

- Transmeta: fine-grain checking (for detect edited code)
- Transmeta: do checking on every basic block execute, will still cheaper than false traps

#### True SMC

- remove SMC code from prevent it to execute

#### Self-reference code

* e.g.hg perform checksum on self instruction
* Original copy is still in code cache, thus it will have correct data
* But some VM have code patching approach, will let the checksum wrong
  - ADORE, PIN
* Sol: read-protect? pseudo-SMC will cause false trap

#### Protecting RT Memory

* runtime mode: bt, interpreting etc
* emulation mode: when execute translated guest code
* protection mode need to change when switching between these 2 mode

| Mem Lay | RT | EM |
|---------|----|----|
| RT DATA | RW | NA |
| RT CODE | EX | NA |
| C CACHE | RW | EX |
| Guest D | RW | RW |
| Guest C | Ro | Ro |


## 3.5 Instruction Emulation

### 3.5.1 Performance Tradeoff

### 3.5.2 Staged Emulation


## 3.6 Exception Emulation

- ABI visible
  - exception registered
  - exception (may unregister) that will cause process terminate
- ABI invisible
  - exception (unregistered) but no signal emitted

### 3.6.1 Exception Detection

1. Traps detected by both host & guest platforms (ABI visible on both)
  - We only care about this
2. Traps no detected by host (ABI invisible on host)
  - Real case? Nah
3. Traps no detected by guest and visible on host (ABI visible on host only)
  - Dont care on guest

### 3.6.2 Interrupt Handling

- Interrupt will need precise state of guest
- Runtime hold the interrupt and let guest reach basic block end
  - Problem: guest in loop, and no basic block end reach
  - Sol: unchain the loop in code cache, let it reach bb end
  - if interrpts are much, vm better disable chaining

> traps handling missed QQ

### 3.6.3 Determining Precise Guest State

#### Interpretation

- Easy

#### BT: Locating the PC

- pc side table only have (src PC, trg PC), we can maintain a reverse TST

#### BT: Register State

#### BT: Memory State


## 3.7 OS Emulation

- maintain compatiblity at ABI level, not emulate ins but functions

### 3.7.1 Same OS Emulation

- Host OS functions are available, but calling convention different
  - How parameters passing
  - Value Return

#### OS Call Translation

- Jump to a wrapper code, that convert the parameters and return value to/from host


#### RT-Implemented OS Functions

#### Nasty Realities


### 3.7.2 Different OS Emulation

#### E.g. Enforcing File Limits


## 3.8 Code Cache Management

- Code cache different from hw cache
  - Code cache in memory, regen is expensive
  - Code cache have varient length
  - No backing store
  - Dependences among basic blocks by linking (dangling basic block pointer)

### 3.8.1 Code Cache Implementations

### 3.8.2 Replacement Algorithms

#### LRU

#### Flush When Full

- huge overhead on after flush

#### Preemptive Flush

- Flush when detected guest is going to new phase (while old code will not reuse)
- When new code is translated

#### Double buffer

- Maintain 2 buffers, when a buffer full flush another buffer; change to another buffer and migrate previous reused code to new buffer and flush after second cache full

#### Fine-Grained FIFO

#### Coarse-Grained FIFO

#### Performance

- Intepreter: low startup, high steady state
- BT: high startup, low steady state

- Staged Emulation
  - Adjust optimize level according to frequency of execute

## 3.9 System Environment


## 3.10 Case Study: FX!32

- Inteprete and profile -> offline translate and optimize -> execute new version and profile

- Persistence code cache, slow

## 3.11 Summary

