---
title: CUBEMX+STM32+HAL库 非阻塞按键
date: 2026-05-21
lastMod: 2026-05-21T18:53:16.758Z
summary: 侧重于CUBEMX配置，使用闲置定时器实现长按、单击与双击
category: STM32
tags: [Button, cubemx, HAL, 定时器]
---

## 前言

区别于江协：通过计时器定时20ms**扫描**按键状态，本文使用GPIO**外部事件中断**来捕捉按键状态，使用**循环缓冲区**获取按键信息，最多支持**32**个按键

## 目标

- 非<abbr title="执行某段程序时，CPU因为需要等待延时或者等待某个信号而被迫处于暂停状态一段时间，程序执行时间比较长或者时间不定">阻塞</abbr>同时按键**灵敏**
- 模块高度**封装**且主程序调用**简洁**
- ~~ 长按时增加非线性 ~~

### **定时器**？外部中断？循环缓冲区？

定时器:即计数器，<abbr title="AHB High-speed Clock">HCLK</abbr>通过**预分频器**为定时器提供时钟脉冲，当计数器接收到上升沿是计数值加一，当到达设定**计数值**时归零同时触发**中断**

外部中断:当GPIO检测到上升沿或者下降沿时触发的中断(_通过外部中断可以避免反复扫描GPIO电平_)

[ **循环缓冲区**](https://www.bilibili.com/video/BV1p75yzSEt9/?spm_id_from=333.1387.upload.video_card.click&vd_source=0c6556f00d4c6d1a537b6b57612d11a6):数据写指针和读指针循环读写的一块区域(本项目使用了视频中循环缓冲区的简化版本，_micro 循环缓冲区_)

### CUBEMX配置

#### GPIO配置

_新建工程_（略）

设置按键GPIO为外部中断，设置触发模式为`External Interrupt Mode with Rising/Falling edge trigger detection`即上升/下降沿外部中断触发模式，更改引脚名称为KEYn

![图片描述](/Button/CUBEMX1.png 'GPIO配置')

在<abbr title="Nested Vectored Interrupt Controller嵌套向量中断控制器">`NVIC`</abbr>中开启**GPIO中断**，框中打勾，由于我的按键引脚为PC0与PB6，对应`EXIT line0 interrupt`与`EXIT line[9,5] interrput`

![图片描述](/Button/CUBEMX2.png 'GPIO中断配置')

#### 定时器配置 `预分频系数` & `计数值`

`Timers`-`TIMn`,选择自己空闲的定时器，n为序号，**Clock Source**选择`Internal Clock`

$$
T_c =\frac{V_P · V_c}{f_H}
$$

$T_c$为**计数周期**，$f_H$为**时钟线HCLK频率**，$V_P$为**为分频系数Prescaler**+1,$V_C$为**计数值Counter Period**+1

![图片描述](/Button/CUBEMXTIME1.png '时钟配置')

- 理想按键扫描间隔为20ms，示例**MCUF407**，主频$f_H$为168MHz,追求性能设置计数周期$T_c$为10ms
- 计算得到$V_P·V_C=1680000$，只涉及计数周期时$V_P$与$V_C$两者没有区分的必要(**使用PWM时$V_C$将作为分配占空比的参照**)
- 为了 美观+直观 ,`Prescaler`填入168-1，`Counter Period`填入10000-1（**两者数值都不要超过$2^{16}-1$**）

点击`NVIC Setting`，开启`TIM global interrupt`,`Sub Priority`数值影响不大

### 程序调用

1. 先引入Button.c与Button.h文件

2. 在Button.h文件中修改BUTTON_NUM

3. 在Button.c文件中修改按键检测信息获取的if else，在定时器中断回调函数中更改定时器中断

4. 在main.c中初始化缓冲区

```main.c
uint8_t ButtonBuffer[BUTTON_BUFFER_LENGTH]={0};//按键缓冲区
uint8_t ButtonIndex[BUTTON_INDEX_LENGTH]={0};//按键缓冲索引区
uint8_t* p_ButtonBuffer=ButtonBuffer;
```

5. 初始化按键响应函数，通过扩充case来增加按键数量

```main.c
void ReaKey() {
  /*ab=Read_ab();
  OLED_ShowNum(10*8,2,ab,4,16);//按鍵调试*/
  for (int j=0;j<BUTTON_RECEIVE_NUM;j++) {//处理按键事件循环
     if (ButtonIndex[0]!=ButtonIndex[1]) {//当读索引不等于写索引时，即缓冲区有新数据
       uint8_t Button_Info=0;
       Button_Info=p_ButtonBuffer[ButtonIndex[0]];//提取数据
       ButtonIndex[0]++;//读索引后移
       ButtonIndex[0]%=BUTTON_BUFFER_LENGTH;//防止超出缓冲区上限

       switch (Button_Info>>3) {//获取按键序号
         case 0: {//按键1
           switch (Button_Info%8) {
             case 1: {//按键按下瞬间

             }
               break;
             case 2: {//按键松开瞬间

             }
               break;
             case 3: {//单击事件
               if (PageNum==0&&SettingFlag==0&&CountStep==0) {//在重量页面下才去皮
                 reset = value;//去皮
               }else if (PageNum==0&&SettingFlag==0&&CountStep!=0) {//在计数模式下取消计数物品
                 CountStep=0;
                 weight_Count=0;
               }
             }
               break;
             case 4: {//双击事件

             }
               break;
             case 5: {//长按事件

             }
               break;
             case 6: {//重复事件
               Servo_Angle+=10;
               Servo_Angle%=1800;
             }
               break;
             default: ;
           }
         }
           break;
         case 1: {//按键2
           switch (Button_Info%8) {
             case 1: {//按键按下瞬间

             }
               break;
             case 2: {//按键松开瞬间

             }
               break;
             case 3: {//单击事件

             }
               break;
             case 4: {//双击事件

             }
               break;
             case 5: {//长按事件

             }
               break;
             case 6: {

             }
               break;
             default: ;
           }

         }
           break;
         default: ;
       }
     }else {
    break;
  }
  }
}
```

### 程序设计

![图片描述](/Button/ButtonDesign.jpg '按键状态区分')

![图片描述](/Button/Button_Process.png '状态流程图')

状态流程图直接仿照江协，但是并没有使用按键标志位，而是直接将事件信息写入micro缓冲区，主循环直接读取缓冲区来执行相应按键操作

micro缓冲区写入程序

```Button.c
void WriteDate(const uint8_t Date) {
	if (B_Index[0]-B_Index[1]!=1) {//如果写索引比读索引大一，就说明缓存区已经满了
		B_Buffer[B_Index[1]]=Date;//写入数据
		B_Index[1]++;//写索引后移
		B_Index[1]%=BUTTON_BUFFER_LENGTH;
	}
}
```

读程序

```main.c
for (int j=0;j<BUTTON_RECEIVE_NUM;j++) {
     if (ButtonIndex[0]!=ButtonIndex[1]) {
       i=p_ButtonBuffer[ButtonIndex[0]];
       ButtonIndex[0]++;//读索引后移
       ButtonIndex[0]%=BUTTON_BUFFER_LENGTH;
       //解析数据代码……//
    }
}
```

按键检测函数

```Button.c
void Key_Detector(const uint16_t GPIO_Pin) {//
	uint8_t Button_Label=0;//i为按键序号
	uint8_t Button_Edge=0;//j为1下降沿或0上升沿
	//按键检测信息获取
	if (GPIO_Pin==KEY1_Pin) {
		Button_Label=0;
		if (HAL_GPIO_ReadPin(KEY1_GPIO_Port,KEY1_Pin)==GPIO_PIN_RESET)
			Button_Edge=1;//否则B_j默认为零
	}else if (GPIO_Pin==KEY2_Pin) {
		Button_Label=1;
		if (HAL_GPIO_ReadPin(KEY2_GPIO_Port,KEY2_Pin)==GPIO_PIN_RESET)
			Button_Edge=1;//否则B_j默认为0
	}

	//处理按键信息
	if (Key_UpdateTime[Button_Label]==0) {//按键更新冷却好了
		if (Button_Edge==1&&(Key_Previous&1<<Button_Label)==0) {//下降沿
			WriteDate(1+(Button_Label<<BUTTON_INFO_BYTE));//按键按下数据存入缓冲区
			Key_Hold|=(1<<Button_Label);//按住不放
			Key_UpdateTime[Button_Label]=KEY_UPDATE_TIME;//冷却时间
			Key_Previous |= (1 << Button_Label);  // 使用 OR 操作设置位
		}else if (Button_Edge==0&&(Key_Previous&(1<<Button_Label))!=0) {//上升沿
			WriteDate(2+(Button_Label<<BUTTON_INFO_BYTE));//按键松开数据存入缓冲区
			Key_Hold&=~(1<<Button_Label);//
			Key_UpdateTime[Button_Label]=KEY_UPDATE_TIME;
			Key_Previous &= ~(1 << Button_Label);  // 使用 AND 操作清除位
		}
	}
}
```

按键分析函数，通过状态解析出触发事件

```Button.c
void Key_State_Analyze(void) {//
	for (uint8_t n=0;n<BUTTON_NUM;n++) {
		if (Key_Time[n]) {//如果开启按键计时
			Key_Time[n]--;//时间减10ms
		}
		//按键冷却刷新
		if (Key_UpdateTime[n]!=0) {
			Key_UpdateTime[n]--;
		}
		//按键处理层，根据检测信息推测按键状态
		//识别状态
		if (Key_State[n]==0) {//状态0
			if (Key_Hold&(1<<n)) {//如果按键保持按下
				Key_Time[n]=KEY_LONG_TIME;//等待长按时间
				Key_State[n]=1;//进入状态1
			}
		}else if (Key_State[n]==1) {//状态1
			if ((Key_Hold&(1<<n))==0) {//如果按键已经松开
				Key_Time[n]=KEY_DOUBLE_TIME;//设置双击阈值时间
				Key_State[n]=2;//进入状态2
			}else if (Key_Time[n]==0) {//如果长按阈值达到
				Key_State[n]=4;//进入状态4
				WriteDate(5+(n<<BUTTON_INFO_BYTE));//长按事件数据存入缓冲区
				Key_Time[n]=KEY_REPEAT_TIME;
			}
		}else if (Key_State[n]==2) {//状态2
			if (Key_Hold&(1<<n)) {
				Key_State[n]=3;//进入状态3
				WriteDate(4+(n<<BUTTON_INFO_BYTE));//双击事件数据存入缓冲区
			}else if (Key_Time[n]==0) {//错过双击时间
				Key_State[n]=0;//回归状态0
				WriteDate(3+(n<<BUTTON_INFO_BYTE));//单击事件数据存入缓冲区
			}
		}else if (Key_State[n]==3) {//状态3
			if ((Key_Hold&(1<<n))==0) {//如果按键松开
				Key_State[n]=0;//回归状态0
			}
		}else {//状态4
			if ((Key_Hold&(1<<n))==0) {//如果按键松开
				Key_State[n]=0;//回归状态0
			}else if (Key_Time[n]==0) {
				WriteDate(6+(n<<BUTTON_INFO_BYTE));//重复事件数据存入缓冲区
				Key_Time[n]=KEY_REPEAT_TIME;//重新计时
			}
		}
	}
}
```

## 总结

这次按键的设计充分使用了**数电知识** ，简化了设计流程，通过**数据缓冲区**将主程序与检测程序隔离，设计更加方便
