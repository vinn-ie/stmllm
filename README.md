# stmllm — STM32 LLM Instructions

A curated set of GitHub Copilot custom instructions, agent configurations, and reusable prompts designed to maximize AI-assisted productivity for **STM32 embedded C/C++ development**.

## Why This Exists

AI coding assistants like GitHub Copilot are powerful — but their output quality for embedded firmware depends heavily on the context and instructions you provide. Without guidance, Copilot will suggest desktop-oriented patterns (`malloc`, `printf` in ISRs, `std::vector`) that are dangerous in embedded systems.

This repository provides a production-ready instruction set that teaches Copilot to think like a senior embedded engineer: safe, deterministic, hardware-aware code that follows industry best practices.

## What's Inside

```
.github/
├── copilot-instructions.md                  # Global rules — loaded into every Copilot interaction
└── instructions/
│   ├── c-firmware.instructions.md           # C-specific rules (HAL patterns, MISRA-C, ISR safety)
│   ├── cpp-firmware.instructions.md         # C++ rules (no-exceptions, RAII, static polymorphism)
│   └── stm32-target.instructions.md         # Target MCU config (clocks, pins, DMA, errata)
└── prompts/
    ├── review-firmware.prompt.md            # Structured firmware code review workflow
    ├── scaffold-driver.prompt.md            # Generate complete peripheral drivers from spec
    └── generate-tests.prompt.md             # Generate unit tests with HAL mock layer
AGENTS.md                                    # Agent instructions (Copilot coding agent, Claude Code, etc.)
docs/
└── copilot-research.md                      # Full research findings and references
```

## Quick Start

### 1. Copy into your STM32 project

```bash
# From your STM32 project root
cp -r path/to/stmllm/.github .
cp path/to/stmllm/AGENTS.md .
```

### 2. Customize the target-specific file

Edit `.github/instructions/stm32-target.instructions.md` — replace the example values with your actual:

- MCU part number and silicon revision
- Clock configuration (HSE, PLL, bus prescalers)
- Peripheral-to-pin assignments
- DMA stream/channel allocation
- Known errata and workarounds
- Build and flash commands

### 3. Start using Copilot

The instructions are automatically picked up:

| Copilot Feature | What Gets Loaded |
|----------------|------------------|
| **Code completion** (inline) | `copilot-instructions.md` + matching path-specific files |
| **Copilot Chat** | `copilot-instructions.md` + path-specific + open files as context |
| **Copilot coding agent** | All instruction files + `AGENTS.md` |
| **Copilot code review** | `copilot-instructions.md` + path-specific |

### 4. Use the prompt files

In VS Code Copilot Chat, invoke prompts explicitly:

- **Code review**: Use the `review-firmware` prompt on your changed files
- **New driver**: Use `scaffold-driver` with your peripheral specification
- **Unit tests**: Use `generate-tests` for a module

## How It Works

### Layered Context Architecture

The instruction files are designed around the **Signal²/(Noise × Size)** principle — maximizing relevant context while minimizing overhead:

```
copilot-instructions.md          ← Always loaded (~50 lines) — persona + hard constraints
  │
  ├─ c-firmware.instructions.md  ← Loaded for *.c, *.h — HAL patterns, MISRA, ISR safety
  ├─ cpp-firmware.instructions.md← Loaded for *.cpp, *.hpp — C++ embedded rules
  └─ stm32-target.instructions.md← Loaded for source/build files — MCU-specific details
```

When editing a `.c` file, Copilot sees the global rules + C rules + target rules. When editing a Python test script, it only sees the global rules. This prevents context rot — the degradation of AI output quality as irrelevant context accumulates.

### Persona-Driven

The instructions establish a **senior embedded engineer persona** that:

- Thinks about timing, power, and hardware constraints
- Defaults to defensive, safe coding patterns
- Cites Reference Manual sections and errata
- Never suggests patterns that are unsafe in firmware (dynamic alloc, blocking ISRs, desktop idioms)

## Key Principles Encoded

| Principle | Implementation |
|-----------|---------------|
| No dynamic allocation | Enforced in global + C + C++ instructions |
| ISR safety | Volatile, critical sections, timing budgets, no blocking |
| HAL correctness | Return value checking, proper init sequences, clock enables |
| MISRA-C compliance | Key rules in C instructions (static, const, explicit casts, switch defaults) |
| Deterministic execution | No recursion in ISR paths, bounded loops, static memory |
| Hardware awareness | Target file documents exact MCU, clocks, pins, errata |
| Testability | Mock-based HAL testing strategy in test generation prompt |

## Supported Environments

| Environment | Instructions | Path-Specific | Agent | Prompts |
|------------|:-----------:|:------------:|:-----:|:-------:|
| VS Code | ✅ | ✅ | ✅ | ✅ |
| JetBrains (CLion) | ✅ | ✅ | ✅ | — |
| STM32CubeIDE (Eclipse) | ✅ | — | ✅ | — |
| Visual Studio | ✅ | ✅ | — | — |
| GitHub.com Chat | ✅ | — | — | — |
| Copilot CLI | ✅ | ✅ | ✅ | — |

