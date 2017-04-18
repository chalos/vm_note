# Chap4

* GCC Profiler Guild Optimization
 * Feed back profile data for next compilation

* IVE - Intel Value Engine

* Different Profile Different Input: e.g: strcpy
 * depends on machine longest word, src string size... 

* Static compilers hard in age of \*LP

* Optimize in runtime and static time: desicion

* DBO:
 * Pros: hotcode, lack constraint, env
 * Cons: runtime overhead[1], lack source meta

* Question:
 * [1] Optimize once run forever
 * Profile: Speculation Error rate, ...
 * Arch Independent/ MArch Dependent: 
````
while(p!=null) {p->x...;}
// speculation will let p->x run first before cmp p with null
// but what if p is really null?
````

* MArch Dependent
 * Fat, Chunby binary

## Overview

* Identify Hotcode
 * Edges: jmp to stub, count+1, jump target

* D. Profile
 * execution frequencies
 * sw, hw, hybrid impl
 * hybrid e.g.: prefetched cache-line marked by hw, sw check if cache access is marked cache-line

* Profiling
 * What to profile? br (taken or not), jmp (addr), cache misses (behaviour: always happen on specified ins), data value (math func args->value map)

* Profile Type
 * Edge, Path

### Path Profile

* HW support makes pp efficient
 * Intel Program Trace

* SW: Path Encoding

* Compiler: Split intersect block into 2
 * A-50-C-50-D & B-50-C-50-E: C should go D or E by history based?
 * Solution ACD & BC'E

### Sampling

* Randomize sampling interval

* Concat non-contigous basic block -> improve locality

