# lab5 
## lab5 開發紀錄：  
### (1) 實做 printf:  
添加以下 code 到 `main.c` 之中：
```
int __io_putchar(int ch){
 uint8_t c=ch;
 HAL_UART_Transmit(&huart4, &c, 1, 100);
 return ch;
}
```  
用途是把 printf 底層實做 `__io_putchar` 重新導向到 UART  輸出
### (2)讓兩個 task 順利依序進行讀寫：  
一開始的狀態是他不會依序去做讀寫(task 1 寫入 task2 讀出), 但如果是 task2 先讀了就會發生錯誤, 所以這邊採用的方式是加入 semaphore 來做 synchronization:  
```  
void Task1(void *pvParameters) {
	uint8_t count=0;

	for (;;) {

		FIL test;
		if (f_open(&test, "TEST.TXT", FA_WRITE) != 0) {
			printf("open file err in task1\r\n");
		} else {
			printf("file open ok in task1\r\n");
		}

		char buff[13];
		memset(buff, 0, 13);
		if(count%2){
			sprintf(buff, "%s", "DataWriteT");
		}else{
			sprintf(buff, "%s", "DataWriteF");
		}
		UINT byteswrite;
		f_write(&test, buff, 12, &byteswrite);
		f_close(&test);
		count++;

		vTaskDelay(1);
		xSemaphoreGive( xSemaphore_2 );
		xSemaphoreTake(xSemaphore_1,portMAX_DELAY);
	}

}

void Task2(void *pvParameters) {
	while (1) {
		xSemaphoreTake(xSemaphore_2,portMAX_DELAY);
		FIL test_1;
		if (f_open(&test_1, "TEST.TXT", FA_READ) != 0) {
			printf("open file err in task2\r\n");
		} else {
			printf("file open ok in task2\r\n");
		}

		char buff[11];
		memset(buff, 0, 11);
		UINT bytesread;
		f_read(&test_1, buff, 11, &bytesread);

		printf("data read is %s in task 2\r\n", buff);

		f_close(&test_1);

		vTaskDelay(1);
		xSemaphoreGive( xSemaphore_1 );
	}
}
```  
### (3)播放音樂：  
這邊我們可以直接利用 API `AUDIO_PLAYER_Start()` 開始播放, 同時用`AUDIO_PLAYER_Process(1);`(state machine) 來管理播放的狀態：  
```  
void Task3(void *pvParameters) {
	for (;;) {
		AUDIO_PLAYER_Start(0);
		while(!isFinished)
		{
			AUDIO_PLAYER_Process(1);
			if (AudioState == AUDIO_STATE_STOP)
			 {
			    isFinished = 1;
			 }
		}
	}
}
```  
### (4)每五秒切歌：  
在 TIMER_6 每次進行 timer interrupt 時都會執行到 `HAL_TIM_PeriodElapsedCallback`, 因此我利用一個 "mytimer" 來計時, 當它等於 5000ms 時就會改變播放狀態(in waveplayer.c 把播放的 pointer 指向下一個 wav)  
```  
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
	/* USER CODE BEGIN Callback 0 */
	if(mytimer==5000)
	{
		AudioState =AUDIO_STATE_NEXT;
		mytimer=0;
	}
	mytimer++;
	/* USER CODE END Callback 0 */
	if (htim->Instance == TIM6) {
		HAL_IncTick();
	}
	/* USER CODE BEGIN Callback 1 */

	/* USER CODE END Callback 1 */
}
```
