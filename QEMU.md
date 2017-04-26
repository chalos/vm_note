# QEMU

參考：http://people.cs.nctu.edu.tw/~chenwj/slide/QEMU/QEMU-tcg-01.txt

## 0

0.10 版本後都使 TCG

src 的 macro很多很複雜，configure 時候，可以再 CFLAG上加 -save-temps 來生成沒有 macro 的 *.i 檔案
````
./configure --extra-cflags="-save-temps"
````

## TCG

source guest binary -> TCG IR -> target host binary

#### IR 類別
* Move
* Logic
* Arith
* Branch
* Func: call, ret
* Mem Op: ld, st
* Qemu Op: tb_exit, goto_tb, qemu_ld, qemu_st


#### Note
- tcg.i 可以看到 TCG IR 的Op
- target-ARCH/* -> src 到 TCG IR
- tcg/ARCH/* -> TCG IR 到 target


#### Data Structure
- `gen_opc_buf`, `gen_opparam_buf`分別放 TCG opcode, operand, 在 `translate-all.c`
- 靜態配置緩衝區 exec.c  `static_code_gen_buffer` 為code cache
- 跳入 ccache 前, 呼叫 `prologue` (exec.c)
- 跳出 ccache 後, 呼叫 `epilogue` (exec.c)
- i386 開始流程:
  - linux-user/main.c : main
  - exec.c : cpu_exec_init_all
  - target-i386/helper.c : cpu_init => cpu_x86_init
  - tcg/tcg.c : tcg_prolouge_init
  - linux-user/main.c : cpu_loop
  - cpu-exec.c : cpu_x86_exec => cpu_exec

##### linux-user/main.c : cpu_loop

````
loop:
	if(setjump(env->jmp_env) is 0): 
		# excption handler

	next_tb <= 0
	loop:
		# to code cache
		next_tb <= tcg_qemu_tb_exec(tc_ptr)
````

##### tcg/tcg.c : tcg_prolouge_init => tcg/i386/tcg-target.c : tcg_target_qemu_prologue

- QEMU branch 以 function call方式到 cc 裡面執行
- 不同平台有不同calling convention，tcg_prologue_init 產生 prologue/epilogue 的工作 轉交 tcg_target_qemu_prologue (?

````
tcg_target_qemu_prologue(TCGContext *tcgctx):
	tcg_out_modrm(
		tcgctx, OPC_GRP5, EXT5_JMPN_Ev, 
		tcg_target_call_iarg_regs[0] )
	
	tb_ret_addr = tcgctx->code_ptr
	# back to tcg_qemu_tb_exec
````

##### CPUState

- `CPUX86State env`，也叫 `CPUState`: 保存 x86 registers狀態
- `#define CPU_COMMON`: 保存 ISA 通用資料的結構

##### TranslationBlock
- `TranslationBlock` [exec-all.h]: (TB)也叫 Basic Block，也保存 basic block 對應 guest binary 的 pc, cs_base, eflags.. 
- `TB->(uint8_t*)tc_ptr` 指向該 TB 的 ccache 
- `TB` 裡面也會指向其他 `TB->{*phys_hash_next, *page_next[2], *jmp_next[2], *jmp_first}`

##### PageDesc
- `PageDesc` [exec.c]: (PD)Guest Page 的第一個 `TB* fisrt_tb`... 
- `PD` 用於 Page replacement 優化：guest page 可能被改寫／清空，qemu 只需對該 page 有關聯的 code 做 sequential 清理
- `TB->(ulong)page_addr[2]` 存放 TB 的 runtime code/data 的 guest page
- `TB->(TB*)page_next[2]`：用 `PD->first_tb` 找 guest page 的第一個 TB, `TB->page_next` 再找下一個 TB
- QEMU 維護一個 multilevel page table `void *l1_map[1<<12]`
- `PD* page_find(idx)=>page_find_alloc(idx, 0)` [exec.c] 則會根據 index 搜索 `l1_map` 內的 PD

##### TCGContext
- 生成 TCG IR 的 meta

##### DisasContext
- Self modify guest code 用的 meta

