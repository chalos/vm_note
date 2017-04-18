# Chapter 2

## Intro of Emulation

> What is Emulation?

* 在 System/Subsystem A 執行 System/Subsystem B 的 Method

* Source ISA vs Target ISA
  - Source: 原本的程式碼 ISA，將會被翻譯
  - Target: 執行該翻譯程式的機器ISA

* 完整的 ISA 包括：
  - Registers
  - Memory Structure
  - Instruction Set
  - Trap and Exceptions

* Emulation 可分為：
  - Interpreter
  - Binary Translation


## 2.1 基本 Interpreter

````
src mem state                         ctx block
|--------|                           | PC   |
| .code  |                           | flag |
|--------|      |-------------|      | reg0 |
| .data  | <--> | Interpreter | <--> | reg1 |
|--------|      |-------------|      | reg2 |
| .stack |                           | ...  |
|--------|                       
```` 

### Decode and Dispatch Interpreter

* main loop -> Decode指令 -> 分派 dispatch 給不同callback

* Source ISA: Power PC 

 ```` C
while(!halt && !interrupt) {
    inst = code[PC];
    opcode = extract(inst, 31, 6);
    switch(opcode) {
        case LoadWordAndZero: LoadWordAndZero(inst);
        case ALU: ALU(inst);
        ...
    }
}
void LoadWordAndZero(char* inst) {
    RT = extract(inst, 25, 5);
    RA = extract(inst, 20, 5);
    d = extract(inst, 15, 16);
    source = (RA == 0)? 0 : regs[RA];
    addr = source + d;
    regs[RT] = (data[addr] << 32) >> 32;
    PC += 4;
}
void ALU(char* inst) {
    RT = extract(inst, 25, 5);
    RA = extract(inst, 20, 5);
    RB = extract(inst, 15, 5);
    ext = extract(inst, 10, 10);
    s1 = regs[RA];
    s2 = regs[RB];
    source = (RA == 0)? 0 : regs[RA];
    switch(ext) {
        case ADD: ADD(inst);
        ...
    }
    PC += 4;
}

 ````

* 1 Load Word = 10 target ISA ins count
* Go through the Switch-case, Low efficiency


## 2.2 Threaded Interpreter

* 把 Dispatch 加在每個 action的末端
* 少了每一個loop去 go through Switch-case

 ```` C
 LoadWordAndZero:
    RT = extract(inst, 25, 5);
    RA = extract(inst, 20, 5);
    d = extract(inst, 15, 16);
    source = (RA == 0)? 0 : regs[RA];
    addr = source + d;
    regs[RT] = (data[addr] << 32) >> 32;
    PC += 4;
    if(halt || interrupt) goto EXIT;
    inst = code[PC];
    opcode = extract(inst, 31, 6);
    ext = extract(inst, 10, 10);
    next = dispatch[opcode, ext];
    goto *next;
    
 ADD:
    RT = extract(inst, 25, 5);
    RA = extract(inst, 20, 5);
    RB = extract(inst, 15, 5);
    ext = extract(inst, 10, 10);
    s1 = regs[RA];
    s2 = regs[RB];
    regs[RT] = s1 + s2;
    PC += 4;
    if(halt || interrupt) goto EXIT;
    inst = code[PC];
    opcode = extract(inst, 31, 6);
    ext = extract(inst, 10, 10);
    next = dispatch[opcode, ext];
    goto *next;
    
 ````


## 2.3 Pre-decode & Direct Threaded Interpreter

### 2.3.1 Predecode

#### CISC PowerPC ISA 為例

* 可以將一些基本的ALU指令集(AND,OR,ADD,SUB...)組合成一個IR
 
 ````
 0x0000: lwz r1,8(r2) -> 0: 7,1,2,8
 0x0004: add r3,r3,r1 -> 1: 8,3,3,1
 0x0008: stw r3,0(r4) -> 2: 37,3,4,0
 ````

* 而每個指令對應的routine可以用定義的offset取代
* SPC在runtime時候依然需要被保留，用來對應indirect branch的例子
 
 ````
  struct instruction {
    unsigned long op;
    unsigned char dest;
    unsigned char src;
    unsigned int src2;
  } code [CODESIZE]; // pre-decoded data
  
  LoadWordAndZero:
    RT = code[TPC].dest;
    RA = code[TPC].src1;
    d = code[TPC].src2;
    source = (RA == 0)? 0 : regs[RA];
    addr = source + d;
    regs[RT] = (data[addr] << 32) >> 32;
    TPC += 1; //< maintain an addtional target pc
    SPC += 4; //< original source pc
    if(halt || interrupt) goto EXIT;
    opcode = code[TPC].op;
    next = dispatch[opcode];
    goto *next;
  
 ````


### 2.3.2 Direct threaded interpreter

* 把opcode記錄成 function pointer
 
 ````
 0x0000: lwz r1,8(r2) -> 0: &lwz,1,2,8
 0x0004: add r3,r3,r1 -> 1: &add,3,3,1
 0x0008: stw r3,0(r4) -> 2: &stw,3,4,0
 
 ...
 opcode = code[TPC].op;
 // next = dispatch[opcode]; //< 少了查表的動作
 goto *opcode;
 ````


### 2.3.3 Compiled Emulation (aka Code Morphing)

* 把每一段指令變成C的inline function重寫成C code
* 用 target ISA compiler 直接 compile
 
 ````
 main.c:
 lwz(1,2,8); //< lwz r1,8(r2)
 add(3,3,1); //< add r3,r3,r1 
 stw(3,4,0); //< stw r3,0(r4)
 ````

* 像Java JIT的做法那樣，把一個 source ISA 再編譯成 target native code 來加速


## 2.4 CISC Interpreting


## 2.5 Binary Translation

* Interpreting會依據對每一個指令的實做去執行
  - Source每到指令印象的Registers狀態都被Abstract

* BT 則是讓每一道Source指令盡可能map到Target指令去
  - 至少Source 和 Target Register 一對一 Mapping
  - Translated sequence 可再透過優化去減少code size


## 2.6 Binary Translation Problem

### 2.6.1 Code Discovery

* Static 很難拿到執行時的資訊(Horspool, Marovac證明可以拿到)

  * Indirect Jump 無法得知jump到的位子

  * 忽略 ijmp: 翻譯jump的下一條指令，不能確定jump指令後是data/指令/padding

  * CISC長度不一: 每一個byte offset都可能是合理指令，或沒法判斷padding問題

 ````
       | movl %esi, 0x08030000(%ebp)
 31 c0 8b b5 00 00 03 08 8b db 00 00 03 00
          | mov %ch, 0 ??
 ````


### 2.6.2 Code Location

* Indirect Jump 的 SPC 和 TPC 的 Mapping 沒辦法預測，需要執行起來後才能知道

```` asm
movl %eax, 4(%esp)
jmp %eax ; SPC = eax, TPC = ?
````


### 2.6.3 Incremental Predecode or Dynamic BT

* 邊執行邊翻譯，再cache起來

* Emulator Manager 提供 Runtime support
* Interpreter 執行 source binary
* Binary Translation 則將 interpreter 程式轉危為 target binary
* Map Table： Key=SPC, Val=TPC

* next SPC:TPC cache miss -> EM Interpret -> 存入Code Cache -> 把source binary起始SPC對應到Code Cache的位子(TPC) 寫到 address mapping table 中

````
--------------                      ----------
| SPC to TPC |                      | Source |
| hash table |                     /| Binary |
--------------   /-> [Interpreter]<     |
     |          / miss               [Translator]
-------------- /                        |
| Emulation  |/                    --------------
| Manager    |-------------------->| Code Cache |
--------------                     --------------
````

* Basic Block: 每次 translate 只處理一個 BB 單位
  - static bb: labels and branches
  - dynamic bb: branches only
  - amount of dynamic bb > static bb, 因為一個loop指令，在static只會出現一次，而在dynamic會出現N次
  - size(single) of dynamic bb > static bb, static 由 labels&branches 切割，dynamic 由只有 branches 切割

* 每到 Basic Block(branch) 結束，Interpreter 則會 context switch 到 EM 上去檢查即將 jump 去的 TPC 位址 


#### Tracking Source Program Code

* EM 和 Target Code 的切換：
  - 執行 Target Code, 用 branch-link 取代 branch 跳到 EM, 將 SPC 嵌在 bl 指令後面，讓 EM 透過 link reg 相對位子去存取
 - 不ret link register 的位子，而是跳到 lookup 後的位子

````
; source code
jmp label

label:
...
````
````
; target code
bl _EM
.int {SPC}

label:
...

_EM:
mov r1, lr
ldr r1, [r1+4] ; r1 = {SPC} data
mov r0, r1
bl hashlook_up ; r0 = {TPC} = label
mov pc, r0
````


### 2.6.4 Same ISA Emulation

* Monitoring (Code Management)

* Optimization


## 2.7 Control Optimization

* Reduce overhead for SPC - TPC mapping in every basic block


### 2.7.1 Translation Chaining

* 如同 Tracking source program, 只是會將 jump 到 EM 的 指令換成 jump TPC

* 不適用 indirect jump

````
bl _EM
.int {SPC}

label:
...

_EM:
mov r1, lr
ldr r1, [r1+4] ; r1 = {SPC} data
mov r0, r1
bl hashlook_up ; r0 = {TPC} = label
mov r2, r0
bl generate_jmp_ins ; r0 = bl r0
str r0, [r1-8] ; bl EM => bl r0
mov pc, r2
````


### 2.7.2 SW I-Jmp Predict (similiar to TLB)

* EM 保留 side table，像 caching 一樣保留 N 個常用的 mapping


### 2.7.3 Shadow Stack

* Procedure call 到 ret 時 pop SPC 需要 EM hashlookup SPC，shadow stack 減少這一次的查照

* 在 runtime stack 旁弄一個 shadow stack (SPC-TPC mapping)
 - call SPC: basic block translated, push SPC-TPC to shadow stack
 - ret: rt.TPC = ( rt.SPC == ss.SPC )? ss.TPC : hashlookup(rt.SPC)


## 2.8 Instruction Set Issues

### 2.8.1 Register Arch

* Target ISA 的 regs 被用來當作：
  - Source ISA 的 General/Special Purpose registers 
  - 指向 runtime 時 Source Context 和 Memory Image 的位置 (至少2組)
  - EM 運算所需
  
 所以Target ISA regs 數量需 **遠大於** Source ISA regs

* 問題: Source ISA regs N >= Target ISA regs N


### 2.8.2 Condition Codes

* condition codes 更新頻繁

* 最容易的狀況：Source ISA 不用任何 cc
* 最困難的情況：Source ISA 有許多複雜的 cc 而 Target ISA 不用任何 cc

* Lazy Evaluation: 
  - 有機會更新 cc 的*指令*和*所有參數*存起來，有需要時再調閱

* Data flow analysis: 
  - BT 時候可以只在需要用 cc 的指令(jmp)前，才嵌入慾更新 cc 值的指令進 target code

* 將每個 cc 設定一個 reserved reg 來加速 emulation，Lazy Eva 則是每一個 cc 都 存有最後一組會改變該 cc 狀態的*指令*和*所有參數*


### 2.8.3 Data Format and Arithmetic

#### Data Format
* Integer(2's compliment) & FP(IEEE standard) 在現代ISA內都標準化
  - 舊 ISA 需要被模擬
  - 熱門 ISA 常被模擬 ：IA32 用 80 bit FP

#### Arithmetic
* 獨特的指令，軟體模擬完整度缺佳
  - PowerPC mul-add，簡單地拆開用乘、加無法完整重現結果

* 實作不同，導致不一樣精準的答案
  - 除法：小數點位移成整數再除 vs 直接對浮點進行除法


### 2.8.4 Memory Address Resolution

* 指令所能讀/寫的資料大小 (byte, half-word, full-word...)

* worst case 用到 per-byte granularity 讀寫一個 word


### 2.8.5 Memory Data Alignment

* Miss Data Alignment 導致多餘的 cache miss

* 如何解決 MDA？
  - Exception Handler: ?
  - Compiler: ````__unaligned```` 宣告指標不對其 或 ````__declspace(align(2^N))```` 宣告對2^N bit對齊的記憶體
  - C runtime 去對齊

* BT 處理 MDA
  - 偵測: Static/Dynamic Profiling, Exception Handling 
  - 處理: per-byte 讀寫

### 2.8.6 Data Order (Big & Little Endian)

* Most ISA support both Endian


### 2.8.7 Address Arch

* 32 bits vs 64 bits -> different size of address space


## HQEMU

### HQEMU vs QEMU

* QEMU 少了優化

* HQEMU 用**2**條thread：傳統QEMU、將 源code 轉換為 LLVM 做優化
  - 優化問題：很多bug、高度重複使用的code才有優化的價值

## LLBT (LLVM based Static BT)

* Front End 從 原本 source code 換成 source binary

* Back End 把 LLVM IR + { *source images (.rodata, .data 等資訊) + linker script* } 翻譯成 target code


## 2.10 Summary - Perf trade off

### Decode-Depatch Interpreter
* Memory: **Low**, 1 ins per routine
* Init perf: **Fast**, interpreter common advantages
* Steady state perf: **Slow**, interpreter common disadvantages
* Code Portability: **Good**


### Indirect Thread Interpreter
* Memory: **Low**
* Init perf: **Fast**
* Steady state perf: **Slow** 
* Code Portability: **Good** 


### Direct Thread Interpreter with Pre-decode
* Memory: **High**, predecode need memory 
* Init perf: **Slow**, predecode need time
* Steady state perf: **Medium**
* Code Portability: **Medium**, with hardcoded routine address will cause its IR non-compatible


### BT
* Memory: **High**, Dynamic: Predecode need memory on new SPC
* Init perf: **Slow**, Dynamic: Predecode and generate target ISA
* Steady state perf: **Fast**
* Code Portability: **Poor**, only work on target ISA


### DBT

#### Why BT?
* 沒有原始碼
* 可以 使用/翻譯 編譯好的 shared library

#### Why Dynamic?
* 可被監視
* 直接使用 target 環境的 library, static需要將 library 翻譯和預先linking
* static BT 問題 (code discovery, code location)