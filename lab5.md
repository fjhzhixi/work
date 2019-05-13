# lab5做了什么

1. 基于磁盘的硬件构成实现通过`I/O`接口**驱动**来与磁盘外设进行交互(主要是**屏蔽丑陋的硬件,提供统一的读写接口**)
2. 基于`I/O`交互以及磁盘的空间搭建文件系统结构(本质上就是用**数据结构**管理磁盘的空间)
3. 基于文件系统来向用户提供操作的接口(以**系统调用**的形式)

# 具体实现

## 外部存储设备驱动

这部分很依赖于底层硬件的构成标准

### 磁盘寻址的原理

~~这部分我们OS实现不用考虑权当复习理论课了~~

#### 地址表达

一个扇区大小(**有效数据大小为一块即512字节**)为最小的数据交换单元

* 物理磁盘使用柱面,磁道,扇区构成**物理地址**来定位一个块, 即

  **某一磁盘寻址依赖于** : (括号中均为常量)

  1. 柱面号`C_index`(一共有`C_num`个柱面)
  2. 磁头号`H_index`(一共有`H_num`层磁头)
  3. 扇区号`S_index`(一个磁道有`S_num`个扇区)

  ![](OS-lab5\disk.png)

* 而**逻辑地址**则将所有扇区抽象成连续的块结构**通过偏移获取**

  **抽象规则为 : `offset = C_index * H_num + H_index * S_num + S_index`**

#### 地址转换

1. OS发出的请求 : 盘号`diskno` + 偏移`offset`
2. 由盘号定位到某一磁盘接受之后寻址 : 
   * **由逻辑地址向物理地址转换 : **
     1. `C_index = offset / (H_num * S_num)`
     2. `H_index = (offset % (H_num * S_num)) / S_num`
     3. `S_index = (offset % (H_num * S_num)) % S_num`

### OS与外设交互的机制

**通过读写设备上的寄存器来进行数据通信** :

1. 外设寄存器也称为 I/O 端口，我们使用 I/O 端口来访问 I/O 设备
2. 外设寄存器通常包括控制寄存器、状态寄存器和数据寄存器
3. **这些硬件 I/O 寄存器被映射到指定的物理内存空间(理论上你可以选择映射到任何内核虚拟空间,但由于无缓存我们选择映射到`kseg1`),即当我们访问这段物理地址时,`CPU`就会知道其对应的实际物理空间对应的不是物理内存,而是外设中的外设寄存器,例如在读数据时外设将数据写入其内部的存储空间,而我们从`kseg1`中直接按虚拟地址读取就可以了**

**故我们读写某一固定的物理内存(固定值取决于外设种类对应的规定)就相当于读写外设的寄存器这样我们就能控制其行为**

### IDE磁盘在硬件层次的规定

* 其映射到的物理内存区域是`[0x13000000, 0x13004200)`

* 在CPU访问时使用的只能是虚拟地址,所以我们要把这块区域**映射到内核空间的`kseg1`中(因为这片空间是内核中的,映射固定(加偏移即可),并且数据不会被缓存**,`kesg`的起始地址为`0xA0000000 `,**所以磁盘外设寄存器对应的虚拟地址为`[0xB3000000, 0xB3004200)`**

  注意这个映射到的虚拟地址是我们选择的,基于**内核空间所有用户进程都可以访问以及这片虚拟内存不存在缓存**

  **注意在此处涉及内核虚拟地址向物理地址的转换(由硬件实现),其规则如下(~~我猜的~~) : **

  * 对`kseg0`中的内核虚拟地址通过最高位清0 : 所以我们在之前通过加`0x8000_0000`实现物理地址映射到`kseg0`
  * **对`kseg1`中的内核虚拟地址通过最高3位清0 : 所以`[0xB3000000, 0xB3004200)`即对应物理地址`[0x13000000, 0x13004200)`**

* 磁盘各个外设寄存器的意义 : 

  ![](OS-lab5\IDE-regs.png)

  注意实际上我们只有一块磁盘,所以`IDE ID`恒为0

* 所以我们的对磁盘的`I/O`操作就如下图

  ![](OS-lab5\w-r.png)

**注意此处有一点我没有想通的机制 : 就是我们认为从磁盘中读取数据和向磁盘写入数据都是瞬间完成的,没有延时,所以没有实现`CPU`轮询状态位或者磁盘通知`CPU`中断类似的机制,我猜时因为我们的外设磁盘是仿真模拟的,所以讲问题简化了**

### 具体实现

1. 实现**用户态对外设寄存器(表现为一段物理地址内存)进行读写的系统调用**

```c
//lib/syscall_all.c
int sys_write_dev(int sysno, u_int va, u_int dev, u_int len) {
    if ((dev >= 0x10000000 && (dev + len - 1) < 0x10000020) ||
        (dev >= 0x13000000 && (dev + len - 1) < 0x13004200) ||
        (dev >= 0x15000000 && (dev + len - 1) < 0x15000200)) {
        u_long dev_va = dev + 0xa0000000;//将设备物理地址映射到内核kseg1空间
        bcopy(va, dev_va, len);//将va对应的虚拟内存地址之后长len字节的数据写入设备物理地址dev
        return 0;
    }
    else {
        return -E_INVAL;
    }
}

int sys_write_dev(int sysno, u_int va, u_int dev, u_int len) {
    if ((dev >= 0x10000000 && (dev + len - 1) < 0x10000020) ||
        (dev >= 0x13000000 && (dev + len - 1) < 0x13004200) ||
        (dev >= 0x15000000 && (dev + len - 1) < 0x15000200)) {
        u_long dev_va = dev + 0xa0000000;//将设备物理地址映射到内核kseg1空间
        bcopy(dev_va, va, len);//从设备物理地址dev中读出长len字节的数据写入va对应的虚拟地址中
        return 0;
    }
    else {
        return -E_INVAL;
    }
}
```

2. 实现**用户态**下的磁盘驱动,**对磁盘设备的空间的访问要基于之前实现的用户态系统调用**(因为设备映射的是内核虚拟空间,用户态不能直接访问内核虚拟地址,所以要通过系统调用)

```c
//fs/ide.c
void ide_read(u_int diskno, u_int secno, void *dst, u_int nsecs)
{
    //0x200表示512字节,这是一个扇区的大小
	int offset_begin = secno * 0x200;//初始扇区地址偏移
	int offset_end = offset_begin + nsecs * 0x200;//最后一个扇区地址偏移
	int offset = 0;
	while(offset_begin + offset < offset_end)
	{
         syscall_write_dev(&diskno, 0x13000010, 4);//写入磁盘号,理论上恒为0(因为只有一个磁盘)
         int va = offset_begin + offset;
         syscall_write_dev(&va, 0x13000000, 4);//写入偏移量决定操作的磁盘空间位置
         va = 0;
         syscall_write_dev(&va, 0x13000020, 4);//写入操作数为0表示读
         syscall_read_dev(&va, 0x13000030, 1);//读取状态值
		if (va != 0)//读操作成功,此时数据已经存到了磁盘外设寄存器中
		{//对于读操作先启动磁盘将数据从由offset决定的扇区读出写入磁盘buffer再从buffer中读出到目标用户地址dst中
			syscall_read_dev(dst + offset, 0x13004000, 0x200);
			offset += 0x200;		
		} else {
			user_panic("disk I/O error");
		}
	}
}

void ide_write(u_int diskno, u_int secno, void *src, u_int nsecs)
{
	i//0x200表示512字节,这是一个扇区的大小
	int offset_begin = secno * 0x200;//初始扇区地址偏移
	int offset_end = offset_begin + nsecs * 0x200;//最后一个扇区地址偏移
	int offset = 0;
	while(offset_begin + offset < offset_end)
	{
         syscall_write_dev(&diskno, 0x13000010, 4);//写入磁盘号,理论上恒为0(因为只有一个磁盘)
         int va = offset_begin + offset;
         syscall_write_dev(&va, 0x13000000, 4);//写入偏移量决定操作的磁盘空间位置
         //对于写操作先将要写入的数据写到磁盘外设寄存器中的buffer中,在向磁盘写入操作位磁盘自身从buffer中读取数据写入由offset决定的扇区中
         syscall_write_dev(src + offset, 0x13004000, 0x200);
         va = 1;
         syscall_write_dev(&va, 0x13000020, 4);//写入操作数为1表示写
         syscall_read_dev(&va, 0x13000030, 1);//读取状态值
		if (va != 0) {//写操作成功
			offset += 0x200;		
		} else {
			user_panic("disk I/O error");
		}
	}
}
```



## 文件系统

文件系统实现**对磁盘的空间以文件的形式组织管理**

![](OS-lab5\dis_mm.png)

### 磁盘在物理和逻辑层次上的表现

我们将磁盘的空间抽象成了文件的形式,在物理层次和逻辑层次磁盘空间有不同的划分 :

* 物理上 : **以扇区划分,大小为`512B`**

  1. **扇区是块设备传输数据的基本单元**,也就是说它是块设备中最小的寻址单位
  2. 体现 : 在磁盘驱动实现中每次偏移加`0x200`

* 逻辑上 : **以数据块划分,大小为`4KB`**

  1. **块是内核对文件系统的一种抽象**,也就是说`OS`执行的所有磁盘操作都是以块为基本单位的.
  2. 体现 : 文件控制块中的数据指针每个指向`4KB`大小的空间

* **关系** :

  **扇区是硬件设备传输数据的最小单位,而块是操作系统传输数据的最小单位。一个块通常对应一个或多个相邻的扇区**

### 超级块

1. 超级块结构

   ```c
   struct Super {
   	u_int s_magic;		//魔数用来校验是否合法
   	u_int s_nblocks;	//保存该磁盘中的数据块个数
   	struct File s_root;	//文件结构的根目录
   };
   ```

### 位图结构

**位图结构用来标记某一数据块是否被使用过,一位代表一个数据块, 为1表示空闲,为0表示不可被使用**

1. 初始化 : 

   所有的存在的块对应的标记位为1,**注意最后一个块标记位之后的位标记为0**

2. 释放块,即将对应的标记位标记位1

   ```c
   //fs/fs.c
   u_int *bitmap; //使用一个int型的数组来实现bitmap
   void free_block(u_int blockno)
   {
           if (blockno==0) return;//0号块是启动引导扇区不允许释放
       	/*
       	 1. bitmap是int型数组,一个元素占据32位,所以(块号/32)得到表示的整数在数组中的下标
       	 2. (blockno % 32)得到对应的在该整数中的偏移
       	 3. 按位或上 1<<(blockno % 32) [100...00] 即可将对应位设置为1
       	*/
           bitmap[blockno/32] = bitmap[blockno/32] | (1<<(blockno%32));
   }
   ```

### 文件控制块

1. 结构

   ```c
   //include/fs.h
   #define MAXNAMELEN	128
   #define NDIRECT		10
   struct File {
   	u_char f_name[MAXNAMELEN];	//文件名
   	u_int f_size;			   //文件的字节大小
   	u_int f_type;			   //文件的类型
   	u_int f_direct[NDIRECT];    //10个直接的数据指针,指向磁盘中组成文件的数据块
   	u_int f_indirect;		   //当文件内容过大时使用间接数据指针
   	struct File *f_dir;		
   	u_char f_pad[256-MAXNAMELEN-4-4-NDIRECT*4-4-4];
   };
   ```

   * 直接指针 : 一共有10个,每个指向一个`4KB`的数据块,所以**只通过直接数据指针最大表示`40KB`大小的文件**

   * 间接指针 : 当文件大小超过`40KB`时使用 : **其为一个指向(存储(指向(存储文件内容的磁盘块)的指针)的数据块)的指针)**

     为方便起见在该**特殊的数据块中我们前10个指针缺省,这些由直接指针实现功能**

   ![](OS-lab5\data_point.png)

   