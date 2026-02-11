# Research: GitHub Copilot Best Practices for STM32 Embedded Engineering

> Compiled February 2026 — based on GitHub official documentation, community projects, and embedded-specific workshops.

---

## Part 1: Universal Best Practices (What Top Users Do)

### 1.1 Context Is Everything — The #1 Success Factor

The single most impactful practice is **providing rich, relevant context**. Copilot is an LLM — its output quality is directly proportional to the quality and relevance of the context it receives.

**Key tactics:**

- **Open relevant files** in your editor — Copilot uses open tabs as context (via "neighboring tabs" technique). Keep HAL driver headers, your current peripheral `.c` file, and related config files open simultaneously.
- **Close irrelevant files** — Noise degrades output. When switching from UART work to SPI work, close the UART files.
- **Use top-level comments** — A file-level comment describing the module's purpose gives Copilot a "north star" for all suggestions in that file.
- **Set `#include` statements first** — Manually add your HAL includes (`#include "stm32f4xx_hal.h"`, `#include "stm32f4xx_hal_uart.h"`) before asking Copilot to generate code. This tells it which peripherals/APIs to target.

### 1.2 Custom Instructions — The Biggest Multiplier

GitHub Copilot supports a **tiered instruction system** (as of 2025-2026):

| Level | File | When Loaded | Best For |
|-------|------|-------------|----------|
| **Repository-wide** | `.github/copilot-instructions.md` | Every interaction | Universal project rules, coding standards, project overview |
| **Path-specific** | `.github/instructions/*.instructions.md` | When editing matching files | Language/domain-specific rules (C vs Python) |
| **Agent instructions** | `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` | Agent workflows | Build commands, project layout, CI details |
| **Prompt files** | `.github/prompts/*.prompt.md` | Explicitly invoked only | Reusable workflows (PR review, test generation) |
| **Personal** | GitHub.com settings | Always (highest priority) | Individual preferences |
| **Organization** | Org settings | Always (lowest priority) | Company-wide standards |

**The "Secret Weapon" pattern** (from the `reubenjohn/agentic-ide-power-user` workshop for embedded engineers):

- Keep **global instructions minimal** (200–500 lines max) — they're loaded into EVERY interaction
- Put **language-specific rules** in path-specific files (e.g., `c-firmware.instructions.md` with `applyTo: "**/*.c,**/*.h"`)
- Put **target-specific rules** in separate files (e.g., `stm32f4.instructions.md`)
- Use **prompt files** for repeatable workflows (code review, driver scaffolding)

### 1.3 Context Rot — The Hidden Enemy

Research shows that LLM performance **degrades as the context window fills up** (20–50% accuracy drop from 10K to 100K+ tokens). This is called "context rot."

**The Burke Holland Equation:**

```
Performance ∝ Signal² / (Noise × Size)
```

**Mitigations:**

- Start **fresh chat sessions** frequently — don't let conversations get long and stale.
- Use the **Research → Plan → Implement** loop: research in one session, create a plan document, then implement in a fresh session with only the plan as context.
- **Delete irrelevant questions** from chat history.
- **Disable unused MCP servers** — each adds 200–500 tokens of overhead to every interaction.
- Use `@workspace` sparingly — powerful but adds context bulk.

### 1.4 Prompting Strategies

**Start general, then get specific:**

```c
// Configure TIM2 for PWM output on PA5
// Requirements:
// - 1kHz PWM frequency at 168MHz system clock
// - Variable duty cycle 0-100%
// - Use HAL_TIM_PWM_Start()
```

**Break complex tasks into steps** — don't ask Copilot to generate an entire driver at once. Ask for:

1. Initialization struct setup
2. Peripheral init function
3. Interrupt handler
4. Higher-level API functions

**Give examples** — show one complete function, then let Copilot follow the pattern for similar functions.

**Use meaningful names** — `UART_TransmitSensorData()` >> `sendData()`. Copilot infers intent from names.

### 1.5 Persona Prompting

GitHub's official docs recommend telling Copilot to act as a specific persona:

> "You are a Senior C++ Developer who cares greatly about code quality, readability, and efficiency."

For STM32 work, this is extremely powerful — telling Copilot it's an embedded systems expert prevents it from suggesting desktop-oriented patterns.

---

## Part 2: STM32-Specific Strategies (From Real-World Projects)

### 2.1 What Successful STM32 Projects Do

**[Strooom/MuMo-v2-Node-SW](https://github.com/Strooom/MuMo-v2-Node-SW)** (STM32WLE5, LoRaWAN environmental monitor):

- Detailed `CLAUDE.md` with complete build commands, test commands, code architecture
- Documents state machine architecture, event-driven design patterns
- Specifies dual AES implementation (hardware vs software) for testing
- Explains conditional compilation (`#ifndef generic`) for native vs embedded testing
- **Key insight**: Documents the HAL wrapper pattern — thin C++ wrappers around STM32 HAL (I2C, SPI, UART, ADC, RTC) that maintain HAL compatibility while adding type safety

**[embedded-pro/e-foc](https://github.com/embedded-pro/e-foc)** (FOC motor control, STM32F407):

- Comprehensive `.github/copilot-instructions.md` with:
  - Hard constraints: NO heap, NO dynamic STL containers, NO recursion in ISRs
  - Specific build/test commands with exact CMake presets
  - Performance budgets: "FOC loop must complete in <400 cycles at 120 MHz for 20 kHz control rate"
  - Pattern guidance: "implement float first, then Q15/Q31 variants"
  - Code review checklist: check for public functions that could be private, magic numbers, const correctness

**[ivrabie/r4motor_firmware](https://github.com/ivrabie/r4motor_firmware)** (STM32F411, Embassy/Rust motor controller):

- `AGENTS.md` with exact build/flash/lint commands
- Detailed style conventions (naming, types, representation, ownership)
- Documents async patterns, concurrency rules, protocol specifics
- Maps every source file to its responsibility

**[reubenjohn/agentic-ide-power-user](https://github.com/reubenjohn/agentic-ide-power-user)** — Workshop targeting embedded/firmware engineers:

- Introduces the **Signal²/(Noise × Size)** equation for context quality
- Documents the full Copilot context assembly pipeline (system prompt → user prompt → history)
- Demonstrates the **Global → Specialized → Prompt file** instruction spectrum
- Shows target-specific instruction files per MCU family to avoid cross-contamination

### 2.2 What Copilot Excels At for STM32

| Task | Quality | Tips |
|------|---------|------|
| **Generating HAL init code** | ★★★★★ | Provide target MCU + peripheral + pin assignments in comments |
| **Writing repetitive register configs** | ★★★★★ | Show one example, it'll pattern-match the rest |
| **Generating DMA configuration** | ★★★★☆ | Specify stream/channel/direction explicitly |
| **Writing ISR handlers** | ★★★★☆ | Comment the interrupt source and expected behavior |
| **Explaining HAL source code** | ★★★★★ | Use `/explain` on HAL functions you don't understand |
| **Unit test generation** | ★★★★☆ | Works great for logic code; needs mocks for HAL-dependent code |
| **Debugging register configs** | ★★★☆☆ | Better when Reference Manual section is cited in comments |
| **Complex timing/clock calculations** | ★★★☆☆ | Always verify calculations manually |
| **Low-power mode configuration** | ★★☆☆☆ | Very chip-specific; provide detailed constraints |
| **Linker script modifications** | ★★☆☆☆ | Keep linker scripts open as context |

### 2.3 Common Pitfalls to Avoid

1. **Trusting HAL version blindly** — Copilot may generate code for a different HAL version. Always set your includes first and specify the version in instructions.
2. **Cross-MCU contamination** — Copilot may mix register names from STM32F4 and STM32H7. Target-specific instruction files prevent this.
3. **Desktop patterns in embedded code** — Without persona/instructions, Copilot may suggest `printf()`, dynamic allocation, or C++ exceptions.
4. **Ignoring return values** — Copilot sometimes skips HAL return value checks. Instructions should mandate this.
5. **DMA/interrupt conflicts** — Copilot doesn't know your full system allocation. Document peripheral/DMA/interrupt assignments in instructions.

---

## Part 3: Advanced Techniques

### 3.1 The Research → Plan → Implement Loop

For complex features (e.g., adding a new communication protocol):

1. **Research session**: Ask Copilot to analyze the codebase, understand existing driver patterns, create a summary document
2. **Plan session**: Start fresh, load the summary, create a step-by-step implementation plan
3. **Implement session**: Start fresh again, load only the plan + relevant files, implement step by step

This combats context rot and keeps each session focused.

### 3.2 Automated Instruction Generation

GitHub supports having the **Copilot coding agent auto-generate your `copilot-instructions.md`**. Navigate to `github.com/copilot/agents`, select your repo, and paste the onboarding prompt from the [official docs](https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot). The agent will inventory your codebase, document build steps, map project structure, test commands, and create a comprehensive instructions file as a PR.

### 3.3 Use Copilot to Generate Test Mocks for HAL

One of the highest-value patterns: ask Copilot to generate mock/stub layers for HAL functions so you can run unit tests on your host machine:

```c
// Generate mock implementations for STM32 HAL UART functions
// These should compile on a desktop (x86) target
// Mock HAL_UART_Transmit to record data in a test buffer
// Mock HAL_UART_Receive to return predefined test data
```

The `Strooom/MuMo-v2-Node-SW` project demonstrates this pattern with software AES vs hardware AES implementations for testability.

### 3.4 IDE Workflow Tips

**VS Code + STM32:**

1. **Open files strategically**: When working on UART driver, keep open: your `uart.c`, `uart.h`, `stm32f4xx_hal_uart.h`, `main.c`, and your `copilot-instructions.md`
2. **Use `@workspace`** to ask architectural questions: "What peripherals are initialized in this project?"
3. **Use inline chat** (Cmd+I / Ctrl+I) for quick fixes: highlight a HAL init struct and ask "Configure this for 9600 baud, 8N1"
4. **Use `/explain`** on HAL callback functions you don't understand
5. **Use `/fix`** when HAL returns errors — highlight the error-producing code
6. **Highlight code before chatting** — always select the relevant code block before asking Copilot Chat about it

---

## Part 4: Key Sources & References

| Source | URL | Focus |
|--------|-----|-------|
| GitHub Official: Best Practices | https://docs.github.com/en/copilot/using-github-copilot/best-practices-for-using-github-copilot | General best practices |
| GitHub Official: Prompt Engineering | https://docs.github.com/en/copilot/using-github-copilot/copilot-chat/prompt-engineering-for-copilot-chat | Prompt strategies |
| GitHub Official: Custom Instructions | https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot | Instruction file setup |
| GitHub Official: Instructions Support | https://docs.github.com/en/copilot/reference/custom-instructions-support | What works where |
| GitHub Official: Response Customization | https://docs.github.com/en/copilot/concepts/prompting/response-customization | Instruction types & precedence |
| GitHub Blog: IDE Tips & Tricks | https://github.blog/developer-skills/github/how-to-use-github-copilot-in-your-ide-tips-tricks-and-best-practices/ | IDE workflow optimization |
| GitHub Blog: Prompt Crafting | https://github.blog/developer-skills/github/how-to-write-better-prompts-for-github-copilot/ | Prompt engineering guide |
| Workshop: Agentic IDE Power User | https://github.com/reubenjohn/agentic-ide-power-user | Embedded-specific workshop |
| Real Project: MuMo STM32WLE5 | https://github.com/Strooom/MuMo-v2-Node-SW | Production STM32 firmware with AI agent instructions |
| Real Project: e-foc Motor Control | https://github.com/embedded-pro/e-foc | FOC motor control with copilot-instructions.md |
| Real Project: r4motor Firmware | https://github.com/ivrabie/r4motor_firmware | Embedded on STM32F411 with AGENTS.md |
