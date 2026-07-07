# 类freertos的rtos概览

## 关键组件

### portmacro.h

1. 规定常用数据类型的代称; 
2. 定义与体系结构相关的宏(如寄存器位置等)和函数, 主要作用是任务挂起和切换; 
3. ...

### rtos_config.h

1. 定义芯片和系统的时钟频率; 
2. 定义中断优先级, 主要是BSASEPRI(高四位有效, 优先级大于或等于高四位值的中断都会被屏蔽)的值; 
3. ...

### projectdefs.h

1. 定义`TaskFuntion_t`函数指针; 
2. 定义`true`和`false`; 

### 