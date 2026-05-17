---
name: embedded-oop-architecture
description: "OOP 5-layer architecture for Cortex-M bare-metal MCU projects (STM32/GD32/APM32). Virtual-table OOP pattern: Interface→Adapter→Device→Board→App. Use when starting a new embedded project, building code framework, or doing multi-platform adaptation."
version: 2.0.0
author: ZHAO Yankun + Hermes
platforms: [universal]
tags: [embedded, mcu, oop, architecture, stm32, gd32, apm32, cortex-m]
metadata:
  hermes:
    tags: [embedded, oop, architecture, stm32, gd32]
    category: embedded
  install:
    hermes: "hermes skills install <url>"
    claude-code: "Tell Claude: 'Install skill from <url> and save to CLAUDE.md'"
    codex: "Tell Codex: 'Read and apply this skill from <url>'"
---

# OOP 5-Layer Architecture (Virtual-Table Pattern)

A C-language OOP architecture for Cortex-M bare-metal projects. Uses **virtual tables** (function pointer tables) to achieve polymorphism and **5-layer separation** to isolate hardware dependencies.

## 5-Layer Model

```
┌────────────────────────────────────────┐
│  App Layer    main.c / *_app.c         │  Business logic, no HW deps
├────────────────────────────────────────┤
│  Board Layer  board_config.h/c         │  Board-level wiring of devices
├────────────────────────────────────────┤
│  Device Layer led.c / uart_comm.c ...  │  Peripheral drivers (via interfaces)
├────────────────────────────────────────┤
│  Adapter Layer stm32_gpio.c / gd32_gpio.c │ Interface→chip-peripheral adapters
├────────────────────────────────────────┤
│  Interface Layer igpio.h / iuart.h ... │  Pure virtual interfaces (headers only)
└────────────────────────────────────────┘
```

**Dependency direction: App → Board → Device → Adapter → Interface (downward only)**

## Naming Conventions

| Rule | Example |
|------|---------|
| Interface files: `i` prefix | `igpio.h`, `iuart.h`, `ii2c.h` |
| Adapter files: `chip_peripheral` | `stm32_gpio.c`, `gd32_gpio.c` |
| App layer: `_app` suffix | `motor_app.c`, `sensor_app.c` |
| Interface types: `_if_t` | `gpio_if_t`, `uart_if_t` |
| Operations tables: `_ops_t` | `gpio_ops_t`, `uart_ops_t` |
| Device types: `_dev_t` | `led_dev_t`, `motor_dev_t` |
| Doxygen comments (Chinese OK) | `/** @brief Initialize GPIO */` |
| Defensive null checks | `if (p_dev == NULL \|\| p_dev->ops == NULL) return;` |
| Explicit unsigned types | `0u`, `1u`, `0x00u` |
| Static instances + poll loop | No RTOS: all `static`, main loop polls each layer |

## Platform Isolation

`Platformdefine.h` — the only file that knows the chip:

```c
/** @brief Platform selection (change only this macro) */
#define PLATFORM_STM32  1
#define PLATFORM_GD32   2
#define PLATFORM_APM32  3

#define CURRENT_PLATFORM  PLATFORM_STM32

#if CURRENT_PLATFORM == PLATFORM_STM32
  #include "stm32f4xx_hal.h"
  #define PLATFORM_NAME "STM32_HAL"
#elif CURRENT_PLATFORM == PLATFORM_GD32
  #include "gd32f4xx.h"
  #define PLATFORM_NAME "GD32_STD"
#elif CURRENT_PLATFORM == PLATFORM_APM32
  #include "apm32f4xx.h"
  #define PLATFORM_NAME "APM32"
#endif
```

Directory layout (compile-time isolation, not #ifdef):

```
src/
├── platform/
│   ├── stm32_code/    ← STM32 adapter implementations
│   └── gd32_code/     ← GD32 adapter implementations
├── interface/         ← Interface layer (platform-independent)
├── device/            ← Device layer (platform-independent)
├── board/             ← Board layer (platform-independent)
└── app/               ← Application layer (platform-independent)
```

## Complete Skeleton

### 1. Interface Layer — `igpio.h`

```c
/**
 * @file    igpio.h
 * @brief   GPIO interface (pure virtual, no platform dependency)
 */
#ifndef IGPIO_H
#define IGPIO_H

#include <stdint.h>
#include <stdbool.h>

typedef enum {
    GPIO_LOW  = 0u,
    GPIO_HIGH = 1u
} gpio_level_t;

typedef struct {
    void (*write)(uint16_t pin, gpio_level_t level);
    gpio_level_t (*read)(uint16_t pin);
    void (*toggle)(uint16_t pin);
} gpio_ops_t;

typedef struct {
    const gpio_ops_t *ops;
} gpio_if_t;

#endif /* IGPIO_H */
```

### 2. Adapter Layer — `stm32_gpio.c`

```c
/**
 * @file    stm32_gpio.c
 * @brief   STM32 GPIO adapter (implements igpio.h vtable)
 */
#include "Platformdefine.h"
#include "igpio.h"

static GPIO_TypeDef *const gpio_ports[] = {
    GPIOA, GPIOB, GPIOC, GPIOD, GPIOE, GPIOF, GPIOG, GPIOH
};

static uint16_t extract_pin(uint16_t pin) {
    return (uint16_t)(1u << (pin & 0x0Fu));
}
static uint8_t extract_port(uint16_t pin) {
    return (uint8_t)((pin >> 8u) & 0x0Fu);
}

static void stm32_gpio_write(uint16_t pin, gpio_level_t level) {
    GPIO_TypeDef *port = gpio_ports[extract_port(pin)];
    HAL_GPIO_WritePin(port, extract_pin(pin),
        (level == GPIO_HIGH) ? GPIO_PIN_SET : GPIO_PIN_RESET);
}
static gpio_level_t stm32_gpio_read(uint16_t pin) {
    GPIO_TypeDef *port = gpio_ports[extract_port(pin)];
    return (HAL_GPIO_ReadPin(port, extract_pin(pin)) == GPIO_PIN_SET)
        ? GPIO_HIGH : GPIO_LOW;
}
static void stm32_gpio_toggle(uint16_t pin) {
    GPIO_TypeDef *port = gpio_ports[extract_port(pin)];
    HAL_GPIO_TogglePin(port, extract_pin(pin));
}

static const gpio_ops_t stm32_gpio_ops = {
    .write  = stm32_gpio_write,
    .read   = stm32_gpio_read,
    .toggle = stm32_gpio_toggle
};

const gpio_if_t gpio_adapter = { .ops = &stm32_gpio_ops };
```

### 3. Device Layer — `led.c`

```c
/**
 * @file    led.c
 * @brief   LED device driver (operates via gpio_if_t, no chip dependency)
 */
#include "led.h"
#include "igpio.h"

typedef struct {
    const gpio_if_t *p_gpio;
    uint16_t pin;
    bool is_active_low;
} led_dev_t;

static led_dev_t g_led;

void led_init(const gpio_if_t *p_gpio, uint16_t pin, bool active_low) {
    if (p_gpio == NULL) return;
    g_led.p_gpio = p_gpio;
    g_led.pin = pin;
    g_led.is_active_low = active_low;
}

void led_on(void) {
    gpio_level_t level = g_led.is_active_low ? GPIO_LOW : GPIO_HIGH;
    g_led.p_gpio->ops->write(g_led.pin, level);
}
void led_off(void) {
    gpio_level_t level = g_led.is_active_low ? GPIO_HIGH : GPIO_LOW;
    g_led.p_gpio->ops->write(g_led.pin, level);
}
void led_toggle(void) {
    g_led.p_gpio->ops->toggle(g_led.pin);
}
```

### 4. Board Layer — `board_config.c`

```c
/**
 * @file    board_config.c
 * @brief   Board config: create and wire all device instances
 */
#include "Platformdefine.h"
#include "igpio.h"
#include "iuart.h"
#include "led.h"
#include "uart_comm.h"

extern const gpio_if_t gpio_adapter;
extern const uart_if_t uart1_adapter;

void board_init(void) {
    led_init(&gpio_adapter, 0x0205u, true);   /* PE5, active-low */
    uart_comm_init(&uart1_adapter, 115200u);
}
```

### 5. App Layer — `main.c`

```c
/**
 * @file    main.c
 * @brief   Application entry (poll-loop scheduling, no RTOS)
 */
#include "Platformdefine.h"
#include "board_config.h"
#include "led.h"

int main(void) {
    HAL_Init();
    SystemClock_Config();
    board_init();

    while (1u) {
        led_toggle();
        HAL_Delay(500u);
    }
}
```

## Dependency Injection Flow

```
board_config.c
  ├── holds extern adapter references (extern gpio_adapter)
  └── calls led_init(&gpio_adapter, ...)
       └── led_dev_t stores gpio_if_t* pointer

led.c
  └── calls via g_led.p_gpio->ops->write(...)
       └── resolves at runtime to stm32_gpio_write or gd32_gpio_write
```

**Key: Device layer only depends on `igpio.h` interface — it never knows the chip model.**

## Switching Platforms (3 steps)

1. Change `CURRENT_PLATFORM` macro in `Platformdefine.h`
2. Swap the `platform/*_code/` directory in your build
3. Recompile — **zero changes to Device/Board/App layer code**

## Pitfalls

| Pitfall | Prevention |
|---------|------------|
| Calling vtable before `init()` | Device `init()` checks NULL first |
| Adapter exposes raw HAL handles | Wrap as `_if_t`, never expose `GPIO_TypeDef*` |
| Device layer includes chip headers | Only include Interface layer `.h` files |
| Globals scattered across layers | Inject via `board_config`, no loose `extern` |

## Install Commands

**Hermes:**
```bash
hermes skills install https://raw.githubusercontent.com/<user>/<repo>/main/skills/embedded-oop-architecture/SKILL.md
```

**Claude Code:** tell Claude —
> "Install this skill: fetch and read https://raw.githubusercontent.com/.../SKILL.md, then save its rules as CLAUDE.md in my embedded project root."

**Codex:** tell Codex —
> "Read and apply this skill from https://raw.githubusercontent.com/.../SKILL.md — use this architecture for all embedded code in this project."
