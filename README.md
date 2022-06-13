# nycu_osdi_2022_final_project

This is a simple_ssd r/w simulator
## Report

### 1) code

https://github.com/13579and2468/osc2022

### 2) Implementation details

#### GC時機

使用變數 least_valid_count      --> 在所有使用中的block中，最少的valid_count是多少

在每次ftl_write的一開始，

1. free_block_number的數量是0
2. 剩下的空page數量剛好是least_valid_count
3. 被改寫的physical address所屬的nand,valid_count != least_valid_count 

若滿足所有上述條件就執行garbage collection.
理論上該條件也是必須要執行gc的最後時刻，因為在這次寫入之後若沒執行gc，則沒辦法完全搬完任何一個block的pages到空的位置.

#### GC

使用變數 least_valid_count_nand --> 在所有使用中的block中，有最少的valid_count的是哪一個block

在滿足GC時機的情況下，把least_valid_count_nand裡的所有page搬到剩下的free page，剛好會塞滿整個block，此時least_valid_count_nand裡面全部都是invalid，這時候erase掉least_valid_count_nand，就會多出一個free block，供接下來使用.

#### GC使用到的變數維護

每一次都會同時更新least_valid_count和least_valid_count_nand，

時機

1. 做ftl_write時, 把舊的pca變成invalid的時候 (直接比對大小是否要更新)
2. 做ftl_write時, 剛好用完一個block (直接比對大小是否要更新)
3. 做nand_erase之後 (重新掃所有block的invalid_count取最小)

### manage PCA state

沒有另外存這個state，因為可以藉由以下方式判斷 (順序比對)

▪ valid  -> check by P2L != INVALID_LBA
▪ empty  -> check by block is free || ( curr_pca.fields.lba < the_pca.fields.lba && curr_pca.fields.nand == the_pca.fields.nand )
▪ stale: data is useless but not erase yet -> 剩下的情況

### manage L2P table

logical address是存index， physical address是存nand(16bits)+page idx(lba)(16bits)的結構

在ftl_write或gc時皆會更新.
平常用來做mapping查找.


### manage P2L table

physical address 需要轉換成idx來查找 如 : `P2L[pca.fields.nand * PAGE_PER_BLOCK + pca.fields.lba]`
logical address 存lba (page idx)

平常ftl_write時會更新，gc時也會更新，gc時會被使用到

### read modify write

做在`ssd_do_write()`，對於寫入`L2P[lba]`存在的page，都要先讀取再把要寫入的部份覆蓋上去再一次寫入新page

### 3) 修改了什麼地方

1. 原來的ftl_write參數有一個很奇怪的 lba_rnage 用不到就拿掉了
2. ssd_do_write() 加入read modify write和串接 ftl_write()
3. ssd_do_read() 串接 ftl_read()
4. ftl_read()
5. ftl_write()
6. check_if_garbage_collect()
7. garbage_collect()
8. 增加變數 least_valid_count, least_valid_count_nand
9. nand_erase 加一行 free_block_number++;
10. NAND_LOCATION

### 4) evaluation

原來的 test1, test2 皆沒有GC

