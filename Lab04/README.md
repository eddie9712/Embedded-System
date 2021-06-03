# lab4
## lab4 開發紀錄  
### step1: 顯示 freelist 相關資訊  
總共要顯示六個欄位分別是 free list 中  free block 的：  
1. `startaddress`:  
這個欄位可以藉由去 traverse 每一個 nodes 中的 `pxNextFreeBlock` 欄位來獲得(每個都會指向下一個 free block 的 start address)
2. `heapSTRUCT_SIZE`:  
固定為 8(包含指向下一個 free block 的 pointer 和 blocksize 大小)   
3. `xBlockSize`  
這個欄位可以直接從每個 nodes 中的 `xBlockSize` 拿到個別的 free block 的 size
4. `endaddress`  
利用 `startaddress` 加上 `xBlockSize` 就可以得到 free block 的 end addres  
* 5,6 欄位分別是 `xFreeBytesRemaining`,`configADJUSTED_HEAP_SIZE` 兩個都可以藉由題供的 macro 取得  
這邊簡單描述如何去 traverse free list, 首先因為 free list 中第一個 head node 是 `xStart` 所以我們用一個 pointer 指向它： `BlockLink_t *ptr=&xStart;`,  
這邊要注意的是, 因為 node 中指向下一個 node(free block) 的 pointer 是下一個 free block 的 start address, 所以存取要格外小心, 還有 end address 是 start address  
加上 block size, 但一個是 address 一個是數值, 所以我把兩個都轉成 `BaseType_t` 再相加
```
void vPrintFreeList( void )
{
	BlockLink_t *ptr=&xStart;
	memset(print_list,'\0',sizeof(print_list));
	sprintf(print_list,"startAddress  |heapSTRUCT_SIZE  |xBlocksize  |endAddress\n\r");
	HAL_UART_Transmit(&huart2,(uint8_t *)print_list,strlen(print_list),0xffff);
	while(ptr->pxNextFreeBlock!=&xEnd)
	{
		memset(print_list,'\0',sizeof(print_list));
		mydata.blocksize=ptr->pxNextFreeBlock->xBlockSize;
		sprintf(print_list,"%p  %d  %d %p\n\r",ptr->pxNextFreeBlock,8,mydata.blocksize,((BaseType_t)ptr->pxNextFreeBlock)+((BaseType_t)mydata.blocksize));
		HAL_UART_Transmit(&huart2,(uint8_t *)print_list,strlen(print_list),0xffff);
		ptr=ptr->pxNextFreeBlock;
	}
	sprintf(print_list,"configadjusted_heap_size:%d xfreebytesremaining:%d  \n\r",configADJUSTED_HEAP_SIZE,xFreeBytesRemaining);
    HAL_UART_Transmit(&huart2,(uint8_t *)print_list,strlen(print_list),0xffff);
}
```
### step2:  
step2 的部份主要就是要實踐 merge 的演算法, 也就是把兩個相鄰的 free block merge 成一塊大塊的 free block 再依大小排序:  
為了要達到 merge 的做法, 我的想法是 **如果 insert 到 free list 的 block 的 start address 等於某塊的 end address 或
是 insert 到 free list 的 block 的 end address 等於某塊的 start address 那麼那兩塊就是可以做 merge 的 blocks**, 因此我們可以去 traverse 每個 nodes 找
出可以 merge 的 nodes 再把它們合起來(改變 blocksize 就好), 最後把合起來的 block 看成新的一塊重新插入, 實做如下：  
```
#define prvInsertBlockIntoFreeList( pxBlockToInsert )                                                                               \
    {                                                                                                                               \
        BlockLink_t * pxIterator,* ptr;                                                                                             \
        ptr=pxBlockToInsert;                                                                                                        \
                                                                                                                                    \
        pxIterator=&xStart;                                                                                                         \
        while(pxIterator!=&xEnd)                                                                                                    \
        {                                                                                                                           \
        	if((BaseType_t)ptr==((BaseType_t)(pxIterator->pxNextFreeBlock)+(BaseType_t)(pxIterator->pxNextFreeBlock->xBlockSize)))  \
			{                                                                                                                       \
        		(pxIterator->pxNextFreeBlock->xBlockSize)+=ptr->xBlockSize;                                                         \
                 ptr = pxIterator->pxNextFreeBlock;                                                                                 \
                 pxIterator->pxNextFreeBlock=pxIterator->pxNextFreeBlock->pxNextFreeBlock;                                          \
            }                                                                                                                       \
            else if(((BaseType_t)ptr+(BaseType_t)(ptr->xBlockSize))==((BaseType_t)(pxIterator->pxNextFreeBlock)))                   \
            {                                                                                                                       \
               ptr->xBlockSize+=pxIterator->pxNextFreeBlock->xBlockSize;                                                            \
               pxIterator->pxNextFreeBlock=pxIterator->pxNextFreeBlock->pxNextFreeBlock;                                            \
            }                                                                                                                       \
            pxIterator=pxIterator->pxNextFreeBlock;                                                                                 \
        }                                                                                                                           \
        /* Iterate through the list until a block is found that has a larger size */                                                \
        /* than the block we are inserting. */                                                                                      \
        for( pxIterator = &xStart; pxIterator->pxNextFreeBlock->xBlockSize <= ptr->xBlockSize; pxIterator = pxIterator->pxNextFreeBlock ) \
        {                                                                                                                           \
            /* There is nothing to do here - just iterate to the correct position. */                                               \
        }                                                                                                                           \
                                                                                                                                    \
        /* Update the list to include the block being inserted in the correct */                                                    \
        /* position. */                                                                                                             \
        ptr->pxNextFreeBlock = pxIterator->pxNextFreeBlock;                                                             \
        pxIterator->pxNextFreeBlock = ptr;                                                                              \
    }
```
