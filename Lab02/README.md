# Lab02 description:
* In this lab, we will detect the motion using the motion sensor on STM32F407VG.
* We have the green light blinking initially yet if we shake the board, ISR(switch the red LED's state) is triggered when the motion detected,  
and then ISR will unblock the handler task(blinks the orange LED for five times).Simutaneously, we need to disable interrupt when the handler task is executing.

## Lab02 內容:
這次的 lab 是要主要是利用 STM32 上的 motion sensor 偵測加速度來觸發 interrupt, 同時使用 FREERTOS 的 semaphore 來控制 task 的 synchronization.
## 開發紀錄：
在一開始進行實驗時, 因為 interrupt callback 的 function name 寫錯, 導致 interrupt 一直無法順利觸發, 然而成功觸發 interrupt 之後面臨到的是 interrupt 無法觸發第二次, 在看了
助教給的 slide 以及官方 document 之後, 才知道要去 read register `OUTS1 (5Fh)` 來 enable interrupt ` MEMS_Read(0x5f,&data);`. 
```
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)   //ISR function
{
       uint8_t data;
       if(HAL_GPIO_ReadPin(GPIOD,GPIO_PIN_14))
       {
           HAL_GPIO_WritePin(GPIOD,GPIO_PIN_14,GPIO_PIN_RESET);
       }
       else
       {
           HAL_GPIO_WritePin(GPIOD,GPIO_PIN_14,GPIO_PIN_SET);
       }
       MEMS_Read(0x5f,&data);
```
接著我 create 兩個 tasks 一個是閃綠燈(low priority)的 task, 另外一個是閃橘燈的 task(handler, high priority), 然後在進到 handler 時執行 semaphore take:
```
                                //interrupt handler
for(;;)                         
	 {
	  xSemaphoreTake(xSemaphore,portMAX_DELAY);
	  int i=0;
	  for(;i<5;i++)
	  {
	    HAL_GPIO_WritePin(GPIOD,GPIO_PIN_13,GPIO_PIN_SET);
	    vTaskDelay(1000);
	    HAL_GPIO_WritePin(GPIOD,GPIO_PIN_13,GPIO_PIN_RESET);
	    vTaskDelay(1000);
	  }
	 }
```
然而以上並不足以做到說橘燈在閃的時候不能有 interrupt, 所以我們必須要在 handler task 執行時 disable interrupt, 所以我把 interrupt 的再次 enable 寫到橘燈閃完之後,就可以了：
```
for(;;)                         //interrupt is disabled until we execute MEMS_READ
	 {
	  xSemaphoreTake(xSemaphore,portMAX_DELAY);
	  int i=0;
	  for(;i<5;i++)
	  {
	    HAL_GPIO_WritePin(GPIOD,GPIO_PIN_13,GPIO_PIN_SET);
	    vTaskDelay(1000);
	    HAL_GPIO_WritePin(GPIOD,GPIO_PIN_13,GPIO_PIN_RESET);
	    vTaskDelay(1000);
	  }
    MEMS_Read(0x5f,&data);    
	 }
```
