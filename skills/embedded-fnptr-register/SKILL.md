---
name: embedded-fnptr-register
description: "Function pointer registration pattern (wlc_lib style) for embedded C projects. Decouple modules by registering callbacks: UART RX, ADC done, key events, timer soft-scheduling. Use when building event-driven embedded firmware without an RTOS."
version: 2.0.0
author: ZHAO Yankun + Hermes
platforms: [universal]
tags: [embedded, mcu, callback, function-pointer, decoupling, event-driven]
metadata:
  hermes:
    tags: [embedded, callback, architecture, stm32, gd32]
    category: embedded
  install:
    hermes: "hermes skills install <url>"
    claude-code: "Tell Claude: 'Install skill from <url> and save to CLAUDE.md'"
    codex: "Tell Codex: 'Read and apply this skill from <url>'"
---

# Function Pointer Registration Pattern

Decouple embedded C modules by **registering function pointers**. Module A exposes a registration API; Module B registers its handler. When an event fires, A calls B through the pointer — neither side depends on the other's internals.

## Core Patterns

### Pattern 1: Single Callback Registration

```c
/** @brief UART receive callback type */
typedef void (*uart_rx_callback_t)(uint8_t byte);

/** @brief UART module */
typedef struct {
    uart_rx_callback_t on_rx;
} uart_mod_t;

void uart_register_rx_callback(uart_mod_t *p_mod, uart_rx_callback_t cb) {
    if (p_mod == NULL) return;
    p_mod->on_rx = cb;
}

/** @brief Called from ISR */
void USART1_IRQHandler(void) {
    uint8_t byte = USART1->DR;
    if (g_uart1.on_rx != NULL) {
        g_uart1.on_rx(byte);
    }
}
```

Consumer side:

```c
static void protocol_on_byte(uint8_t byte) {
    parse_frame(byte);
}

void protocol_init(void) {
    uart_register_rx_callback(&g_uart1, protocol_on_byte);
}
```

### Pattern 2: Multi-Callback Registry

```c
typedef enum {
    KEY_PRESS,
    KEY_RELEASE,
    KEY_LONG_PRESS,
    KEY_EVENT_MAX
} key_event_t;

typedef void (*key_callback_t)(uint8_t key_id);

#define KEY_MAX_CALLBACKS  4u

typedef struct {
    key_callback_t callbacks[KEY_EVENT_MAX][KEY_MAX_CALLBACKS];
    uint8_t count[KEY_EVENT_MAX];
} key_mod_t;

/**
 * @return true on success, false if table full
 */
bool key_register_callback(key_mod_t *p_mod, key_event_t event, key_callback_t cb) {
    if (p_mod == NULL || cb == NULL) return false;
    if (event >= KEY_EVENT_MAX) return false;
    if (p_mod->count[event] >= KEY_MAX_CALLBACKS) return false;
    p_mod->callbacks[event][p_mod->count[event]++] = cb;
    return true;
}

/** Dispatch event to all registered callbacks */
void key_dispatch(key_mod_t *p_mod, key_event_t event, uint8_t key_id) {
    if (p_mod == NULL) return;
    for (uint8_t i = 0u; i < p_mod->count[event]; i++) {
        if (p_mod->callbacks[event][i] != NULL) {
            p_mod->callbacks[event][i](key_id);
        }
    }
}
```

### Pattern 3: Callback with Context (void* ctx)

```c
typedef void (*timer_callback_t)(void *ctx);

typedef struct {
    timer_callback_t callback;
    void *ctx;
    uint32_t period_ms;
    uint32_t last_tick;
} timer_slot_t;

void timer_register(timer_slot_t *p_slot, timer_callback_t cb,
                    void *ctx, uint32_t period_ms) {
    if (p_slot == NULL || cb == NULL) return;
    p_slot->callback = cb;
    p_slot->ctx = ctx;
    p_slot->period_ms = period_ms;
    p_slot->last_tick = get_sys_tick();
}

void timer_poll(timer_slot_t *p_slot) {
    if (p_slot == NULL || p_slot->callback == NULL) return;
    if (get_sys_tick() - p_slot->last_tick >= p_slot->period_ms) {
        p_slot->last_tick += p_slot->period_ms;
        p_slot->callback(p_slot->ctx);
    }
}
```

Consumer side:

```c
typedef struct {
    uint16_t pin;
    bool state;
} led_t;

static void led_blink_cb(void *ctx) {
    led_t *p_led = (led_t *)ctx;
    p_led->state = !p_led->state;
    gpio_write(p_led->pin, p_led->state);
}

void led_init_blink(led_t *p_led, uint32_t period_ms) {
    timer_register(&g_timer_slots[0], led_blink_cb, p_led, period_ms);
}
```

## Application Scenarios

### 1. UART Protocol Parsing

```
UART ISR → g_uart1.on_rx(byte) → protocol layer parse_frame(byte)
                                   → frame complete → g_protocol.on_packet(pkt)
                                                       → business logic
```

Each layer only exposes a registration API; the layer above has no knowledge of how the layer below transmits/receives.

### 2. ADC Conversion Complete

```c
typedef void (*adc_done_callback_t)(uint16_t *buffer, uint16_t len);
void adc_register_done_callback(adc_done_callback_t cb);
// DMA IRQ complete → callback → data processing module
```

### 3. Key Matrix

```c
key_register_callback(&g_key, KEY_PRESS,      led_on_key);
key_register_callback(&g_key, KEY_LONG_PRESS, motor_stop);
key_register_callback(&g_key, KEY_RELEASE,    buzzer_beep);
```

### 4. Timer Soft-Scheduler (no RTOS multitasking)

```c
timer_register(&slots[0], task_led,    &g_led,    500u);
timer_register(&slots[1], task_sensor, &g_sensor, 100u);
timer_register(&slots[2], task_comm,   &g_comm,   50u);

void main_loop(void) {
    while (1u) {
        for (int i = 0; i < MAX_TIMERS; i++) {
            timer_poll(&slots[i]);
        }
    }
}
```

## Pitfalls

| Pitfall | Prevention |
|---------|------------|
| Callback executes inside ISR | Set flags only in callback; process in main loop |
| Null check missing | Always `if (p->callback != NULL) p->callback(...)` |
| Long blocking in callback | Return fast; defer heavy work to main loop |
| Dangling pointer after unregister | Module `deinit()` sets callback to NULL |
| Multi-callback array overflow | Reject when `count >= MAX` |
| void* ctx type mismatch | Caller and callback must agree on ctx type contract |

## Install Commands

**Hermes:**
```bash
hermes skills install https://raw.githubusercontent.com/<user>/<repo>/main/skills/embedded-fnptr-register/SKILL.md
```

**Claude Code:** tell Claude —
> "Install this skill: fetch https://raw.githubusercontent.com/.../SKILL.md, apply its callback-registration pattern to this project."

**Codex:** tell Codex —
> "Read and apply this skill from https://raw.githubusercontent.com/.../SKILL.md — use function-pointer registration for all event-driven code."
