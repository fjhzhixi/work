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

* 磁盘各个外设寄存器的意义 : 

  ![](OS-lab5\IDE-regs.png)

  注意实际上我们只有一块磁盘,所以`IDE ID`恒为0

* 所以我们的对磁盘的`I/O`操作就如下图

  ![](OS-lab5\w-r.png)

**注意此处有一点我没有想通的机制 : 就是我们认为从磁盘中读取数据和向磁盘写入数据都是瞬间完成的,没有延时,所以没有实现`CPU`轮询状态位或者磁盘通知`CPU`中断类似的机制,我猜时因为我们的外设磁盘是仿真模拟的,所以讲问题简化了**

### 具体实现

```c
//fs/ide.c
int read_sector(int diskno, int offset) {
    int *int_addr;
    int_addr = 0xB3000010;
    *int_addr = diskno; //写入磁盘号
    int_addr = 0xB3000000;
    *int_addr = offset; //写入偏移量
    int_addr = 0xB3000020;
    *int_addr = 0; //写入操作位0表示读
    int_addr = 0xB3000030;
	return *int_addr;	//返回状态位
}

void ide_read(u_int diskno, u_int secno, void *dst, u_int nsecs)
{
    //0x200表示512字节,这是一个扇区的大小
	int offset_begin = secno * 0x200;//初始扇区地址偏移
	int offset_end = offset_begin + nsecs * 0x200;//最后一个扇区地址偏移
	int offset = 0;
	while(offset_begin + offset < offset_end)
	{
		if (read_sector(offset_begin + offset))
		{//对于读操作先启动磁盘将数据写入磁盘buffer再由内核地址拷贝到用户地址
			user_bcopy( 0x93004000,dst + offset, 0x200);
			offset += 0x200;		
		} else {
			user_panic("disk I/O error");
		}
	}
}

int write_sector(int diskno, int offset) {
    int *int_addr;
    int_addr = 0xB3000010;
    *int_addr = diskno; //写入磁盘号
    int_addr = 0xB3000000;
    *int_addr = offset; //写入偏移量
    int_addr = 0xB3000020;
    *int_addr = 1; //写入操作位1表示写
    int_addr = 0xB3000030;
	return *int_addr;	//返回状态位
}

void ide_write(u_int diskno, u_int secno, void *src, u_int nsecs)
{
	int offset_begin = secno * 0x200;
	int offset_end = offset_begin + nsecs * 0x200;
	int offset = 0;

	while (offset_begin + offset < offset_end)
	{//对于写操作先从用户地址把数据拷贝到指定内核地址(实际写入磁盘缓存区)中再启动写入磁盘
		user_bcopy(src + offset,0x93004000,  0x200);
		if (write_sector(offset_begin + offset))
		{			
			offset += 0x200;
		} else {
			user_panic("disk I/O error");
		}
	}
}
```

## 文件系统

