# Embedded Skills for AI Agents

Reusable embedded C architecture skills compatible with **Hermes**, **Claude Code**, and **Codex CLI**.

## Skills

### 1. OOP 5-Layer Architecture (`embedded-oop-architecture`)

Virtual-table OOP pattern for Cortex-M bare-metal projects (STM32 / GD32 / APM32).

```
Interface → Adapter → Device → Board → App
```

- Polymorphism via `_ops_t` function pointer tables
- `Platformdefine.h` for chip isolation
- Dependency injection through `board_config`
- Complete `.h` / `.c` skeleton templates
- Switch platforms (STM32 ↔ GD32) without touching Device/Board/App code

### 2. Function Pointer Registration (`embedded-fnptr-register`)

Lightweight callback-registration pattern for event-driven embedded C.

- Single callback, multi-callback registry, and `void* ctx` patterns
- UART RX / ADC done / key matrix / timer soft-scheduler examples
- ISR-safe design with main-loop deferral

---

## Install

### Hermes

```bash
hermes skills install https://raw.githubusercontent.com/gsiliconk/mbedded-skills/main/skills/embedded-oop-architecture/SKILL.md
hermes skills install https://raw.githubusercontent.com/gsiliconk/mbedded-skills/main/skills/embedded-fnptr-register/SKILL.md
```

### Claude Code

Tell Claude:

> "Install these skills from https://github.com/gsiliconk/mbedded-skills — save them as project rules."

### Codex CLI

Tell Codex:

> "Read and apply the skills from https://github.com/gsiliconk/mbedded-skills"

---

## Structure

```
embedded-skills/
├── README.md
└── skills/
    ├── embedded-oop-architecture/
    │   └── SKILL.md
    └── embedded-fnptr-register/
        └── SKILL.md
```

## Author

**ZHAO Yankun (赵炎坤)** — [@gsiliconk](https://github.com/gsiliconk)

Junior embedded software engineer. STM32 / GD32 / APM32. National Undergraduate Electronics Design Contest provincial first prize.

## License

MIT
