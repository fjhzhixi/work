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
         char operate = 0;
         syscall_write_dev(&operate, 0x13000020, 1);//写入操作数为0表示读
         syscall_read_dev(&va, 0x13000030, 4);//读取状态值
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
         char operate = 1;
         syscall_write_dev(&operate, 0x13000020, 1);//写入操作数为1表示写
         syscall_read_dev(&va, 0x13000030, 4);//读取状态值
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

### 磁盘在物理和逻辑层次上的表示

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

在该实验中我们使用`u_int *bitmap`(在`fs.c`中定义)来实现位图结构

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
   	u_int f_size;			   //文件包含的数据块的字节大小
   	u_int f_type;			   //文件的类型
   	u_int f_direct[NDIRECT];    //10个直接的数据指针,指向磁盘中组成文件的数据块
   	u_int f_indirect;		   //当文件内容过大时使用间接数据指针
   	struct File *f_dir;		
   	u_char f_pad[256-MAXNAMELEN-4-4-NDIRECT*4-4-4];
   };
   ```

   * 直接指针 : 一共有10个,每个指向一个`4KB`的数据块,所以**只通过直接数据指针最大表示`40KB`大小的文件**

   * 间接指针 : 当文件大小超过`40KB`时使用 : **其为一个指向(存储(指向(存储文件内容的磁盘块)的指针)的数据块)的指针)**,间接指针指向的还是有个块,但是其**特殊之处在于该块中存储的不是文件的数据内容,而是指向存储文件数据内容的块的指针们**

     为方便起见在该**特殊的数据块中我们前10个指针缺省,这些由直接指针实现功能**

   ![](OS-lab5\data_point.png)

**注意** :

1. 在我们的体系中, **目录也是文件,是一种特殊的文件而已,特殊之处在于 : **
   * **一般文件的控制块指针指向的数据块存储的是文件自身的数据**
   * **目录文件的控制块指针指向的数据块存储的是其目录下包含的文件(或者文件目录)的控制块**
2. 我们是用**数组的形式将数据块组织起来的**,所以在文件控制块中我们所谓的指向**数据块的指针其实就是保存该数据块的数组下标即可**(之后统一用指针表达比较形象)

### 如何构建一个文件

该功能的实现是在`fsformat.c`中实现,其作用是**基于数个已经存在的源文件将其组织成磁盘上的文件的形式,最后生成`fs.img`磁盘文件**

#### 基本流程

1. 初始化磁盘结构 : 
   * 第一个磁盘块为`boot`启动块
   * 第二个磁盘块为记录数据块使用情况的位图`map`块
   * 第三个磁盘块为保存`super`块
2. 对每个源文件 : 
   * 填写文件控制块
   * 拷贝数据块并建立链接指针
3. 初始位图,标记已经使用的数据块
4. 将构造好的磁盘结构生成文件`fs.img`文件

**图示** :

#### 数据结构

```c
typedef struct Super Super; //超级块结构
typedef struct File File;   //文件控制块结构

#define BY2BLK		BY2PG//4096,即一个块的字节(Byte)数,一个块4KB大小
#define BIT2BLK		(BY2BLK*8)//一个块中的位数(Bit)
#define NBLOCK 1024 // 一个磁盘中的数据块的数目
uint32_t nbitblock; // bitmap位图结构占据的数据块个数
uint32_t nextbno;   // 始终指向当前第一个可用的数据块结构

struct Super super;//超级块结构

enum {//块存储的都是数据,但是数据的含义不同
    BLOCK_FREE  = 0,//空闲块
    BLOCK_BOOT  = 1,//根目录块
    BLOCK_BMAP  = 2,//位图结构块
    BLOCK_SUPER = 3,//超级块
    BLOCK_DATA  = 4,//存储文件数据块
    BLOCK_FILE  = 5,//存储文件控制块的块
    BLOCK_INDEX = 6,//存储文件控制块中的间接数据指针的块
};//表示数据块的类型
//与所有数据块对应的数据结构
struct Block {
    uint8_t data[BY2BLK];//字节数组
    uint32_t type;
} disk[NBLOCK];//该数组结构代表一个磁盘中所有可用的数据块
```

**注意 : 一个`block`中存储的数据有多种含义**

#### 具体实现

函数调用图示 :

##### 功能函数 

1. `next_block(int type)` : 获得**当前可用的第一个数据块下标返回,并设定该块的使用类型**

   ```c
   int next_block(int type) {
       disk[nextbno].type = type;
       return nextbno++;//先return再+++
   }
   ```

2. `save_block_link(struct File *f, int nblk, int bno)` : **将第`bno`个数据块链接到`f`文件的第`nblk`个指针**

   ```c
   void save_block_link(struct File *f, int nblk, int bno)
   {
       assert(nblk < NINDIRECT);
       if(nblk < NDIRECT) {//当可以用直接指针表示时
           f->f_direct[nblk] = bno;//建立映射
       }
       else {
           if(f->f_indirect == 0) {//当第一次用间接指针表示时
               f->f_indirect = next_block(BLOCK_INDEX);//为间接指针分配一个数据块用来存储指向数据块的指针,实际上就是给f_indirect赋值一个空闲块的下标
           }
           /*通过disk[f->f_indirect].data获得存储指针的数据块的数据数组
            *注意此处的强制类型转换不可以少,因为data[]自身是字节数组,但是一个bno占一个字大小
            *所以要(uint32_t *)之后再通过[nblk]索引
           */
           ((uint32_t *)(disk[f->f_indirect].data))[nblk] = bno;
       }
   }
   ```

   说明 : 

   * `#define NINDIRECT  (BY2BLK/4)` : `NINDIRECT`值为1024,即一个文件**最多外接1024个数据块**,这是因为一个文件控制块只有一个间接指针,指向一个`4KB`大小的存储指针的块,而一个指针`4B`大小,所以最多存储1024个指向数据块的指针
   * `#define NDIRECT 10` : 表示一个文件控制块有10个直接指针

3. `make_link_block(struct File *dirf, int nblk)` : **该函数是对目录文件的操作(第一个参数)**

   为该目录文件控制块的`nblk`指针**分配一块存储目录下属文件的控制块的块**,增加目录文件的大小并返回分配的块的下标

   ```c
   int make_link_block(struct File *dirf, int nblk) {
       save_block_link(dirf, nblk, nextbno);
       dirf->f_size += BY2BLK;
       return next_block(BLOCK_FILE);
   }
   ```

4. `create_file(struct File *dirf)` : **该函数是对目录文件的操作(第一个参数)**

   **在目录文件`dirf`中找到一个可以写入(所属文件的控制块)的空闲数据块地址,返回该地址[当不存在空闲数据区域时新分配数据块用来保存文件控制块]**

   ```c
   struct File *create_file(struct File *dirf) {
       struct File *dirblk;
       int i, bno, found;
       int nblk = dirf->f_size / BY2BLK;//获得当前目录文件有效的数据指针个数
       if(nblk == 0) {//该目录文件没有数据块,使用make_link_block分配,此时nblk = 0
           return (struct File *)(disk[make_link_block(dirf, 0)].data);
       }
       if(nblk <= NDIRECT) {//还在使用直接数据块指针阶段
           bno = dirf->f_direct[nblk-1];
       }
       else {//使用间接数据块指针
           bno = ((uint32_t *)(disk[dirf->f_indirect].data))[nblk-1];
       }
       //至此获得当前文件最后一个数据块的下标,
       /*之后按文件控制块的大小遍历该数据块
        *检查文件名是否为空来确定是否是空闲文件控制块
        *返回找到的第一个空闲文件控制块的指针
       */
       dirblk = (struct File *)(disk[bno].data);
       for(i = 0; i < FILE2BLK; ++i) {
           if(dirblk[i].f_name[0] == '\0') {
               // found spare file link.
               return &dirblk[i];
           }
       }
       /*当最后一个数据块没有空闲文件控制块时
        *给目录文件使用make_link_block新分配一个
        *返回新数据块的首地址(以文件控制块指针的形式)
       */
       return (struct File *)(disk[make_link_block(dirf, nblk)].data);
   }
   ```

   **注意 : 有如下的关系** :

   * **`f_size = nblk * BY2BLK` : `nblk`为该(目录)文件有效的数据块指针的个数**

##### 实现流程

1. main`函数 : 运行生成磁盘镜像文件

   ```c
   int main(int argc, char **argv) {
       /*参数的含义
        *第一个参数为文件名本身argv[0]
        *第二个参数是产生的磁盘镜像的目标文件路径argv[1]
        *之后的参数是基于的源文件argv[2]...
       */
       int i;
       init_disk();	//初始化
       if(argc < 3 || strcmp(argv[2], "-r") == 0 && argc != 4) {//参数错误
           fprintf(stderr, "\Usage: fsformat gxemul/fs.img files...\n\fsformat gxemul/fs.img -r DIR\n");
           exit(0);
       }
       if(strcmp(argv[2], "-r") == 0) {
           for (i = 3; i < argc; ++i) {
               write_directory(&super.s_root, argv[i]);//不要求实现
           }
       }
       else {
           for(i = 2; i < argc; ++i) {
               write_file(&super.s_root, argv[i]);//逐个读取源文件以磁盘文件的形式组织起来
           }
       }
       flush_bitmap();//初始化位图控制块
       finish_fs(argv[1]);//生成目标文件
       return 0;
   }
   ```

2. `init_disk()` : 初始化磁盘结构

   * 第一个块初始化为`boot`
   * 第二个块初始化为`super`
   * 之后若干个需要大小的块初始化为`bitmap`

   ```c
   void init_disk() {
       int i, r, diff;
       //第一个块为boot结构
       disk[0].type = BLOCK_BOOT;
       //建立bitmap结构
       //NBLOCK为块的数目,所以应该分配NBLOCK个bit大小来存储bitmap结构
       //(+ BIT2BLK - 1)的操作是因为当填不满某一块时也要分配给bitmap(剩余一部分)
       nbitblock = (NBLOCK + BIT2BLK - 1) / BIT2BLK;
       nextbno = 2 + nbitblock;
       for(i = 0; i < nbitblock; ++i) {
           disk[2+i].type = BLOCK_BMAP;
       }
      //??最开始的使用的块为什么不标记为使用的??
       for(i = 0; i < nbitblock; ++i) {
           memset(disk[2+i].data, 0xff, NBLOCK/8);
       }
       //当多余的无效的位都标记为0表示不可用
       if(NBLOCK != nbitblock * BY2BLK) {
           diff = NBLOCK % BY2BLK / 8;
           memset(disk[2+(nbitblock-1)].data+diff, 0x00, BY2BLK - diff);
       }
       //初始化super块中的数据
       disk[1].type = BLOCK_SUPER;
       super.s_magic = FS_MAGIC;
       super.s_nblocks = NBLOCK;
       super.s_root.f_type = FTYPE_DIR;
       strcpy(super.s_root.f_name, "/");
   }
   ```

3. `write_file(struct File *dirf, const char *path)` : **将`path`指定的文件写入`dirf`文件目录之下**

   1. 为文件新建一个控制块并加入到`dirf`目录文件中(`create_file`实现)
   2. 填写新建的文件控制块的各个数据域
   3. 将文件中的内容拷贝到数据块中并**建立到新建文件控制块的链接** (`save_block_link`实现)

   ```c
   void write_file(struct File *dirf, const char *path) {
       int iblk = 0, r = 0, n = sizeof(disk[0].data);
       uint8_t buffer[n+1], *dist;
       struct File *target = create_file(dirf);//将target文件控制块加入dirf目录文件中
       int fd = open(path, O_RDONLY);//该处的open是在fcntl.h标准头文件中包含的C标准函数
       const char *fname = strrchr(path, '/');//获得最后一个/的位置
       if(fname)
           fname++;//找到了之后该位置之后的字符串就是文件名
       else
           fname = path;//没找到则该path就是顶层,本身就是文件名
       strcpy(target->f_name, fname);//向文件控制块写入文件名
       target->f_size = lseek(fd, 0, SEEK_END);//将文件指针移动到文件末尾,获得文件的大小
       target->f_type = FTYPE_REG;//该文件类型为常规类型
       //开始遍历path得到的文件将其中的内容写入文件控制块指向的数据块中
       lseek(fd, 0, SEEK_SET);//将参数移动到文件的头部(偏移为0)
       while((r = read(fd, disk[nextbno].data, n)) > 0) {//将fd文件中读出的数据写入到disk[nextbno].data中,一次写入一个data块大小(n字节)
           save_block_link(target, iblk++, next_block(BLOCK_DATA));//建立数据块到所属文件控制块的映射
       }
       close(fd);
   }
   ```

4. `flush_bitmap` : 将使用过的数据块在位图管理结构`bitmap`中标记为已使用的

   ```c
   void flush_bitmap() {
       int i;
       //nextbo为使用过的数据块个数
       for(i = 0; i < nextbno; ++i) {
           ((uint32_t *)disk[2+i/BIT2BLK].data)[(i%BIT2BLK)/32] &= ~(1<<(i%32));
       }
   }
   ```

5. `finish_fs(char *name)` : **根据之前组织好的内存中的数据生成磁盘镜像文件**

   ```c
   void finish_fs(char *name) {
       int fd, i, k, n, r;
       uint32_t *p;
       memcpy(disk[1].data, &super, sizeof(super));//将super放入磁盘第一个数据块
       fd = open(name, O_RDWR|O_CREAT, 0666);
       for(i = 0; i < 1024; ++i) {
           reverse_block(disk+i);
           write(fd, disk[i].data, BY2BLK);//将数据块内容写入文件
       }
       close(fd);
   }
   ```

   