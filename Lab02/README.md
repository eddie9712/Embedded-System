# Lab02 description:
* In this lab, we will detect the motion using the motion sensor on STM32F407VG.
* We have the green light blinking initially yet if we shake the board, ISR(switch the red LED's state) is triggered when the motion detected,  
and then ISR will unblock the handler task(blinks the orange LED for five times).Simutaneously, we need to disable interrupt when the handler task is executing.
