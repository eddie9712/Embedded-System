# Lab1 description:
1.Must use MultiTask, e.g.  One for LED ,  other for Button.  
2.Inter process communication (IPC).  
3.LED have two state  
(1)The green LED lights up for 5 seconds, then turns to a red LED lights up for 5 seconds (green LED off), and then switches back to the green LED lights up for 5 seconds (red LED off), and so on.  
(2)The red LED blinking.If the button is pressed, the LED will switch to another state.  
4.Debounce.Edge detection.  

# lab1-traffic-light
## 開發紀錄: 
## First Version:
一開始寫好兩個 state 之後,是採用兩個 tasks 然後把兩個 state 寫在第一個 task 並且用一個 flag 作為切換 state 的判斷依據, 而另一個 task 方面則是利用 polling 的方式偵測 button 
 按下的 event.
```
xTaskCreate(ButtonHandler,"task1",1024,(void *)1,1,NULL);
xTaskCreate(LEDHandler,"task2",1024,(void *)1,1,NULL);
.....

void LEDHandler(void *pvParameter)
{
	int flag=0;
	for(;;)
	{
	   //first state
	   xQueueReceive(xQueue1,&flag,0);
	   if(flag==1)
	   {
		   HAL_GPIO_WritePin(GREEN_LED_GPIO_Port,GPIO_PIN_12,GPIO_PIN_SET);
		   vTaskDelay(5000);
		   HAL_GPIO_WritePin(GREEN_LED_GPIO_Port,GPIO_PIN_12,GPIO_PIN_RESET);
		   HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_SET);
		   vTaskDelay(5000);
		   HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_RESET);
	   }
	   else if(flag==2)
	   {
		   HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_SET);
		   vTaskDelay(1000);
		   HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_RESET);
		   vTaskDelay(1000);
	   }
	 }
}
void ButtonHandler(void *pvParameter)
{
	int mes=0;b
	for(;;)
	{
	   if(HAL_GPIO_ReadPin(BLUE_BUTTON_GPIO_Port,GPIO_PIN_0))
	  {
		  mes=(mes == 1 ? 2 : 1);
		  xQueueOverwrite( xQueue1, &mes );
	  }

	}
}
```
但以上的作法有非常大的問題, 首先是兩個 task 的執行;雖然兩個 task 擁有相同的 priority 但是我並沒有設置 `TIME_SLICING` 這個參數, 導致 `button handler` 不會在每一個 tick 的
時候都切換 task(所以按按鈕的 event 可能不會被偵測), 接著是就算如果真的被偵測到了它也無法做即時的切換(必須等到 task 執行完後才會依據 queue 收到的內容去改 flag), 最後還有 button debounce 上的問題也
尚未處理.  

## Final Version:
為了準確監測 button 的事件, 我設定了 `TIME_SLCING` 的參數(在每個 tick 都會換 task 執行一段 time slicing) 並且利用 `vTaskDelay()` 做 button debounce, 但在即時切換的部份
我把這兩個 state 寫到不同的 tasks 並且加入 `vTaskSuspend()` 和 `vTaskResume()` 進行 mutual exclusive 的控制, 並且把 queue 的 message 做為執行權的給予(一開始他們會同時執行
但都被 `mes` 卡住, 然後 button 按下後會讓其中一個 state 先執行):

```
Queue1=xQueueCreate(1,sizeof(int));     //create a queue

  xTaskCreate(ButtonHandler,"task1",1024,(void *)1,2,NULL);
  xTaskCreate(state1,"task2",1024,(void *)1,1,&Handle2);    //for blinking the LED
  xTaskCreate(state2,"task3",1024,(void *)1,1,&Handle3);    

  vTaskStartScheduler();
  
  ....
  void state2()
{
  int mes2=0;
  for(;;)
  {
    xQueueReceive(xQueue1,&mes2,0);
    if(mes2 == 2)
    {
       HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_SET);
       vTaskDelay(1000);
       HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_RESET);
       vTaskDelay(1000);
    }
  }
}
void state1(void *pvParameter)
{
	int mes1=0;
	for(;;)
	{
	   xQueueReceive(xQueue1,&mes1,0);
	   if(mes1 == 1)
	   {
		   HAL_GPIO_WritePin(GREEN_LED_GPIO_Port,GPIO_PIN_12,GPIO_PIN_SET);
		   vTaskDelay(5000);
		   HAL_GPIO_WritePin(GREEN_LED_GPIO_Port,GPIO_PIN_12,GPIO_PIN_RESET);
		   HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_SET);
		   vTaskDelay(5000);
		   HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_RESET);
	   }
	 }
}
void ButtonHandler(void *pvParameter)
{
   int mes=1;
	for(;;)
	{
	     while(!HAL_GPIO_ReadPin(BLUE_BUTTON_GPIO_Port,GPIO_PIN_0)){
			    vTaskDelay(5);
			}
			vTaskDelay(60);
			if(HAL_GPIO_ReadPin(BLUE_BUTTON_GPIO_Port,GPIO_PIN_0)){
				if(mes==1)
				{
					mes=2;
					vTaskSuspend(xHandle2);
					HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_RESET);
					HAL_GPIO_WritePin(GREEN_LED_GPIO_Port,GPIO_PIN_12,GPIO_PIN_RESET);
					vTaskResume(xHandle3);
				}
				else if(mes==2)
	        	{
					mes=1;
					vTaskSuspend(xHandle3);
					HAL_GPIO_WritePin(RED_LED_GPIO_Port,GPIO_PIN_14,GPIO_PIN_RESET);
					HAL_GPIO_WritePin(GREEN_LED_GPIO_Port,GPIO_PIN_12,GPIO_PIN_RESET);
                    vTaskResume(xHandle2);

		        }
				xQueueOverwrite( xQueue1, &mes);
			}
			   vTaskDelay(60);

	  }
}
```
