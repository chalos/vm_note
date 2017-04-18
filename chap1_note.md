# Chapter 1

* 機器（Machine) = 平臺 （Platform);
  OS角度，機器=硬體;
  軟體角度，機器=OS+可訪問硬體
   
* Virtualize & Abstractize
    - 虛擬化不隱藏細節
    - guest 做 action e *(eg MPIS: add r1,r1,r2)* 讓 state A -> B
      host 也需要做 action e' *(eg x86: add eax, ebx)* 讓 state A' -> B'
      而其中，需要有一個 virt-real 的 mapping function V(X)=X'
      
* Emulation vs Simulation
    - Emulation 仿真，真的去做被模仿的機器(target)做的事；慢
    - 而Simulation，英文可以變成 silimiar emulation，讓外面看起來是在Emulate；比Emu快


## 1.1 Architecture

````
|-----------1-----------------------| - SW
|         app                       |
|      |------------2-----------|   |
|      |              lib       |   |
|---3--|--------------3-----|   |   |
|             os            |   |   |
|---4---|--5-|----6-----|   |   |   |
| drver | fs | tsk dis. |   |   |   |
|---8---|--8-|-----8----|-8-|-7-|-7-| - ISA
|       exec hardware               | - HW
|       aka CPU           |-----9---|
|                         |    MMU  |
|---------10--------------|-10-|    |
|           system bus         |    |
|---------11----------|---|-11-|-12-|
|      ctrl           |   |   ctrl  |
|---------13----------|   |----14---|
|      I/O & NIC      |   |   RAM   |
|---------------------|   |---------|
````

* ISA [7] [8]，區分軟、硬體
    - 同一個 ISA 軟體執行在不同硬體上
    - User ISA [7], System ISA [8]
    
* ABI [3] [7]
    - User Level Machine Code (ULMC) [7]
    - syscall來使用 System Level Machine Code (SLMC) [3]

* API [2] [7]
    - Abstraction of ABI+, define on source code level
    - e.g. clib
 

## 1.2 VM basis

### Defination of Machine

* Pov of User Prog: OS + ULMC
* Pov of System: ISA

### Virtualization: 

* 放一層虛擬軟體在中間

* VM 可分爲:
    1. Process VM
    2. System VM

* Defination:
    - Host: Underlying Machine, 虛擬軟體下面的機器
    - Guest: 虛擬軟體上面的軟體
    - Native Machine: System VM 下的 host (?)
    - Runtime: 運行中的虛擬軟體
    - Runtime Software: 運行時的Guest (?)
    - VMM: System VM 下的虛擬軟體


## 1.3 Process VM

````
|-----------------------------------| - SW
|         app                       |
|               |---------------|   |
|               |     lib       |   |
|======3==============3=====||  |   |
|                           ||  |   |
|             os            ||  |   |
|                           ||  |   |
|---------------------------|=7=|=7=| - ISA
|            CPU                    | - HW
...
````
* 虛擬軟體位置: [3] [7] 有 '=' 的地方 (ABI)

### 1.3.1 Multiprogram/Concurrent Program Design

* 也叫 Virtual Memory (?)

* 每個 Process 覺得它們擁有全部資源

* OS 提供每一個 Process 一個一樣的 Process VM (每個 Process 看見相同的 Kernel Space?)

### 1.3.2 Emulator and Dynamic BT for non-same ISA

* 一個ISA的機器上執行另外一個ISA的軟體

* 例子: src->target, OS
    - **Rosetta**: PowPC->x86, Mac
    - **FX!32**: x86-32->DEC Alpha, WinNT
    - **IA32-EL**: x86-32->IA64, IPF
    - **Aries**: HP PA-RISC->IA64, IPF

* 上述例子舉的，都是在相同 OS 下的

````
|---------------------------|
|     app                   |
|------------------|        |
|     os           |        |
|------------------|===EMU==|
````

* 最直覺可以使用 Interprete 方式一道道翻譯; 初始化快，執行效能不佳

* Binary Transformation，將 Binary 先翻譯一次cache起來，再執行; 初始化慢，較能較好

* 搭配使用，先 Profile 找出程式碼裡面的 hotspot，將其 BT


### 1.3.3 Dynamic Optimizer under same ISA

* Emu 做 BT 後，同 ISA 下的 Dynamic Optimizer 在 Emu 上收集資訊分析，on the fly 進行 patching 優化後的 binary 並執行

* 例子: Dynamo

### 1.3.3+ Dynamic Instrumental under same ISA

* 類似 profiler 那樣，Instrumental Sw 執行後再將 target code 執行起來

* 例子: Valgrind, [PIN](https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool) (by Intel)

### 1.3.4 High Level Language VM

* Portable

* Host = OS + Machine; Binary = IR;

* 例子: Java, MS CLI

* 另外 LLVM，將 M 種 HLL 轉成固定格式的 IR，再編到 N 種不同的平台上


## 1.4 System VM

* 歷史：
    - 機器早期昂貴，使用者共享機器，希望使用不同OS/環境
    - 機器便宜，VM不流行
    - 現在因爲軟體隔離、軟體移植性: 開發測試部署環境、SDx等又流行

 * VMM 提供對平臺的複製 (像 OS 對 Process 那樣)

### 1.4.1 Implementation of VM under same ISA

* Type 1: Bare Metal
    - VMM 權限最高，攔截所有Guest對底層的互動
    - 效能較高，VMM需要有底下硬體的driver

* Type 2: Hosted VM
    - 現有的OS上安裝，可以直接依賴OS上有的硬體driver
    - SLMC時，需要經過多層軟體，效能較差

* Host & Guest 相同 ISA

### 1.4.2 Emulator under non-same ISA

* Host & Guest 不同 ISA

* Whole System VMs，需透過 Interpreter or BT 來從 原ISA 轉到 新ISA

* Challenges:
    - 兩種 ISA 相差太大，無法更直接的轉換後使用 (register多->少)

### 1.4.3 Co-design VM under non-same ISA

* VM 效能不佳，而 協作VM 把性能損失降到最小

* Transmeta Crusor 用 VLIW (+ins reordering) 來提升效能

* 計算機架構 [ILP章節 $3.7](https://ubuntu.chalos.xyz/~chalos/pdf/CA/H&P%205th%20ed-CA-4-Ch3%20(Instr%20Level%20Parallelism%20ILP).pdf) 有提到
    - static binary translation (reordering)
    - 中間可能再有一層另外設計過的 dynamic optimization 硬體

## 1.5 分類
````
VM分類
  |- Process VM
  | |- Same ISA
  | | |- Multiprogramming (大部分的OS支援)
  | | `- Dynamic Optimizer & Instrumental (Dynamo) & (PIN, Valgrind)
  | `- Different ISA 
  |   |- Emu & Dynamic BT (FX!32, IA32EL, Rosetta, Aries)
  |   `- HLL VM (Java, MS Cli)
  |
  `- System VM
    |- Same ISA
    | |- Hypervisor/VMM (Type1 & 2: Xen & VMware, KVM)
    `- Different ISA
      |- Emulator (QEMU, Virtual PC)
      `- Co-Design VM (Transmeta)
````

## 1.6 總結

* 書中分享了一個運行層層VM的例子 XD

````
Java App
 |
 | JVM (Process VM, Diff ISA)
 |
Java Runtime Process
 |
 | MultiProgramming by OS (Process VM, Same ISA)
 |
Linux IA32
 |
 | VMWare (System VM, Same ISA)
 |
Window IA 32
 |
 | Transmeta (System VM, Diff ISA)
 |
Transmeta Crusoe VLIW
````
