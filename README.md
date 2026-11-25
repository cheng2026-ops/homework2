# 电控组第二次培训作业

- **程越**
- ***20263008143***

## 实现思路

使用 STM32CubeMX 配置 STM32F103C8T6 的三个 GPIO 为推挽输出，并将引脚用户标签设为 LED0、LED1、LED2（分别对应 PA6、PA7、PB0），生成代码后宏 `LED0_Pin`/`LED0_GPIO_Port` 等会自动写入 `main.h`，业务代码中只使用标签，不直接写引脚号，方便后续改板。

LED 采用低电平点亮接法（阳极经限流电阻接 3.3V，阴极接 GPIO），因此 `GPIO_PIN_RESET` 为亮、`GPIO_PIN_SET` 为灭。

主循环中按 LED0 → LED1 → LED2 的顺序轮流点亮，每次只让一个 LED 处于低电平、其余两个置高，形成周期 1.5 s 的三灯流水灯效果。

延时使用**非阻塞方式**实现：不调用会死等的 `HAL_Delay`，而是记录上一次切换的时刻 `led_tick`，每轮循环用 `HAL_GetTick()` 判断是否已过 500 ms，到了才切换 LED 并执行 `led_tick += 500`。这样主循环始终在高速运行，等待期间 CPU 可以执行其他任务；计时采用无符号减法，`HAL_GetTick()` 计数溢出时也不会出错。

## 核心代码

```c
static uint32_t led_tick = 0;   /* 上次切换 LED 的时刻 */
static uint8_t  led_index = 0;  /* 当前点亮的 LED 序号：0/1/2 */

led_tick = HAL_GetTick() - 500; /* 使第一次循环立即切换，避免上电后三灯同亮 */

while (1)
{
  if (HAL_GetTick() - led_tick >= 500)
  {
    led_tick += 500;

    /* 只有当前序号对应的 LED 拉低点亮，其余两个置高熄灭 */
    HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, (led_index == 0) ? GPIO_PIN_RESET : GPIO_PIN_SET);
    HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, (led_index == 1) ? GPIO_PIN_RESET : GPIO_PIN_SET);
    HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, (led_index == 2) ? GPIO_PIN_RESET : GPIO_PIN_SET);

    led_index = (led_index + 1) % 3;
  }

  /* 这里可以执行其他任务：延时期间 CPU 不会被死等阻塞 */
}
```

对应的 GPIO 初始化（CubeMX 生成，`MX_GPIO_Init`）：

```c
/*Configure GPIO pins : LED0_Pin LED1_Pin */
GPIO_InitStruct.Pin = LED0_Pin|LED1_Pin;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

/*Configure GPIO pin : LED2_Pin */
GPIO_InitStruct.Pin = LED2_Pin;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
HAL_GPIO_Init(LED2_GPIO_Port, &GPIO_InitStruct);
```

## 验收视频

- [验收视频.mp4](验收视频.mp4)：CLion 中编译通过并启动烧录，OpenOCD 控制台输出 `** Programming Started **`、`device id = 0x00006410`、`** Programming Finished **`，烧录成功后板上三灯按 500 ms 流水点亮。
- [演示视频.mp4](演示视频.mp4)：三灯流水灯运行近景。