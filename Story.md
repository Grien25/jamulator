### Statically Recompiling NES Games with LLVM and Go — Deep Summary

This story summarizes the full `index.html` article in this repo. It walks through the problem Andrew Kelley explored: statically disassembling and recompiling NES ROMs into native executables using LLVM IR and Go, the engineering path taken (assembler → disassembler → LLVM-based codegen → runtime), the technical challenges (runtime address checks, parallel subsystems, interrupts, jump tables, indirect jumps, “dirty” assembly tricks), the pragmatic compromises (a small C runtime and eventually an interpreter fallback), and the final takeaway: static recompilation is instructive but impractical compared to JIT for real games.

### Big Idea and Motivation
- **Goal**: Turn NES games (6502 machine code) into optimized native binaries by translating to LLVM IR, letting LLVM do optimization and codegen.
- **Why LLVM**: Universal IR, broad targets, great diagnostics (via `clang`), and strong optimization passes.
- **Scope**: Explore what’s possible statically, avoiding full emulation if feasible.

### NES/6502 Crash Course (essentials used in the project)
- **Memory map** (simplified):
  - `0000–07ff`: 2KB WRAM
  - `2000–2007`: PPU registers (graphics)
  - `4000–4015`: APU registers (audio)
  - `4016–4017`: Controller I/O
  - `8000–ffff`: PRG ROM (game code)
- **Entry vectors** at `fffa–ffff` (little endian): NMI, Reset, IRQ.
- **Registers**: `A`, `X`, `Y` (8-bit); `P` (status bits: carry/zero/interrupt/decimal/brk/overflow/negative); `SP` (stack pointer); `PC` (16-bit).

### Custom ABI for a Simple “Hello, World”
- To simplify testing, the article invents a tiny ABI extension:
  - Write to `$2008` → putchar
  - Write to `$2009` → exit(code)
- A minimal 6502 sample prints “Hello, World!” and exits.

### Assembler: Lexer/Parser → AST → Binary
- **Tools**: Go `nex` (lexer) + `go tool yacc` (parser) to build an assembler for a subset of 6502.
- **AST model**: `LabelStatement`, `Instruction`, `DataStatement` (`.db`, `.dw`), `OrgPseudoOp`, etc.
- **Build**: Makefile generates the Go sources from grammar and lex rules, then builds the tool.
- **Assembling**: Multi-pass over AST to compute offsets, resolve labels, and emit the 32KB PRG payload.

### Disassembler: From Bytes Back to Structured Assembly
- Start as `.db` bytes everywhere, then iteratively “promote” bytes into instructions by tracing control flow from the 3 vectors (NMI/Reset/IRQ).
- **Control-flow marking**: Based on opcodes, mark next instruction(s) and branch destinations as code; others left as data.
- **Readability passes**:
  - Detect ASCII runs → join as quoted strings.
  - Detect long runs of filler bytes → replace with `.org` gaps.
  - Collapse adjacent `.db` into fewer statements.
- Result: A close facsimile of source, with sensible labels for vectors and discovered addresses.

### Code Generation: 6502 → LLVM IR (via Go + gollvm)
- **Plan**:
  1) Create an LLVM module; declare `putchar` and `exit`.
  2) Build `main` and allocate stack “register” variables (`X`, `Y`, `A`, status bits).
  3) Pass 1: identify labeled data, create LLVM globals.
  4) Pass 2: create basic blocks for labels and map names → blocks.
  5) Pass 3: emit IR per instruction into proper blocks.
- **Examples**:
  - Immediate `LDX #$00`: store to `%X`, set/clear `S_zero`/`S_neg`.
  - Indexed `LDA label, X`: GEP into global data, load into `%A`, update status bits dynamically.
- **Verification and building**: Emit LLVM bitcode, run `llc -filetype=obj`, and link with `gcc`.

### Optimization Passes
- Applied before writing bitcode: constant propagation, instruction combining, promote memory to register (mem2reg), GVN, CFG simplify.
- Outcome: Large simplifications while preserving behavior; confirms LLVM can optimize the naive register-variable approach effectively.

### NES ROM Layout and Metadata
- **.nes format**: 16-byte header (mapper, mirroring, etc.), PRG ROM (code), CHR ROM (graphics for PPU).
- The tool extracts CHR data and mirroring, exports as private LLVM globals (`rom_chr_data`, `rom_mirroring`) for use by the runtime.

### First Real ROM: Zelda Title Screen Simulator
- Disassembly shows only a handful of new instructions; the NMI is stubbed to `RTI` to punt on interrupts initially.
- Result: Mostly works, but a bent sword shows timing/interrupt issues remain.

### Challenge 1 — Runtime Address Checks (Indirect Stores)
- Instruction like `STA ($00), Y` computes a runtime address; could target WRAM or memory-mapped I/O like PPU/APU.
- Solution: Emit IR that calculates the 16-bit address at runtime and funnels the write through `dynStore(addr, val)`.
- `dynStore` performs address range checks and dispatches appropriately:
  - `addr < 0x2000` → WRAM (with mirroring mask)
  - `0x2000 ≤ addr < 0x4000` → PPU registers (mask low 3 bits) and call out to PPU hooks
  - Otherwise → panic (or future handling)
- Similar helpers exist for compile-time-known loads (`loadWord`, `load`) and for PPU reads.

### Challenge 2 — Parallel Systems (CPU vs PPU vs APU)
- CPU, PPU, and APU run in parallel at fixed ratios (PPU ≈ 3× CPU cycles).
- Static recompilation can’t eliminate all emulation: one subsystem must be emulated.
- **Approach**: After each translated CPU instruction, call `rom_cycle(cycles)` in a small C runtime that steps the PPU 3× cycles and manages SDL/OpenGL rendering.
- The generated module links against a bundled runtime (`runtime/`) ported from Fergulator’s PPU and a thin front end for rendering.
- The runtime contract is defined in `rom.h` (e.g., `rom_start`, `rom_cycle`, `rom_ppu_*`, CHR read/mirroring exports).

### Challenge 3 — Interrupts (NMI)
- NMI during vblank can occur between any two instructions.
- A naive “check-after-every-instruction” harms code quality and still leaves stack semantics unresolved.
- **Chosen design**:
  - Promote CPU “registers” to globals in the generated module.
  - Change `rom_start` to take an interrupt selector.
  - Runtime calls `rom_start(RESET)` initially; later, when PPU signals vblank, runtime calls `rom_start(NMI)` from inside `rom_cycle`.
  - At the top of the NMI block, generated code pushes `PC` and status to the 6502 stack via helper routines; `RTI` becomes a natural return from `rom_start` by emitting a native `ret` after restoring status and ticking cycles.
- Caveats: If a game abuses `RTI` side effects or fakes returns without `RTI`, this scheme breaks.

### Challenge 4 — Detecting Jump Tables
- SMB uses a dynamic jump table idiom: a `JSR` to a routine that pulls the return address, indexes a table of `.dw` (16-bit) addresses following the `JSR`, and `JMP ($0006)` to the chosen target.
- Disassembler adds a state machine (`detectJumpTable`) to recognize the ASL/TAY/PLA/STA/…/JMP pattern and reinterpret the data after the `JSR` as `.dw` label targets, enabling further disassembly and codegen.

### Challenge 5 — Indirect Jumps
- `JMP (addr)` targets are only known at runtime.
- **Dynamic jump table in IR**: Generate a `switch` on the current `PC` to known label blocks; default initially was a “panic.”
- Later, this default becomes the entry to an interpreter (see next section) to handle non-label targets safely.

### Challenge 6 — Dirty Assembly Tricks
- SMB packs bytes aggressively:
  - Injecting an opcode byte (e.g., `$2c` for `BIT abs`) to “sabotage” the next instruction when entering at a particular label.
  - Jumping into the middle of instructions.
  - Synthesizing control flow by pushing addresses and using `RTS` as a jump.
  - Indirect jumps landing in RAM or non-disassembled regions.
- These invalidate strict static assumptions about code-vs-data and control flow.

### Pragmatic Pivot — Embed an Interpreter Fallback
- To guarantee correctness under all above tricks:
  - Include the full PRG ROM as data.
  - After each generated instruction, update the global `PC`.
  - Add an `Interpret` basic block that reads the opcode at `PC`, executes it (small interpreter), and then jumps to the dynamic jump table.
  - Change `DynJumpTable` default case to go to `Interpret` instead of panicking.
  - If code generation “runs into data,” branch to `Interpret`.
- Performance cost: global state must be accurate before each `rom_cycle` call (interrupts can re-enter), defeating many optimizations. But correctness is preserved, and only ~6 opcodes needed interpreting to get SMB1 playable.

### Results
- Zelda Title Screen: works natively with PPU runtime; fixing NMI handling straightened the sword (timing correctness).
- Super Mario Bros. 1: playable via mostly native code with interpreter fallback for edge cases.
- A short video demonstrates SMB1 running in the recompiled executable and a playback feature used for debugging.

### Mappers (Unsolved Here)
- Cartridge mappers add bank switching and more complex memory behavior, making complete static disassembly even harder.
- The article argues this space clearly favors JIT over static recompilation.

### Conclusion and Takeaways
- **Educational win**: Built a working pipeline, learned NES architecture, LLVM IR/codegen, and compiler/interpreter trade-offs.
- **Practical verdict**: Static recompilation of NES games is not viable for production—parallel subsystems, interrupts, ambiguous code/data, dynamic control flow, and mapper complexity force compromises that erode optimization benefits.
- **Better approach**: JIT with speculative assumptions, deoptimization, and runtime profiling fits the problem domain far better. Keep emulator and ROM separated for distribution/legal clarity.

### Key Components and Terms (as seen in the code/article)
- Go assembler/disassembler: AST types, `Parse`, `ast.Print`, passes to detect ASCII and `.org`, control-flow discovery.
- Codegen structures: `Compilation` with fields like `mod`, `builder`, `labeledData`, `labeledBlocks`, register/value slots, and helper methods (`testAndSetZero/Neg`, `dynTestAndSetZero/Neg`).
- Runtime dispatch: `dynStore(addr, val)`, `loadWord`, `load(addr)`, PPU hooks like `rom_ppu_write_*`, and read functions.
- Runtime C stubs: `rom_cycle(uint8_t cycles)`, `rom_start(uint8_t vector)`, PPU stepping, SDL/OpenGL rendering; `rom.h` with mirroring and CHR APIs.
- Interrupts handling: pushing/pulling status/PC onto the 6502 stack via helpers (`pushWordToStack`, `pushToStack`, `pullStatusReg`), `RTI` lowering to `ret`.
- Disassembly enhancements: `detectJumpTable` state machine, label/block creation, dynamic jump table (`DynJumpTable`) for indirect `JMP`.
- Interpreter fallback: `Interpret` block that decodes PRG bytes at runtime; `DynJumpTable` default branches to it.

### Why This Matters
The project shows how far you can push static techniques with strong IR tooling and careful analysis, and exactly where the boundary lies. It’s a concise tour of compiler construction, IR generation, and emulator realities—ending with a principled pivot to hybrid native+interpretation where correctness trumps theoretical optimization.

