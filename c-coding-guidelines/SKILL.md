---
name: c-coding-guidelines
description: C 语言编码规范——写 C 代码(.c/.h)、嵌入式 C 项目的文件组织、命名约定、函数设计、宏定义、ISR 约束、static/const 正确性等编码准则。适用于 STM32、ESP32、ARM Cortex-M 等平台。
license: MIT
metadata:
  tags: c, embedded, coding-style, naming, formatting
---

# C 语言编码规范

在编写、修改或审查嵌入式 C 代码时，严格遵循以下准则。适用于 ESP32、STM32 等嵌入式平台，兼顾 C99/C11 标准。

## 1. 文件级注释

每个 `.h` 和 `.c` 文件顶部必须有 Doxygen 风格的文件头块。

例如：

```c
/*******************************************************************************
 * @file           : hal.h
 * @author         : name (email@example.com)
 * @brief          : Hardware Abstraction Layer — porting interface
 * @date           : 2026-05-05
 *
 * Optional module overview — describe the file's role and key design decisions.
 *
 * SPDX-License-Identifier: MPL-2.0
 * This Source Code Form is subject to the terms of the Mozilla Public
 * License, v. 2.0. If a copy of the MPL was not distributed with this file,
 * You can obtain one at https://mozilla.org/MPL/2.0/.
 * Copyright (c) 2025 Name. All rights reserved.
 *****************************************************************************/
```

**规则**：
- 必须包含 `@file`、`@author`、`@brief`、`@date`
- SPDX 许可证标识之后，必须写出完整的许可证条款正文（不可省略成 `...`），后接版权声明。具体文本取决于项目实际采用的许可证
- 可选附加模块概述段落（许可证块之前，多行 `*` 注释块）

---

## 2. Include Guard

头文件必须使用 `#ifndef` / `#define` / `#endif` 防卫宏。

例如：

```c
#ifndef FOC_HAL_H
#define FOC_HAL_H

#include <stdint.h>
#include "foc_config.h"

/* ... 正文 ... */

#ifdef __cplusplus
extern "C" {
#endif

/* ... */

#ifdef __cplusplus
}
#endif

#endif // FOC_HAL_H
```

**规则**：
- Guard 宏名 = `全大写文件名_扩展名`，如 `FOC_HAL_H`
- `#endif` 后用 `// FILENAME_H` 注释闭合对象
- 所有头文件必须包含 `extern "C"` 包裹块，保障 C++ 混编兼容
- 禁止使用双下划线前缀（`__FOC_HAL__`）或以 `_` 开头加大写字母——这些是编译器保留的

---

## 3. Include 分组与顺序

Include 指令按四组排列，组间空行分隔，每组内按字母序排列。

例如：

```c
// 1. C 标准库
#include <math.h>
#include <stdio.h>
#include <stdlib.h>

// 2. RTOS / 系统头文件
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

// 3. 硬件驱动 / 平台 SDK
#include "driver/gptimer.h"
#include "driver/gpio.h"
#include "driver/mcpwm_cmpr.h"

// 4. 本项目组件
#include "as5600.h"
#include "foc.h"
#include "foc_info.h"
#include "smo.h"
```

---

## 4. 命名约定

| 类别 | 格式 | 示例 |
|------|------|------|
| 宏 / 常量 | `UPPER_SNAKE_CASE` | `FOC_FABS(x)`, `ENCODER_RES`, `MOTOR_PP`, `PWM_PERIOD` |
| 函数 | `snake_case` | `foc_create()`, `pwm_set_duty()`, `foc_tick()` |
| 文件级静态函数 | `snake_case` | `static void esp32_mcpwm_init()` |
| 变量 | `snake_case` | `adc_cache_u`, `mid_freq_cnt`, `smo_observer` |
| typedef 类型 | `snake_case` + `_t` 后缀 | `foc_t`, `foc_ops_t`, `as5600_handle_t` |
| 回调 typedef | `snake_case` + `_t` 后缀 | `foc_duty_set_t`, `foc_angle_get_t` |
| 枚举类型 / 值 | `snake_case` 类型 + `UPPER_SNAKE_CASE` 值 | `foc_mode_t` / `FOC_MODE_VELOCITY`, `foc_state_t` / `FOC_STATE_READY` |
| 结构体成员 | `snake_case` | `set_duty_cycle`, `pwm_start`, `get_angle` |

---

## 5. static 与 const 正确性

**核心原则：默认加上最紧的限制，需要放宽时再显式去除。**

例如：

```c
/* ----- 文件内部函数：一律 static ----- */
static void esp32_mcpwm_init(void) { ... }
static void timer_2khz_init(void) { ... }

/* ----- 文件内部变量：static ----- */
static const char *TAG = "foc_port_esp32";           // 不可变字符串
static volatile int adc_cache_u = 0;                 // ISR-任务共享缓存

/* ----- 不可变数据：const ----- */
const foc_ops_t ops = {
    .set_duty_cycle  = pwm_set_duty,
    .pwm_start       = pwm_start,
    .pwm_pause       = pwm_pause,
};

/* ----- const 用法一览 ----- */
const char *ptr;           // 指向不可变数据的可变指针
char * const ptr;           // 指向可变数据的不可变指针
const char * const ptr;    // 指向不可变数据的不可变指针

/* ----- 参数：不修改则 const ----- */
foc_err_enum_t foc_link_ops(foc_t *foc, const foc_ops_t *ops);
                         // 不会修改ops         ^^^^^
```

**规则**：
- 文件级函数和全局变量必须加 `static`
- 不修改的参数加 `const`
- 不修改的局部/全局变量加 `const`
- ISR 与任务间共享的变量加 `volatile` + `static`
- 只读数据结构（回调表、配置表）加 `const`

---

## 6. 类型使用

| 场景 | 使用类型 | 说明 |
|------|---------|------|
| 大小/计数/索引 | `size_t` | 无符号，匹配 `sizeof` 返回值和 `memset` 参数 |
| 确切位宽 | `uint8_t` / `uint16_t` / `uint32_t` | 硬件寄存器、协议字段、位掩码 |
| 信号处理标量 | `float` / `double` | FOC 计算、PID 参数、滤波器系数 |
| 布尔值 | `bool` + `true` / `false` | `#include <stdbool.h>` |
| 错误码 | 项目级枚举 typedef | `foc_err_enum_t`，不用裸 `int` |
| 原始下标的 for 循环 | `int` | 仅在值域明确受限时（如 `for (int i = 0; i < 3; i++)`） |

例如：

```c
/* 正确 */
size_t num_samples = 0;
uint16_t raw_angle;
float velocity_rps;
bool calibration_success = false;

/* 错误 */
int len = sizeof(foc_t);       // 应该用 size_t
unsigned short duty;           // 应该用 uint16_t 表示硬件寄存器位宽
char flag = 1;                 // 应该用 bool flag = true
```

---

## 7. 函数设计

### 7.1 单一职责

一个函数只做一件事。例如：

```c
/* 正确：只读取编码器原始值，不做角度换算、不打印、不校准 */
static void encoder_read_raw(uint16_t *out) {
    *out = (uint16_t)(spi_rx_buf[0] << 8) | spi_rx_buf[1];
}

/* 错误：读取、换算角度、计算转速、串口打印全塞在一个函数里 */
static void encoder_process(void) {
    uint16_t raw = (uint16_t)(spi_rx_buf[0] << 8) | spi_rx_buf[1];
    float theta = raw * (TWOPI / ENCODER_RES);
    float vel = (theta - g_last_theta) * LOOP_FREQ;
    printf("theta=%.3f vel=%.1f\r\n", theta, vel);
    g_last_theta = theta;
}
```

### 7.2 参数校验前置

例如：

```c
foc_err_enum_t foc_link_ops(foc_t *foc, const foc_ops_t *ops) {
#if FOC_USE_FULL_ASSERT == 1
    FOC_NULL_ASSERT(foc, FOC_ERR_NULL_PTR);     // 校验在最前面
    FOC_NULL_ASSERT(ops, FOC_ERR_NULL_PTR);
#endif
    memcpy(&foc->ops, ops, sizeof(foc_ops_t));
    return FOC_ERR_OK;
}
```

### 7.3 提前返回，避免深层嵌套

例如：

```c
/* 正确：错误情况及早返回 */
if (voltage_divider <= 0.0f || voltage_divider > 1.0f) {
    return FOC_ERR_INVALID_PARAM;
}
/* ... 主逻辑 ... */

/* 错误：深层 if-else 嵌套 */
if (ptr != NULL) {
    if (state == READY) {
        if (condition) {
            /* ... 三层嵌套才到主逻辑 */
        }
    }
}
```

### 7.4 分配后立即初始化

例如：

```c
/* 正确 */
foc_t *foc = (foc_t *)FOC_MALLOC(sizeof(foc_t));
memset(foc, 0, sizeof(foc_t));

/* 错误：分配后不初始化就跑 */
foc_t *foc = malloc(sizeof(foc_t));
foc->velocity = 0.0f;  // 除此之外的字段全是垃圾值
```

---

## 8. 结构体初始化

优先使用**指定初始化器**（C99 designated initializers），未列出的字段自动归零。

例如：

```c
/* 推荐 —— 指定初始化器，清晰且安全 */
foc_ops_t ops = {
    .set_duty_cycle    = pwm_set_duty,
    .pwm_start         = pwm_start,
    .pwm_pause         = pwm_pause,
    .get_angle         = get_angle_from_encoder,
    .get_current_sample = get_current_sample,
    .delay             = foc_port_delay_us,
};

/* 也可以 —— 显式零初始化后逐字段赋值 */
foc_ops_t ops = {0};
ops.set_duty_cycle = pwm_set_duty;
ops.pwm_start = pwm_start;

/* 错误 —— 顺序依赖，新增字段后静默破坏 */
foc_ops_t ops = { pwm_set_duty, pwm_start, pwm_pause };  // 不写字段名，改结构体就死
```

ESP-IDF / 硬件配置结构体同样遵循指定初始化器，例如：

```c
gptimer_config_t config = {
    .clk_src = GPTIMER_CLK_SRC_DEFAULT,
    .direction = GPTIMER_COUNT_UP,
    .resolution_hz = GPTIMER_RESOLUTION_HZ,
};
```

---

## 9. 宏与预处理

### 9.1 函数式宏必须括号保护

例如：

```c
/* 正确 —— 每个参数和整体都加括号 */
#define FOC_FABS(x)    ((x) < 0.0f ? -(x) : (x))
#define FOC_FMAX(a, b) ((a) > (b) ? (a) : (b))
#define FOC_FMIN(a, b) ((a) < (b) ? (a) : (b))

/* 危险 —— 缺少括号保护 */
#define FMAX(a, b) a > b ? a : b    // FMAX(x, y) + 1 展开为 x > y ? x : y + 1
#define SQUARE(x) x * x             // SQUARE(1+2) 展开为 1+2*1+2
```

### 9.2 `#if` / `#elif` / `#endif` 必须注释条件

例如：

```c
#if FOC_MATH_CAL_METHOD == 0
#include <math.h>
#define FOC_SINE(x)     (sinf(x))
#elif FOC_MATH_CAL_METHOD == 1
#include <arm_math.h>
#define FOC_SINE(x)     (arm_sin_f32(x))
#endif /* FOC_MATH_CAL_METHOD */

#if FOC_MATH_CAL_METHOD != 0
#include "qmath_fixed.h"
#ifndef FOC_ATAN2
#define FOC_ATAN2(y, x)  q31_atan2((y), (x))
#endif
#endif /* FOC_MATH_CAL_METHOD != 0 */
```

### 9.3 条件编译保持简洁

条件编译块不宜超过 30 行。超过则提取为独立的 `_impl.c` 文件，通过构建系统选择。

---

## 10. 嵌入式专项规范

### 10.1 ISR 约束（硬性规则）

ISR 中**绝对禁止**：
- `printf` / `ESP_LOGI` 等日志输出
- `malloc` / `free` 等堆操作
- 阻塞等待（信号量、队列 take、延时）
- 长循环（超过 10 行）

ISR 中**只能做**：
- 读写硬件寄存器
- 设置 `volatile` 标志 / 缓存值
- `vTaskNotifyGiveFromISR` / `xQueueSendFromISR` / `xSemaphoreGiveFromISR`
- 简单的整数运算和分支

**浮点约束**：
- ESP32：ISR 中**禁止**浮点运算（硬件不支持自动保存 FPU 上下文，会导致寄存器损坏）
- STM32（Cortex-M4/M7 带 FPU）：**允许**浮点运算，但编译器会自动压栈 FPU 寄存器（约增加 $1$–$2$ µs 延迟）。如需严格控制 ISR 延迟，仍建议将浮点计算移到任务中

例如：

```c
/* 正确的 ISR 模式：极轻量通知 */
static bool IRAM_ATTR timer_3khz_callback(gptimer_handle_t timer,
                                          const gptimer_alarm_event_data_t *edata,
                                          void *user_ctx) {
    BaseType_t mustYield = pdFALSE;
    vTaskNotifyGiveFromISR(foc_high_freq_task_handle, &mustYield);
    return (mustYield == pdTRUE);
}
```

### 10.2 ESP32 特定

- ISR 函数必须标注 `IRAM_ATTR`
- 浮点运算放在任务中，不在 ISR 中
- 使用 `DRAM_ATTR` 标注 DMA 缓冲区（若需要）
- `esp_rom_delay_us` 用于微秒级忙等（<1000 us）
- 长时间延时用 `vTaskDelay` 让出 CPU

### 10.3 ISR 与任务通信

通信通道按优先级递减：
1. `vTaskNotifyGiveFromISR` （最快，无内存分配）
2. `xQueueSendFromISR` / `xQueueOverwriteFromISR`（数据传递）
3. `volatile` 共享变量（单向 read/write，不冲突时使用）
4. 禁止在 ISR 中调用 `xQueueSend`（不是 FromISR 版本）

### 10.4 内存与栈

- `malloc` 后立即 `memset(ptr, 0, size)` 零初始化
- `free` 后不悬空，如局部指针则不必置 NULL；如全局/持久指针则置 NULL
- FreeRTOS 任务栈大小：计算型任务 >= 4096 字节，I/O 任务 >= 2048 字节

例如：

```c
xTaskCreate(foc_high_freq_task_func, "foc_cur", 4096, NULL, 6, &task_cur);
xTaskCreate(get_angle_task,         "get_angle", 2048, NULL, 6, &task_angle);
```

---

## 11. 错误处理

### 11.1 使用枚举错误码

禁止用裸 `int` 0/1/-1 表示错误。必须定义枚举，0 为成功。

例如：

```c
typedef enum {
    FOC_ERR_OK            = 0,
    FOC_ERR_NULL_PTR      = -1,
    FOC_ERR_INVALID_PARAM = -2,
    FOC_ERR_NOT_READY     = -3,
    FOC_ERR_TIMEOUT       = -4,
} foc_err_enum_t;
```

### 11.2 公共接口返回错误码

例如：

```c
foc_err_enum_t foc_link_ops(foc_t *foc, const foc_ops_t *ops);
```

### 11.3 不忽略返回值

例如：

```c
/* 正确 */
uint8_t as5600_res = as5600_init(&motor1_as5600_handle);
if (as5600_res != 0) {
    printf("Motor1 as5600 init failed (%d)\r\n", as5600_res);
    return;
}

/* 错误 */
as5600_init(&motor1_as5600_handle);  // 返回值被丢弃
```

### 11.4 `#if` 守卫的 assert

assert 应通过编译期开关控制启用/禁用，release 版本可以零开销消除。例如：

```c
/* ---- 头文件 ---- */
#if ASSERT_ENABLE == 1
#define ASSERT_NULL(ptr, ret)  do { if ((ptr) == NULL) return (ret); } while (0)
#else
#define ASSERT_NULL(ptr, ret)  ((void)0)
#endif

/* ---- 使用位置 ---- */
err_t fn_init(struct dev_t *dev, const cfg_t *cfg) {
#if ASSERT_ENABLE == 1
    ASSERT_NULL(dev, ERR_NULL);            /* 仅在调试版本生效 */
    ASSERT_NULL(cfg, ERR_NULL);
#endif
    /* ... 实际逻辑，此时 dev 和 cfg 一定非 NULL ... */
    return OK;
}
```

---

## 12. 格式化

- 缩进：4 空格（不用 Tab）
- 行宽：不超过 120 字符
- 大括号：K&R 风格（函数/结构体的 `{` 在同一行末尾）
- `if` / `for` / `while` 的 `{ }` 不可省略，即使只有一行
- 指针星号靠变量名：`int *ptr`（而非 `int* ptr`）
- 逗号后加空格：`func(a, b, c)`
- 运算符两侧加空格：`a + b`、`x = y`
- 函数名与 `(` 之间不加空格：`func()`、`foc_tick(foc)`

---

## 13. 禁止事项

- 禁止在头文件中定义非 `static inline` 函数（会导致 ODR 违规）
- 禁止使用 `goto`（除非跳出多层嵌套的错误清理路径，且仅 `goto cleanup;`）
- 禁止使用 `#pragma once`（非标准；用 Include Guard 代替，见第 2 节）
- 禁止在 ISR 中调用 `printf` / `malloc` / `free`（见第 10.1 节）
- 禁止在 ISR 中进行浮点运算 —— ESP32 严格禁止；STM32 Cortex-M4/M7 带 FPU 可放宽（见第 10.1 节）
- 禁止裸 `0` / `1` 表示布尔值（用 `false` / `true` + `stdbool.h`）
- 禁止未加括号保护的函数式宏（见第 9.1 节）
- 禁止未初始化的局部变量（见第 7.4 节）
- 禁止分配内存后不检查 NULL（见第 7.4 节）
- 禁止 `memcpy` 前不确认目标缓冲区大小

---

## 14. 自检要求

输出前请自检，如出现违反以上规则的输出，重新输出。

## 15. 架构设计哲学

编写或审查涉及模块划分、接口设计、分层架构的 C 代码时，加载 [./rules/architecture.md](./rules/architecture.md) 获取架构级设计准则。

---
