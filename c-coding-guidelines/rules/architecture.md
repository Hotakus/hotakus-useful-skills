---
name: architecture
description: C 架构设计规范——模块划分、接口设计、依赖管理、分层架构、vtable 多态、不透明类型封装、通信模式选择。适用于嵌入式 C 项目的架构决策与设计审查。
license: MIT
---

# Architecture Guidelines — C 架构设计规范

本文档是 `c-coding-guidelines` Skill 的架构设计扩展。编码规范回答"怎么写"，架构规范回答"怎么设计"。适用于模块划分、接口设计、依赖管理、通信模式选择等架构决策场景。

## 1. 背景 / 为什么需要架构规则

编码规范确保每行代码的质量（命名、缩进、注释）。但即使每行代码都符合规范，错误的模块划分、循环依赖、紧耦合仍然会导致项目腐烂。

架构规则解决的是**结构性问题**：
- 模块之间如何解耦
- 接口如何定义才能可替换
- 依赖方向如何控制才能可测试
- 通信模式如何选择才能高效

这些决策一旦做错，后期重构成本极高。本文档提供一套经过嵌入式项目验证的架构决策框架，让每个模块职责清晰、边界分明、可替换可测试喵。

## 2. 不透明类型与封装

### 2.1 不透明类型（Opaque Type）

**公共头文件仅暴露前置声明 `typedef struct xxx_s xxx_t;`，完整结构体定义放在 `_internal.h` 或 `.c` 文件中。**

理由：隐藏成员布局防止外部代码直接访问，保持二进制兼容性，降低重构成本。

**正确**：

```c
// motor.h — public header
typedef struct motor_s motor_t;  // forward decl, no member access
motor_t *motor_create(void);
float motor_get_speed(const motor_t *m);
```

**错误**：

```c
// motor.h — WRONG: exposes full struct in public header
typedef struct motor_s {
    float speed;       // external code can R/W directly
    uint16_t angle;    // changing layout breaks all callers
} motor_t;
```

### 2.2 访问器函数

**不透明类型的所有成员通过 getter/setter 访问，外部代码禁止直接读写成员。**

理由：访问器提供校验、转换、日志等切入点的可能；直接成员访问无法拦截非法值。

**正确**：

```c
// getter — const 保证不修改对象
float motor_get_speed(const motor_t *m);
void motor_set_speed(motor_t *m, float speed_rps);
```

**错误**：

```c
motor->speed = 100.0f;  // direct write bypasses validation
                        // no clamp, no unit conversion, no trigger
```

## 3. 虚函数表与多态

### 3.1 vtable 结构体

**用函数指针结构体实现多态，禁止用 switch-case 按类型 ID 分发。**

理由：switch-case 方案新增硬件类型需要修改核心代码，违反开闭原则。vtable 方案让核心与实现完全解耦（参考 FOC 项目中 `foc_ops_t` 的模式喵）。

**正确**：

```c
typedef struct motor_ops_t {
    void (*set_duty)(float a, float b, float c);
    void (*get_angle)(uint16_t *out);
} motor_ops_t;
```

**错误**：

```c
void set_duty(motor_t *m, float a, float b, float c) {
    switch (m->hw_type) {         // adding new hw = modify core
        case HW_A: a_pwm_set(a, b, c); break;
        case HW_B: b_pwm_set(a, b, c); break;
    }
}
```

### 3.2 依赖注入

**vtable 通过 memcpy、工厂函数或注册表注入核心。核心模块零依赖具体实现。**

理由：核心在编译期不知道具体硬件，链接期才决定使用哪个实现。（参考 FOC 项目中 `foc_link_ops()` 的 `memcpy` 注入模式。）

**正确**：

```c
motor_err_t motor_link_ops(motor_t *m, const motor_ops_t *ops) {
    if (m == NULL || ops == NULL) return MOTOR_ERR_NULL_PTR;
    memcpy(&m->ops, ops, sizeof(motor_ops_t));
    return MOTOR_ERR_OK;
}
```

**错误**：

```c
void motor_init(motor_t *m) {
    a_pwm_init();                    // core depends on specific platform
    m->ops.set_duty = a_pwm_set;     // can't test without real PWM
}
```

### 3.3 可选回调

**可选回调在头文件中标注为可 NULL，调用前检查后再调用。**

理由：嵌入式 HAL 中某些功能（如母线电压读取、故障信号）可能不存在，统一接口通过 NULL 回调优雅降级。

**正确**：

```c
if (m->ops.get_bus_voltage) {
    m->ops.get_bus_voltage(&v);   // safe: checked before call
}
```

**错误**：

```c
m->ops.get_bus_voltage(&v);       // NULL check missing, crash if unset
                                  // caller can't tell if optional
```

## 4. 接口与实现分离

### 4.1 公共头文件规则

**公共头文件只暴露调用者需要的内容：前置声明、API 函数声明、必要的常量宏。不暴露实现细节。**

理由：减少编译依赖，降低头文件变动引发的级联重编译。

**正确**：`motor.h` 只包含 `typedef struct motor_s motor_t;` 和 `motor_t *motor_create(void);` 等 API。

**错误**：

```c
// motor.h — WRONG: exposes implementation details
#include "a_pwm.h"              // leaks platform dependency to all includers
static inline float _internal_helper(void) { ... }  // internal function leaked
```

### 4.2 内部头文件

**内部细节放在 `_internal.h` 中，仅被本模块的 `.c` 和受信任的子模块包含。`_internal.h` 不在公共包含路径中。**

理由：防止外部代码意外依赖内部布局，保持封装边界。

**正确**：

```c
// motor_internal.h — NOT in public include path, included only by motor.c
typedef struct motor_s {
    float speed;
    uint16_t angle;
    motor_ops_t ops;
} motor_t;
```

**错误**：

```c
// sensor.h — WRONG: includes internal header in public API
#include "motor_internal.h"     // leaks motor internals to all sensor users
```

### 4.3 纯类型头文件

**跨模块共享的类型提取到纯类型头文件中（仅 typedef + 常量），不包含任何逻辑实现。**

理由：打破循环依赖 —— 两个模块各自包含对方的头文件会导致编译错误。

**正确**：`motor_types.h` 包含 `typedef enum motor_state_t { ... } motor_state_t;` 和 `#include "pid.h"`（类型依赖）。

**错误**：

```c
// motor.h and sensor.h — each #include each other → circular dependency
// motor.h: #include "sensor.h"
// sensor.h: #include "motor.h"  // compile error: type not yet defined
```

## 5. 模块分层架构

### 5.1 分层原则

**代码按严格单向依赖分层：Application → Service/Mode → Core Engine → HAL。水平组件（PID、滤波器、观测器）通过 vtable 注入，不形成独立层。**

理由：单向依赖保证模块可独立测试。上层替换不影响下层，下层修改不破坏上层。

**正确**：

```c
// App layer calls Service, Service calls Core, Core calls HAL
void app_task(void *arg) {
    motor_service_run();          // app does NOT call pwm_set_duty() directly
}

void motor_service_run(void) {
    motor_tick(&g_motor);         // service calls core's public API
}
```

**错误**：

```c
// WRONG: app skips service and core, calls HAL directly
void app_task(void *arg) {
    pwm_set_duty(0.5f, 0.5f, 0.0f);   // bypasses 2 layers
}

// WRONG: HAL calls app callback (reverse dependency)
void hal_pwm_isr(void *arg) {
    app_emergency_stop();             // HAL should NOT know about app
}
```

### 5.2 层间通信

**跨层调用只能通过定义好的接口（vtable 或公开 API）进行，不允许层跨越。水平组件在核心调度下通过 vtable 注入运行，不形成独立调用链。**

理由：保证层间契约清晰可见，任何跨层通信都在接口定义处留痕。

## 6. 组件通信模式选择指南

**根据场景选择通信方法，不盲目使用同一种机制。**

理由：不同通信模式的开销、复杂度、线程安全性差异显著。错选会导致性能下降或可维护性恶化。

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 同层同步调用 | 直接函数调用 | 开销最小，调用链清晰 |
| 跨层可替换逻辑 | vtable 回调 | 多态解耦，编译期不绑定 |
| 任务间数据传递 | Queue / Message | 异步安全，避免竞态 |
| ISR → 任务信令 | TaskNotify | 最快 ISR 退出，零内存分配 |

**正确**：

```c
// ISR → task: 使用 TaskNotify（最快）
static void IRAM_ATTR timer_isr(void *arg) {
    vTaskNotifyGiveFromISR(motor_task_handle, NULL);
}

// Task 等待通知，不浪费 CPU
void motor_task(void *arg) {
    while (1) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        motor_tick(&g_motor);
    }
}
```

**错误**：

```c
// WRONG: 任务间传递复杂可变结构体通过全局 volatile
volatile motor_state_t g_shared_state;  // race condition

// WRONG: ISR 中仅设一个标志位却用 QueueSend
void IRAM_ATTR timer_isr(void *arg) {
    xQueueSendFromISR(flag_queue, &dummy, NULL);  // overkill vs TaskNotify
}

// WRONG: 任务轮询全局标志而不是阻塞等待通知
volatile bool flag;
void motor_task(void *arg) {
    while (1) {
        if (flag) {          // busy-waits, wastes CPU cycles
            flag = false;
            motor_tick(&g_motor);
        }
    }
}
```

## 7. SOLID 原则的 C 映射

### 7.1 单一职责（SRP）

**一个 `.c` / `.h` 对只负责一类职责。**

理由：职责混杂的模块难以测试、难以复用、修改时影响范围不可控。

**正确**：`pid.c` 只做 PID 控制计算（输入误差 → 输出控制量）。

**错误**：

```c
// pid.c — WRONG: mixes PID calculation with unrelated concerns
void pid_compute(pid_t *p, float err) {
    pid_update(p, err);
    motor_set_pwm(p->out);  // PID module should not know about motor
    log_printf("pid=%f\n", p->out);  // nor about logging
    sensor_read(&val);      // nor about sensors
}
```

### 7.2 开闭原则（OCP）

**对扩展开放（新增 vtable 实现），对修改关闭（不修改核心代码）。**

理由：新增硬件平台时，核心控制逻辑不应有任何代码变更。

**正确**：新增传感器类型 → 实现 `sensor_ops_t` → 调用 `sensor_register()` 注册。核心循环零修改。

**错误**：

```c
// WRONG: core loop branches on sensor type
void motor_tick(motor_t *m) {
    if (m->sensor == SENSOR_A) {
        a_read(&m->angle);
    } else if (m->sensor == SENSOR_B) {
        b_read(&m->angle);
    }
    // adding SENSOR_C requires modifying core loop
}
```

### 7.3 里氏替换（LSP）

**每个 vtable 实现必须完整满足接口契约。实现不能弱化前置条件或强化后置条件。**

理由：调用方依赖契约而非实现。某个实现行为不一致会导致难以追踪的运行时故障。

**正例**：所有 `sensor_ops_t` 实现输出有效的角度值（0–2π）。错误通过错误码返回。

**反例**：某个实现的 `read()` 返回未初始化的角度而不报错 → 电机飞车。

### 7.4 接口隔离（ISP）

**vtable 小而专注，调用方不依赖它们不需要的回调。**

理由：大而全的 vtable 迫使每个实现都提供无意义的空函数桩，增加维护负担。

**正确**：`pwm_ops_t`（3 个函数）、`adc_ops_t`（2 个函数）—— 按职责拆分。

**错误**：

```c
// WRONG: 15+ unrelated callbacks in one vtable
typedef struct mega_ops_t {
    void (*pwm_set)(float a, float b, float c);          // PWM
    void (*adc_read)(uint16_t *out);                     // ADC
    void (*enc_read)(uint16_t *angle);                   // encoder
    void (*can_send)(uint32_t id, uint8_t *data);        // CAN
    void (*log_print)(const char *msg);                  // logging
} mega_ops_t;  // mixed concerns: every impl must stub unused callbacks
```

### 7.5 依赖反转（DIP）

**高层模块依赖抽象（vtable），不依赖具体实现。**

理由：核心代码编译时不知道具体硬件平台，链接时才绑定。

**正确**：`motor_core.c` 包含 `"motor_ops.h"`（抽象接口），不包含 `"a_pwm.h"`（具体实现）。

**错误**：

```c
// motor_core.c — WRONG: directly depends on platform header
#include "a_pwm.h"             // core now tied to platform A
#include "a_encoder.h"

void motor_tick(motor_t *m) {
    a_pwm_set_duty(m->duty_u, m->duty_v, m->duty_w);  // can't port
    a_encoder_read(&m->angle);                          // to platform B
}
```

## 8. 禁止事项（架构反模式）

1. 禁止在公共头文件中暴露完整 struct 定义（纯数据 POD 配置参数除外）
2. 禁止核心模块直接 `#include` 平台特定头文件（必须通过 HAL vtable）
3. 禁止用 switch-case 按类型分发替代 vtable 多态
4. 禁止循环依赖（A.h `#include` B.h, B.h `#include` A.h）
5. 禁止跨层调用（上层跳过中间层直接调用底层）

## 9. 自检要求

架构设计完成后，逐项核查以下条目（每项均可通过阅读代码验证）：

- [ ] 所有公共头文件是否只暴露前置声明 + API？无 struct 成员泄露？
- [ ] 需要多态的场景是否使用了 vtable 而非 switch-case？
- [ ] 核心模块是否零依赖平台特定头文件？
- [ ] 分层依赖方向是否单向（上层 → 下层），无反向依赖？
- [ ] vtable 中可选回调是否标注且调用前检查 NULL？
- [ ] 头文件是否避免了循环依赖？
- [ ] 每个模块是否职责单一（一个 `.c` 只做一类事）？
- [ ] ISR 与任务通信是否选择了合理机制（TaskNotify > Queue > volatile）？
- [ ] vtable 接口是否足够小，避免给实现者强加无意义空函数？
- [ ] 新增硬件平台是否只需要新增 vtable 实现，无需修改核心代码？
