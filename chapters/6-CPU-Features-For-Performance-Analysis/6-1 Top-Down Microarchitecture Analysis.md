---
typora-root-url: ..\..\img
---

## Top-down Microarchitecture Analysis {#sec:TMA}

Top-down Microarchitecture Analysis (TMA) methodology is a very powerful technique for identifying CPU bottlenecks in the program. It is a robust and formal methodology that is easy to use even for inexperienced developers. The best part of this methodology is that it does not require a developer to have a deep understanding of the microarchitecture and PMCs in the system and still efficiently find CPU bottlenecks. However, it does not automatically fix problems; otherwise, this book would not exist.

At a high-level, TMA identifies what was stalling the execution of every hotspot in the program. The bottleneck can be related to one of the four components: Front End Bound, Back End Bound, Retiring, Bad Speculation. Figure @fig:TMA_concept illustrates this concept. Here is a short guide on how to read this diagram. As we know from [@sec:uarch], there are internal buffers in the CPU that keep track of information about instructions that are being executed. Whenever a new instruction is fetched and decoded, new entries in those buffers are allocated. If an uop for the instruction was not allocated during a particular cycle of execution, it could be for one of two reasons: either we were not able to fetch and decode it (Front End Bound); or the Back End was overloaded with work and resources for the new uop could not be allocated (Back End Bound). Uop that was allocated and scheduled for execution but not retired is related to the Bad Speculation bucket. An example of such a uop can be an instruction that was executed speculatively but later was proven to be on a wrong program path and was not retired. Finally, Retiring is the bucket where we want all our uops to be, although there are exceptions. A high Retiring value for non-vectorized code may be a good hint for users to vectorize the code (see [@sec:Vectorization]). Another situation in which we might see a high Retiring value but slow overall performance is in a program that operates on denormalized floating-point values, thus making such operations extremely slow (see [@sec:SlowFPArith]).

![The concept behind TMA's top-level breakdown. *© Image from [@TMA_ISPASS]*](../../img/pmu-features/TMAM_diag.png){#fig:TMA_concept width=90%}

