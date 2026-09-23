---
title: 步进台钳
date: 2026-07-06
lastMod: 2026-07-06T18:26:20.758Z
summary: 一个实用的电动台钳
category: 日常
tags: [pliers, electricity]
---

## 前言

为了满足日常焊接需求，固定PCB板，因而制作电动台钳

熟悉步进电机，同时实用多余的丝杆步进电机，为日后的贴片机做准备

## 前置知识（定时器？）

定时器的输出比较翻转：当计数器高于脉冲值时输出高电平，当定时器计数值等于计数值时置零同时输出低电平

## 目标

### 基础部分

- 步进丝杠恒定速度前进与后退
- 步进电机旋转指定距离
- 切换速度挡位

### 进阶部分

- 自动限位，当卡好板子以后自动停止

## 原理与程序框架

### 步进电机的控制框架

整个系统分 四层：硬件信号 → 定时器脉冲 → 速度换算 → 加速斜坡。

---

#### 硬件信号（2 根线）

通过主控F407的PA0与PA1连接TMC2209的STEP与DIR，当DIR为H正传，DIR为L反转

PA0 (STEP)：TIM2_CH1 输出比较翻转模式，自动产生 50% 占空比方波。每个完整方波周期 = 1
个微步脉冲。纯硬件产生，无中断、无 CPU 参与。

PA1 (DIR)：普通 GPIO，HAL_GPIO_WritePin 直接控制转动方向。

#### TIM2 如何产生脉冲（输出比较翻转模式）

TIM2 (84MHz, APB1×2)

CNT: 0 ──→ 26250（PA0翻转） ──→ 52499 ──→ 0（PA0归零） ──→ ...

CubeMX 初始化为起步速度（1 mm/s → 1600 Hz），之后 Stepper_Ramp() 动态改 ARR/CCR 来加速。ARPE=Enable 保证运行时修改 ARR不会产生毛刺。

---

#### 速度换算（mm/s → Hz → ARR）

这是最核心的数学关系：

线速度 → 转速: 转速 = 线速度 / 导程
eg: 50 mm/s ÷ 8 mm/rev = 6.25 rev/s

转速 → 脉冲频率: 脉冲频率 = 转速 × 全步/圈 × 细分
eg: 6.25 × 200 × 64 = 80,000 Hz

脉冲频率 → ARR: ARR = TIM2时钟 / 脉冲频率 − 1
eg: 84,000,000 / 80,000 − 1 = 1049

---

#### 加速斜坡（为什么需要）

步进电机从零直接跳到高速会失步（转子跟不上旋转磁场，表现为振动不转）。解决办法是逐步加速：

![图片描述](/pliers/斜坡加速.png '加速斜坡')

Stepper_Ramp() 在主循环中被调用，内部用 HAL_GetTick() 做时间间隔控制（每 RAMP_UPDATE_MS 执行一次），逐步把 ramp_speed从 START_SPEED_MM_S 推到 TARGET_SPEED_MM_S。

### CUBEMX中配置

首先开启DIR引脚，同时打开定时器

（示例启用定时器2）左侧 Timers → 勾选 TIM2，Channel1 选 Output Compare CH1

| 参数                |       值        |                           说明 |
| :------------------ | :-------------: | -----------------------------: |
| Prescaler           |        0        |                  不分频，84MHz |
| Counter Mode        |       Up        |                       向上计数 |
| Counter Period      |      52499      | ARR = 84M/1600-1（起步 1mm/s） |
| Auto-reload preload |     Enable      |               允许运行时改速度 |
| CH1 OC Mode         | Toggle on match |             比较匹配时翻转 PA0 |
| CH1 Pulse           |      26250      |                     50% 占空比 |

![图片描述](/pliers/CUBEMXTIM2配置.png 'TIM2配置')
