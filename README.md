# RegAlloc — C++20 header-only register allocation toolkit

**130 individually named method/technique headers**, with a dependency-free reusable register-allocation model. The public classes follow the requested naming style, e.g. `regalloc::GeorgeAppelGC`, `regalloc::WimmerFranzLS`, `regalloc::LLVMGreedyRA`, `regalloc::ILPRA`.

## What is actually implemented

This is a **working educational/reference library**, **not** 110 independently reproduced research implementations or a drop-in replacement for a production compiler allocator. All 110 method classes have executable `allocate` functions (not empty scaffolds), but closely related names delegate to configurable shared engines. They are differentiated by policies (priorities, conservative/aggressive coalescing, optimistic vs pessimistic graph selection, eviction, recoloring, live-hole handling, target constraints) where the model supports it.

Core engines:

| Engine | Implemented behavior |
| --- | --- |
| Graph coloring | Interference graph, simplify/select, pessimistic spill, optimistic selection, move coalescing with union-find, conservative/iterated/aggressive choice, bounded recoloring |
| Linear scan | Start-ordered allocation, discontiguous intervals/lifetime holes, register availability, one-victim eviction; heuristic score prioritization for named variants |
| Greedy | Priority ordering, copy preferences, register costs, eviction, neighbor recoloring |
| Exact | Bounded depth-first branch-and-bound with incumbent, constraint domains, register alias masks, copy move penalties, and explicit `optimal` status |
| Simulated annealing | Feasible one-value mutation, Metropolis acceptance, cooling |
| Genetic | Population, tournament-style selection, feasible crossover/mutation, elitism |
| Matching | Exhaustive bounded unique-register bipartite matching for small problems; otherwise greedy fallback |

Utilities include CFG liveness via fixed-point dataflow, live-range construction, live-range splitting, pressure, spill/reload action planning, spill-slot reuse, copy coalescing diagnostics, parallel-copy cycle resolution, callee-save analysis, and physical-register scavenging.

Formulation helpers export a **one-hot SAT feasibility formula** (DIMACS), an SMT-LIB2 Optimize model (including copy costs), a linear mixed-integer programming model in LP format (without copy costs), and an in-memory PBQP-style unary/pairwise cost model. The named `SATRA`, `SMTRA`, `PBQPRA`, `ILPRA`, and `MIPRA` allocators run **the built-in bounded search**, not a SAT/SMT/ILP solver. They expose explicit export helpers for external solvers.

## Limitations that matter

- `Result` assigns **one register or spill slot per value**. Real interval splitting returns a **new** `Problem` with fragments; there is not a built-in program rewriting or register-to-register boundary copy insertion pass. The `IntervalSplittingLS` name is a **splitting-oriented heuristic** until you manually split with `LiveRangeSplitting::run`.
- SSA-specific, dominance-tree, chordal coloring, scheduling-integrated, next-use/Belady, PBQP, dynamic-programming, trace-JIT, GPU, register windows/rotation, and VLIW names are **model-based approximations or policy adapters**, not faithful end-to-end implementations of the corresponding papers/architectures. Details in [`docs/METHODS.md`](docs/METHODS.md).
- There is no machine instruction encoder, CFG-to-SSA converter, global instruction scheduler, model-trained RL policy, architecture-specific ABIs, or automatic reload/store machine IR transformation.
- Values have contiguous register **index** tuples, class/bank constraints, and 64-bit physical register unit masks for aliases/subregisters. No vector lane packing, per-instruction fixed operands, call-clobber insertion, or position-dependent register reassignment.
- The exact engine sets `optimal=true` **only** if the full search finishes before `exact_limit`. With an exhausted limit, it returns its best feasible incumbent and `optimal=false`. `optimal` refers to its explicit weighted model, not total machine-code execution time.
- An empty live-range list models a dead value (no interference); it can still have an assigned register. All segments use `[begin,end)` with **nonnegative instruction points**.

## Quick start

```cpp
#include <regalloc/GeorgeAppelGC.hpp>
#include <regalloc/Model.hpp>
#include <iostream>

int main() {
    auto p = regalloc::makeProblem(2);  // r0 and r1
    regalloc::Value a, b, c;
    a.name = "a"; a.ranges = {{0, 7}}; a.spill_cost = 15;
    b.name = "b"; b.ranges = {{2, 8}}; b.spill_cost = 10;
    c.name = "c"; c.ranges = {{8, 11}}; c.spill_cost = 5;
    p.values = {a, b, c};
    p.copies = {{1, 2, 2.0}};

    auto result = regalloc::GeorgeAppelGC{}.allocate(p);
    std::string error;
    if (!regalloc::validateResult(p, result, &error))
        throw std::runtime_error(error);
    for (std::size_t i = 0; i < result.locations.size(); ++i)
        std::cout << p.values[i].name << " -> " << result.locations[i].reg << "\n";
}
```

Compile as C++20:

```sh
g++ -std=c++20 -O2 -Iinclude examples/showcase.cpp -o showcase
./showcase
```

Alternatively:

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
ctest --test-dir build --output-on-failure
```

## APIs

- `#include <regalloc/RegAlloc.hpp>`: all classes and utilities.
- `#include <regalloc/GeorgeAppelGC.hpp>`: one allocator.
- `regalloc::SomeRA{}.allocate(problem)` and `regalloc::SomeRA::options()`.
- `regalloc::allocate(problem, options)` selects any engine and policy at runtime.
- `regalloc::analyzeLiveness(cfg)` and `regalloc::makeProblemFromCFG(cfg, physical_regs)`.
- `regalloc::ILPRA::model(problem)`, `SATRA::dimacs(problem)`, `SMTRA::smtlib2(problem)`, `PBQPRA::model(problem)`.
- `regalloc::validateResult(problem, allocation, &error)` checks register interference and constraints.

See the full [named method catalogue](docs/METHODS.md), [`examples/showcase.cpp`](examples/showcase.cpp), and [`tests/property.cpp`](tests/property.cpp).