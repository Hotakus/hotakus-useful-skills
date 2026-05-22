---
name: embedded-testing-guidelines
description: 嵌入式测试指南——写测试、单元测试、集成测试、硬件在环测试时的分层策略、测试接缝识别、Mock 策略、框架选型、离板与片上测试实践。覆盖 ESP32、STM32、通用 ARM Cortex-M 等平台。
license: MIT
metadata:
  tags: testing, embedded, c, unit-testing, integration-testing
---

# 嵌入式测试指南

在编写、组织嵌入式 C 项目的测试时，严格遵循以下准则。本指南覆盖 ESP32（ESP-IDF）、STM32（裸机 / FreeRTOS）、通用 ARM Cortex-M、Zephyr RTOS 及 PC 端离板测试场景，不绑定任何具体 HAL 设计。

## 1. 测试分层

嵌入式测试按硬件依赖度划分为三层。投入产出比从高到低排列，优先覆盖上层。

| 分层 | 运行环境 | 典型场景 | 反馈速度 | 覆盖率目标 |
|------|---------|---------|:---:|:---:|
| **离板纯逻辑** | PC 端 `gcc` / `clang` | 数学函数、坐标变换、PID、LPF、协议解析、状态机 | 毫秒级 | ≥ 80% |
| **片上单元测试** | 目标 MCU + 测试框架 | 需 FreeRTOS / 中断模拟的逻辑、校准算法、外设驱动 mock | 秒级 | 核心路径 |
| **硬件在环 (HIL)** | 真板 + 真实外设 | 校准流程、故障保护链路、电机实物验证 | 分钟级 | 关键链路 |

---

## 2. 找到测试接缝

所有嵌入式项目都有一条"纯 C 逻辑 ↔ 硬件 I/O"的边界线。测试的关键是在此切开，mock 硬件侧，验证逻辑侧。不同项目用不同方式组织这条线。

### 2.1 回调结构体注入型

项目将硬件操作抽象为一组函数指针，运行时通过结构体注入。Mock 方式：换一组函数指针。

例如：

```c
/* 原接口定义 */
typedef struct {
    void (*set_pwm)(uint16_t duty);
    uint16_t (*read_adc)(uint8_t chan);
} hal_ops_t;

/* 真实硬件 */
const hal_ops_t real_ops = { .set_pwm = pwm_write, .read_adc = adc_read };

/* 测试 mock —— 只捕获写入的值 */
static uint16_t g_captured_duty;
static uint16_t g_mock_adc_seq[] = {2048, 2100, 2060};
static uint8_t g_mock_seq_idx;
static void mock_set_pwm(uint16_t duty) { g_captured_duty = duty; }
static uint16_t mock_read_adc(uint8_t chan) { return g_mock_adc_seq[g_mock_seq_idx++]; }

const hal_ops_t mock_ops = { .set_pwm = mock_set_pwm, .read_adc = mock_read_adc };
```

### 2.2 HAL 库调用型

项目直接调用平台 SDK（如 `HAL_GPIO_WritePin`、`gpio_set_level`）。Mock 方式：链接时覆盖符号（`-Wl,--wrap`）或编译时 `#define` 替换。

例如：

```c
/* 链接时 mock —— 在测试 Makefile 中 */
/* LDFLAGS += -Wl,--wrap=HAL_GPIO_WritePin */

/* mock 实现 */
void __wrap_HAL_GPIO_WritePin(GPIO_TypeDef *port, uint16_t pin, GPIO_PinState state) {
    g_last_pin = pin;
    g_last_state = state;
}
```

### 2.3 寄存器直操型

裸机项目直接写 `TIM1->CCR1 = duty`。Mock 方式：将寄存器宏重定义为普通变量。

例如：

```c
/* 测试模式下的寄存器替换 */
#ifdef TESTING
    static uint32_t mock_tim1_ccr1;
    #define TIM1_CCR1 mock_tim1_ccr1
#else
    #define TIM1_CCR1 (*(volatile uint32_t *)0x40001034)
#endif
/* 被测代码无需修改 */
void set_duty(uint32_t val) { TIM1_CCR1 = val; }
```

### 2.4 驱动对象型

项目以面向对象 C 组织驱动，每个外设是一个带函数指针的结构体。Mock 方式：注入假对象。

例如：

```c
struct spi_dev {
    int (*transfer)(struct spi_dev *self, const uint8_t *tx, uint8_t *rx, size_t len);
    void *priv;
};

/* Mock 对象：返回预设数据 */
static int mock_spi_transfer(struct spi_dev *self, const uint8_t *tx, uint8_t *rx, size_t len) {
    memcpy(rx, g_mock_spi_rx_buf, len);
    return 0;
}
struct spi_dev mock_spi = { .transfer = mock_spi_transfer };
```

### 2.5 事件回调型

项目通过注册回调响应硬件事件（蓝牙 GATT 回调、传感器数据就绪通知、定时器 ISR 通知）。Mock 方式：直接调用回调函数，注入模拟事件。

例如：

```c
static sensor_data_t g_last_event;

/* 被测模块注册的回调 */
static void on_sensor_ready(const sensor_data_t *data) {
    g_last_event = *data;
}

/* 测试中手动触发 */
void test_sensor_callback_handling(void) {
    sensor_data_t fake = { .temp = 25.3f, .humidity = 60 };
    on_sensor_ready(&fake);              /* 不经过硬件 ISR */
    TEST_ASSERT_EQUAL_FLOAT(25.3f, g_last_event.temp);
}
```

### 2.6 Linux 嵌入式型

项目运行在嵌入式 Linux（Raspberry Pi、Yocto 等），操作 sysfs、spidev、i2c-dev。Mock 方式：`LD_PRELOAD` 拦截 `open` / `read` / `write` / `ioctl`，或构造 tmpfs 下的假设备文件。

例如：

```c
/* 被测代码 */
int fd = open("/dev/spidev0.0", O_RDWR);
ioctl(fd, SPI_IOC_MESSAGE(1), &tr);

/* LD_PRELOAD mock */
int open(const char *path, int flags, ...) {
    if (strstr(path, "spidev")) return MOCK_FD_SPI;
    return ((int (*)(const char *, int, ...))real_open)(path, flags);
}
```

---

## 3. Mock 策略

三种策略按侵入性从小到大排列。优先选编译时，最不济选运行时。

| 策略 | 机制 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **编译时** | `#define` 替换 / `#ifdef TESTING` | 零运行时开销 | 代码中需预留测试分支 | 寄存器直操、HAL 宏 |
| **链接时** | `-Wl,--wrap=symbol` / `LD_PRELOAD` | 被测代码零修改 | 需 GNU ld，平台受限 | HAL 库调用型、Linux 嵌入式 |
| **运行时** | 函数指针注入 / 结构体替换 | 灵活，可逐用例替换 | 少量运行时开销 | 回调结构体注入型、驱动对象型 |

---

## 4. 框架选型

| 平台 | 推荐框架 | 组织方式 |
|------|---------|---------|
| ESP-IDF | Unity（SDK 内置） | `test/` 目录，执行 `idf.py test` |
| STM32 裸机 | Ceedling + CMock | Ruby 生态，自动生成 mock |
| STM32 + FreeRTOS | 手动 Unity + CMake | 自建 `test/` 目录，链接 FreeRTOS 内核 |
| 通用 ARM Cortex-M | Ceedling 或 Zephyr Ztest | 前者灵活，后者集成度高 |
| Zephyr RTOS | Ztest（内置） | `tests/` 目录，`twister` 运行 |
| PC 端离板测试 | Unity + CMake + CTest | 任意平台，`.c` 文件直接编入 CI |

---

## 5. 离板测试实践

目标：在 PC 上用 `gcc` 编译运行嵌入式源码，不依赖目标 MCU 工具链。

### 5.1 剥离平台依赖

核心手法是**条件编译**，把所有平台 API 调用隔离在一个宏/接口之后：

- RTOS 依赖：`#ifndef TESTING` 包裹 `vTaskDelay`、`xQueueReceive` 等。测试模式下用 `usleep()` 或空操作替代。
- ISR 标记：`IRAM_ATTR` 等在测试宏中定义为空。
- 硬件寄存器：定义 mock 变量替代（见 2.3）。
- 浮点依赖：PC 端用标准 IEEE 754，若 MCU 用软浮点（如 `q31`），则测试中需同时验证定点版本。

例如：

```c
/* test_config.h —— 测试模式下消除平台依赖 */
#ifdef TESTING
    #include <unistd.h>
    #define IRAM_ATTR
    #define DRAM_ATTR
    #define vTaskDelay(ms)  usleep((ms) * 1000)
    #define ESP_LOGI(tag, fmt, ...)  printf(fmt "\n", ##__VA_ARGS__)
#endif
```

### 5.2 CMake 双构建目标

同一个 `CMakeLists.txt` 同时支持交叉编译和 PC 端测试构建：

```cmake
if(DEFINED ENV{IDF_PATH})
    # ESP-IDF 构建
    idf_component_register(SRCS ${SOURCES} ...)
else()
    # PC 端测试构建
    add_executable(test_foc
        test/test_math.c
        test/test_pid.c
        src/foc/foc_math.c
        src/foc/foc_pid.c
    )
    target_compile_definitions(test_foc PRIVATE TESTING)
    target_link_libraries(test_foc m Unity::unity)
    enable_testing()
    add_test(NAME foc COMMAND test_foc)
endif()
```

---

## 6. 片上测试实践

### 6.1 ESP-IDF Unity

目录结构：

```
project/
├── main/
├── components/
│   └── foc/
│       ├── src/
│       └── test/                  # 测试目录
│           ├── CMakeLists.txt     # idf_component_register + TEST_REQUIRES unity
│           ├── test_math.c
│           └── test_pid.c
```

`CMakeLists.txt` 最小内容：

```cmake
idf_component_register(SRCS "test_math.c" "test_pid.c"
                       TEST_REQUIRES unity)
```

运行：

```bash
idf.py build
idf.py test                    # 烧录并运行所有测试
idf.py test-component foc      # 仅测指定组件
```

### 6.2 STM32 + Ceedling

```bash
gem install ceedling
ceedling new my_project         # 生成 test/ 目录和 project.yml
ceedling test:all               # 编译并运行
```

`project.yml` 中指定交叉编译器：

```yaml
:tools_test_compiler:
  :executable: arm-none-eabi-gcc
:tools_test_linker:
  :executable: arm-none-eabi-gcc
```

---

## 7. 典型用例模板

### 7.1 纯逻辑测试 —— PID 阶跃响应

```c
#include "unity.h"
#include "pid.h"

static pid_t pid;

void setUp(void) {
    pid_init(&pid, 1.0f, 0.1f, 0.0f, 100.0f, -100.0f);  // Kp=1, Ki=0.1
}

void tearDown(void) { }

void test_pid_step_response_settles(void) {
    float setpoint = 10.0f;
    float measurement = 0.0f;
    float output;

    /* 模拟 100 步阶跃响应 */
    for (int i = 0; i < 100; i++) {
        output = pid_update(&pid, setpoint, measurement, 0.001f);
        measurement += output * 0.01f;  /* 简化对象模型 */
    }

    TEST_ASSERT_FLOAT_WITHIN(0.5f, setpoint, measurement);
}
```

### 7.2 Mock HAL 测试 —— 校准流程

```c
/* 注入假 ADC 数据，验证电流偏置校准的数学结果 */
static int g_fake_adc[3] = {2050, 2080, 2030};  /* 模拟 1.65V 偏置附近的 ADC 值 */

static void mock_get_current(int *u, int *v, int *w) {
    *u = g_fake_adc[0];
    *v = g_fake_adc[1];
    *w = g_fake_adc[2];
}

void test_current_offset_calibration(void) {
    foc_ops_t mock_ops = { .get_current_sample = mock_get_current };
    foc_t *foc = foc_create("test");
    foc_link_ops(foc, &mock_ops);

    foc_current_calibration(foc, 50);            /* 采样 50 次 */

    float expected_offset = (2050 + 2080 + 2030) / 3.0f * VREF / ADC_RES;
    TEST_ASSERT_FLOAT_WITHIN(0.01f, expected_offset, foc->current_offset);
    foc_destroy(foc);
}
```

### 7.3 ISR 模拟测试 —— 故障检测

```c
/* 不进入硬件 ISR，直接调用 tick 函数注入过流数据 */
static int g_overcurrent_adc[3] = {4000, 4000, 4000};  /* 远超阈值 */

static void mock_get_overcurrent(int *u, int *v, int *w) {
    *u = g_overcurrent_adc[0];
    *v = g_overcurrent_adc[1];
    *w = g_overcurrent_adc[2];
}

void test_fault_trigger_on_overcurrent(void) {
    foc_ops_t mock_ops = {
        .get_current_sample = mock_get_overcurrent,
        .pwm_start  = mock_pwm_start,
        .pwm_pause  = mock_pwm_pause_capture,  /* 捕获是否被调用 */
    };
    foc_t *foc = foc_create("test");
    foc_link_ops(foc, &mock_ops);
    foc_set_state(foc, FOC_STATE_READY);

    foc_tick(foc);  /* 模拟一次电流环 tick */

    TEST_ASSERT_EQUAL(FOC_STATE_FAULT, foc_get_state(foc));
    TEST_ASSERT_TRUE(g_pwm_pause_called);       /* 确认 PWM 被紧急关闭 */
    foc_destroy(foc);
}
```

---

## 8. 持续集成

### 8.1 PC 端离板测试 CI

适用于任何 CI 系统，无需硬件。示例 GitHub Actions 片段：

```yaml
- name: Build & Run Off-Board Tests
  run: |
    mkdir build && cd build
    cmake .. -DTESTING=ON
    cmake --build .
    ctest --output-on-failure
```

### 8.2 ESP32 真机 CI

需要 GitHub Actions self-hosted runner 连接 ESP32 板子：

```bash
idf.py build
idf.py test --junit-results=test_results.xml
```

---

## 9. 自检要求

输出前请自检，如出现违反以上规则的输出，重新输出。
