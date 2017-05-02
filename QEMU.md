# QEMU

參考：
http://people.cs.nctu.edu.tw/~chenwj/slide/QEMU/QEMU-tcg-01.txt
https://lists.gnu.org/archive/html/qemu-devel/2011-04/pdfhC5rVdz7U8.pdf
http://wiki.qemu.org/Documentation/TCG/frontend-ops

## 0

0.10 版本後都使 TCG

src 的 macro很多很複雜，configure 時候，可以再 CFLAG上加 -save-temps 來生成沒有 macro 的 *.i 檔案
````
./configure --extra-cflags="-save-temps"
````

## TCG

- source guest binary -> TCG IR -> target host binary
- Debug: `--enable-debug-tcg`
- 執行 qemu 時: `qemu -d out_op,out_asm` 來看 tcg log


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

- `cpu_loop()` -> `trapnr = cpu_x86_exec(env)` -> `prologue` -> tc ... eob/trap -> `epilogue` -> `switch(trapnr)` -> `process_pending_signals(env)`
- `cpu_loop()` 處理 IRQ, translation, 執行guest`cpu_loop()`
- `prologue` 做 push EM stack (aka context switch) + save to memory
- `epilogue` 設 return addr

````
loop:
	if(setjump(env->jmp_env) is 0): 
		# excption handler

	next_tb <= 0
	loop:
		# to code cache
		next_tb <= tcg_qemu_tb_exec(tc_ptr)
````

##### TCG

- tcg/tcg.c : tcg_prolouge_init => tcg/i386/tcg-target.c : tcg_target_qemu_prologue

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
- `TCGv` => `TCGv_i32` => `int`
- `TCGv_ptr` => `TCGv_i32` => `int`

##### DisasContext (DS)
- `DisasContext` 在 translate.c 內宣告
- Self modify guest code 用的 meta

##### Function`gen_eob()`
- end of block



## Traces

##### From `case ret`
````
0 gen_pop_T0(DS* s)
 1 gen_op_movl_A0_reg(reg=R_ESP=4)
  2 tcg_gen_mov_t1(ret=cpu_A0=15, arg=cpu_regs[reg=4])
  => tcg_gen_mov_i32($@)
  if(ret != arg):
   3 tcg_gen_op2_i32(opc=INDEX_op_mov_i32,15,cpu_regs[4])
 1 gen_op_ld_T0_A0(idx=s->dflag+1+s->mem_index=2)
  2 gen_op_ld_v(idx=2, t0=cpu_T[0]=13, a0=cpu_A0=15)
  mem_idx = idx>>2-1 = -1
  switch(idx & 0xff):
  case2:
   3 tcg_gen_qemu_ld32u(ret=t0=13, addr=a0=15, mem_idx=-1)
    4 tcg_gen_op3i_i32(opc=INDEX_op_qemu_ld32. ret=13, addr=15, mem_idx=-1)
0 gen_pop_update(DS *s)
 1 gen_stack_update(DS *s, addend=2<<s->(dflags=1)=4)
  2 gen_op_add_reg_im(size=1, reg=R_ESP=4, val=added=4)
   switch(size)
   case1:
   3 tcg_gen_addi_t1(ret=cpu_tmp0=-1, arg1=cpu_regs[reg=4]=15, arg2=val=4)
    4 tcg_const_i32(arg2) = t0 = 26
    4 tcg_gen_add_i32(ret=-1,arg1=15,t0=26)
     5 tcg_gen_op3_i32(INDEX_op_add_i32, ret=-1, arg1=26, arg2=4)
    4 tcg_temp_free_i32(t0=26)
   3 tcg_gen_ext32u_t1(ret=cpu_tmp0=-1, arg=cpu_tmp0=-1)
    => tcg_gen_mov_i32($@)
     if(ret != arg):
    4 tcg_gen_op2_i32(opc=INDEX_op_mov_i32,-1,-1)
   3 tcg_mov_t1(cpu_regs[reg=4]=9, cpu_tmp0=-1)
0 gen_op_jmp_T0()
0 pop_stack(cpu_env=0, next_eip=cpu_T[0])
0 gen_eob(DS* s)
 1 tcg_gen_exit_tb(val=0)
  2 tcg_gen_op1i(INDEX_op_exit_tb, val=0)
 1 s->is_jmp = DISAS_TB_JUMP = 3
````