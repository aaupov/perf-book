---
typora-root-url: ..\..\img
---

## Replace Branches with Predication {#sec:BranchlessPredication}

Some branches could be effectively eliminated by executing both parts of the branch and then selecting the right result (*predication*). Example of code when such transformation might be profitable is shown on [@lst:PredicatingBranchesCode]. If TMA suggests that the `if (cond)` branch has a very high number of mispredictions, you can try to eliminate the branch by doing the transformation shown on the right.

Listing: Predicating branches.

~~~~ {#lst:PredicatingBranchesCode .cpp}
int a;                                             int x = computeX();
if (cond) { /* frequently mispredicted */   =>     int y = computeY();
  a = computeX();                                  int a = cond ? x : y;
} else {
  a = computeY();
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For the code on the right, the compiler can replace the branch that comes from the ternary operator, and generate a `CMOV` x86 instruction instead. A `CMOVcc` instruction checks the state of one or more of the status flags in the `EFLAGS` register (`CF, OF, PF, SF` and `ZF`) and performs a move operation if the flags are in a specified state or condition. Similar transformation can be done for floating-point numbers with `FCMOVcc,VMAXSS/VMINSS` instructions. [@lst:PredicatingBranchesAsm] shows assembly listings for the original and the branchless version.

Listing: Predicating branches - x86 assembly code.

~~~~ {#lst:PredicatingBranchesAsm .bash}
# original version                  # branchless version
400504: test edi,edi                400537: mov eax,0x0
400506: je 400514                   40053c: call <computeX> # compute x; a = x
400508: mov eax,0x0                 400541: mov ebp,eax     # ebp = x
40050d: call <computeX>      =>     400543: mov eax,0x0
400512: jmp 40051e                  400548: call <computeY> # compute y; a = y
400514: mov eax,0x0                 40054d: test ebx,ebx    # test cond
400519: call <computeY>             40054f: cmovne eax,ebp  # override a with x if needed
40051e: mov edi,eax
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In contrast with the original version, the branchless version doesn't have jump instructions. However, the branchless version calculates both `x` and `y` independently, and then selects one of the values and discards the other. While this transformation eliminates the penalty of a branch misprediction, it is potentially doing more work than the original code. Performance improvement, in this case, very much depends on the characteristics of `computeX` and `computeY` functions. If the functions are small and the compiler is able to inline them, then it might bring noticeable performance benefits. If the functions are big, it might be cheaper to take the cost of a branch mispredict than to execute both functions. 

It is important to note that predication does not always benefit the performance of the application. The issue with predication is that it limits the parallel execution capabilities of the CPU. For the original version of the code, the CPU can predict that the branch will be taken, speculatively call `computeX` and continue executing the rest of the program. This type of speculation is not possible for the branchless version as the CPU has to wait for the result of the `CMOVNE` instruction to proceed.

The typical example of the tradeoffs involved when choosing between the regular and the branchless versions of the code is binary search:[^3]

* For a search over a large array that doesn't fit in CPU caches, a branch-based binary search version performs better because the penalty of a branch misprediction is low comparing to the latency of memory accesses (which are high because of the cache misses). Because of the branches in place, the CPU can speculate on their outcome, which allows loading the array element from the current iteration and the next one at the same time. It doesn't end there: the speculation continues, and you might have multiple loads in flight at the same time.
* The situation is reversed for small arrays that fit in CPU caches. The branchless search still has all the memory accesses serialized, as explained earlier. But this time, the load latency is small (only a handful of cycles) since the array fits in CPU caches. The branch-based binary search suffers constant mispredictions, which cost roughly 10-20 cycles. In this case, the cost of a mispredict is much more than the cost of a memory access, so the benefits of speculative execution are hindered. The branchless version usually ends up being faster in this case.

The binary search is a neat example that shows how one can reason about when choosing between standard and branchless implementation. The real-world scenario can be more difficult to analyze, so again, measure to find out if it would be beneficial to replace branches in your case.

Without profiling data, compilers don't have visibility into the misprediction rates. As a result, compilers usually prefer to generate branches, i.e. original version, by default. They are conservative at using predication and may resist generating `CMOV` instructions even in simple cases. Again, the tradeoffs are complicated, and it is hard to make the right decision without the runtime data. HW-based PGO (see [#sec:secPGO]) will be a huge step forward here. Also, there is a way to indicate to the compiler that a branch condition is unpredictable by hardware mechanisms. Starting from Clang-17, the compiler now respect a `__builtin_unpredictable`, which can be very effective at replacing unpredictable branches with `CMOV` x86 instructions. For example:

```cpp
if (__builtin_unpredictable(x != 2))
  y = 0;
if (__builtin_unpredictable(x == 3))
  y = 1;
```

BOLT has an experimental CMOV conversion optimization[^1] that leverages LBR misprediction information available in recent microarchitectures (Skylake and higher). However, LLVM's CMOV conversion passes[^2][^4] that take dataflow and scheduling information into account are still generally more effective. A hybrid or cooperative approach might yield the best result but there has not been much progress in this area.

[^1]: BOLT CMOVConversion - [https://github.com/llvm/llvm-project/commit/687e4af1c05ae36af88900d41150e260d8f273c0](llvm/llvm-project@687e4af).
[^2]: LLVM cmov-vs-branch optimization - RFC [https://discourse.llvm.org/t/rfc-cmov-vs-branch-optimization/6040](https://discourse.llvm.org/t/rfc-cmov-vs-branch-optimization/6040), [https://github.com/llvm/llvm-project/commit/ca7c307d18160485ab30289dc1b14bd16ac35c52](llvm/llvm-project@ca7c307) `SelectOpti` commit.
[^3]: Discussion on branchless binary search - [https://stackoverflow.com/a/54273248](https://stackoverflow.com/a/54273248).
[^4]: LLVM `X86CmovConverterPass`.
