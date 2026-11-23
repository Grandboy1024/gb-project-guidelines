# C / FreeRTOS 编码与防御规范 (CODING)

规范编号：DEV-001  
版本：1.0.0  
适用范围：强制执行  

---

## 1. Modern C99/C11 严格强类型编码规范

1. **类型确定性**：
   - 严禁使用无尺寸约定的原生类型（如裸 `int`、`short`、`long`）。
   - 必须统一包含 `<stdint.h>` 与 `<stdbool.h>`，使用显式位宽类型：`uint8_t`、`int8_t`、`uint16_t`、`uint32_t`、`bool`。
2. **结构体与枚举定义**：
   - 结构体与枚举必须显式 `typedef`，结构体命名采用蛇形加 `_t` 结尾：
     ```c
     typedef struct {
         uint8_t brightness;      // 亮度 0~100
         uint16_t color_temp_k;   // 色温 2700~6500K
         bool is_power_on;        // 开关状态
     } light_state_t;
     ```
3. **常量与只读防护 (Const-Correctness)**：
   - 任何不修改输入内容的指针参数必须显式声明为 `const`（如 `const uint8_t *data, size_t len`）。

---

## 2. 内存安全与零动态碎片铁律

1. **动态内存分配限制**：
   - **绝对禁止**在主业务循环、高频调度任务或定时器回调中频繁调用 `malloc()` / `free()`。
   - 频繁变动的数据结构必须采用静态分配、预分配环形缓冲区 (Ring Buffer) 或固定大小内存池。
2. **指针使用前防御性判空**：
   - 所有接受指针入参的公开函数，第一行必须执行判空防御：
     ```c
     esp_err_t light_set_state(const light_state_t *state) {
         if (state == NULL) {
             ESP_LOGE(TAG, "Invalid argument: state is NULL");
             return ESP_ERR_INVALID_ARG;
         }
         // ...
         return ESP_OK;
     }
     ```
3. **缓冲区边界检查**：
   - 严禁使用不安全的字符串函数（如 `strcpy`、`sprintf`、`strcat`）；必须使用带长度保护的 `strncpy`、`snprintf`、`strncat`。

---

## 3. 中断服务例程 (ISR) 极速准则

1. **ISR 极速退出**：
   - ISR 内部严禁执行复杂运算、浮点计算、I/O 阻塞等待或调用 `vTaskDelay()`。
   - ISR 内部绝对禁止调用 `ESP_LOGx()` 或 `printf()`。
2. **ISR 专用 API 配对**：
   - 中断中通知任务必须使用带 `FromISR` 后缀的 FreeRTOS 原语，并处理高优先级任务唤醒：
     ```c
     static void IRAM_ATTR pir_gpio_isr_handler(void *arg) {
         BaseType_t high_task_wakeup = pdFALSE;
         uint32_t event = EVENT_PIR_TRIGGERED;
         xQueueSendFromISR(s_event_queue, &event, &high_task_wakeup);
         if (high_task_wakeup == pdTRUE) {
             portYIELD_FROM_ISR();
         }
     }
     ```

---

## 4. 照明核心算法与外设驱动规范

1. **LEDC 高频无频闪 PWM**：
   - PWM 频率必须配置为 $\ge 20\text{kHz}$，位深采用 13-bit（分辨率 0~8191），杜绝人耳可闻啸叫与摄影机条纹频闪。
2. **人眼非线性 Gamma 2.2 调光校正**：
   - 人眼对低亮度极其敏感。严禁线性映射 PWM；必须采用 Gamma 2.2 预计算查找表 (LUT)：
     $$\text{PWM\_Duty} = \left(\frac{\text{Brightness}}{100}\right)^{2.2} \times \text{MAX\_DUTY}$$
3. **CCT 双色温恒功率混色算法**：
   - 调节色温时保持总光通量恒定，避免暖光向冷光过渡时亮度出现突变凹陷：
     $$\text{Duty}_{\text{WW}} = \text{Total\_Duty} \times \frac{6500 - \text{CCT}}{6500 - 2700}, \quad \text{Duty}_{\text{CW}} = \text{Total\_Duty} - \text{Duty}_{\text{WW}}$$

---

## 5. NVS 持久化安全与磨损防抖

1. **2 秒防抖落盘机制**：
   - 滑动调光或高频色彩调节时，仅在内存中即时更新硬件 PWM；启动一个 2 秒单次软件定时器，待用户停止滑动 2 秒后再将最终状态写入 NVS。
2. **数据版本与校验**：
   - 存入 NVS 的结构体必须包含版本号与 CRC16 校验码，防止固件升级后读取到脏数据引发不可预知异常。
