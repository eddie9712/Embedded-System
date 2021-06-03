# lab3

## 開發紀錄  
一開始先 create 4 個 tasks 並設定相關的 priority:
```
  xTaskCreate(TaskMonitor_App,"TaskMonitor",130,NULL,2,NULL);  //create task monitor
  xTaskCreate(green_blink,"green_blink",130,NULL,1,NULL);  //green_led blinking
  xTaskCreate(red_blink,"red_blink",130,NULL,1,NULL);  //red_led blinking
  xTaskCreate(task_delay,"task_delay",130,NULL,14,NULL);  //task_delay
```  
而 taskmonitor 是用來取得現在 task 的相關資料, 其他 task 做為 task 狀態的樣本,  
```
void TaskMonitor_App(void *pvParameters){
	int i = 500;
	    TaskMonitor();
	    i += 1;
	    vTaskDelay(i);

	}
}
```
然後我們把它 api 的 implementation 寫在 tasks.c 之中,在我們要取的 tasks 相關資訊時會進行以下的動作：  
### 1. 第一部份: 
先準被相關資料結構,包括從 TCB 拿出 data 所放的`struct data`,存取 linked list 用的 pointer`tskTCB *ptr2 ,ListItem_t *ptr;` 和把 data 傳到 USART 的 buffer(XX_monitor)：  
```
char MonitorTset[70];
char task_monitor[70];
char idle_monitor[70];
char red_monitor[70];
char green_monitor[70];
char delay_monitor[70];
struct data{
   int priority_base;
   int priority_actual;
   char  pstack_m[12];
   char  topofpstack_m[12];
   char  state_m[10];
};
struct data taskmonitor;
struct data redblink;
struct data greenblink;
struct data taskdelay;
struct data idletask;
tskTCB *ptr2;
 ListItem_t *ptr;
```  
### 2. 第二部份:
接著是開始把相關的 tasks data 從代表不同 state 的 linked list 拿出, 這邊我是根據不同的 list 搭配 for loop 去 traverse 每個 tasks(同時搭配兩個 pointers :拿 list 的 item  
和 指向 TCB 的 pointer)  
```
 ptr = (&(&(pxReadyTasksLists[i]))->xListEnd)->pxNext;
 ptr2=ptr->pvOwner;
```  
在可以 access task 相關的資訊後, 我把它們存到一開始建立的 data 中(並看它們是在哪個 list 決定它們是什麼 state)  
* e.g.  
```

		                   	   redblink.priority_actual=(int)ptr2->uxPriority;
		                   	   redblink.priority_base=(int)ptr2->uxBasePriority;
		                   	   Uint32ConvertHex(ptr2->pxStack,redblink.pstack_m);
		                   	   Uint32ConvertHex(ptr2->pxTopOfStack,redblink.topofpstack_m);
		                   	   strcpy(redblink.state_m,"blocked2");
```
### 3. 第三部份: 
在取得了 task 的資訊後, 利用 `sprintf()` 把要傳的 data 放到 char array 的 buffer 中, 在利用 USART 的 api 傳到 USART 端口:
```
HAL_UART_Transmit(&huart2,(uint8_t *)MonitorTset,strlen(MonitorTset),0xffff);
```
