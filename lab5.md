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
   //include/fs.h
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

![](OS-lab5\gen_file.png)

#### 数据结构

```c
//fs/fsformat.c
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

![](OS-lab5\main_call.png)

1. `main`函数 : 运行生成磁盘镜像文件

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
       flush_bitmap();//改变位图控制块
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
      //标记可用的位(前3个块在最后标记为不可用)
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


#### 结果

最后的文件结构为 :

![](OS-lab5\file_struct.png)

### 块缓存机制

**我们使用磁盘中的数据是通过将其加载到内存中的位置实现的**

1. 映射机制

   将`DISKMA1P`到`DISKMAP+DISKMAX`这一段虚存地址空间 `(0x10000000-0xcﬀﬀﬀ)`作为缓冲区,当磁盘读入内存时,分配相应的物理页来保存数据,映射关系直接是**地址从低到高,数据块索引从小到大的线性映射**

   **注意 : 在我们的体系中磁盘上的一个数据块大小与内存中的一页相同,都是`4KB`**

2. 功能函数 : 

   ```c
   //fs/fs.c
   //根据数据块索引得到在内存中映射的虚拟地址
   u_int diskaddr(u_int blockno)
   {
           if ( super && blockno > (super->s_nblocks)) {
                   user_panic("diskaddr failed!");
           }
           return DISKMAP+blockno*BY2BLK;
   }
   /*判断该虚拟页是否有物理页与之对应
    *有返回1,没有返回0
    *通过判断页目录与页表的对应项是否有效
   */
   u_int va_is_mapped(u_int va)
   {
           return (((*vpd)[PDX(va)] & (PTE_V)) && ((*vpt)[VPN(va)] & (PTE_V)));
   }
   /*判断该索引的数据块是否在内存中有有效的虚拟页对应
    *如果有返回va值,没有返回0
    *通过判断其映射的虚拟地址是否与物理页与之对应
   */
   u_int block_is_mapped(u_int blockno)
   {
           u_int va = diskaddr(blockno);
           if (va_is_mapped(va)) {
                   return va;
           }
           return 0;
   }
   /*检查该虚拟地址对应的数据是否改变过
    *改变过返回1,否则返回0
    *通过检查对应页表项的dirty位来实现
   */
   u_int va_is_dirty(u_int va)
   {
           return (* vpt)[VPN(va)] & PTE_D;
   }
   /*判断当前块是否有效
    *有效则返回1,无效则返回0
    *根据位图结构中对应位的值
   */
   int
   block_is_free(u_int blockno)
   {
           if (super == 0 || blockno >= super->s_nblocks) {//块号超过范围
                   return 0;
           }
           if (bitmap[blockno / 32] & (1 << (blockno % 32))) {
                   return 1;
           }
           return 0;
   }   
   /*检查该块是否修改过
    *修改过返回1,否则返回0
    *通过检查块映射虚拟地址的dirty位
   */
   u_int block_is_dirty(u_int blockno)
   {
           u_int va = diskaddr(blockno);
           return va_is_mapped(va) && va_is_dirty(va);
   }
   //从磁盘中读取blockno代表的数据块到其规定的虚拟地址缓存中,并将blk的值设置为该虚拟地址的值,若是第一次加载则设置isnew为1
   int read_block(u_int blockno, void **blk, u_int *isnew)
   {
           u_int va;
           if (super && blockno >= super->s_nblocks) {
                   user_panic("reading non-existent block %08x\n", blockno);
           }
       	//检查该块是否在位图结构中对应的位有效
           if (bitmap && block_is_free(blockno)) {
                   user_panic("reading free block %08x\n", blockno);
           }
           va = diskaddr(blockno);
   		/*检查该块是否在虚拟内存中有有效的映射缓冲块
   		 *如果已有映射说明磁盘中的数据已经读到了缓冲块中,只需设置isnew标记位为0即可
   		 *如果没有映射则新分配一个物理页用来作为该块在虚拟内存中的缓冲块,并从磁盘中读取数据填入,并且设置isnew标记位1
   		 *设置blk为作为缓存块的虚拟地址
   		*/
           if (block_is_mapped(blockno)) { //the block is in memory
                   if (isnew) {
                           *isnew = 0;
                   }
           } else {                        //the block is not in memory
                   if (isnew) {
                           *isnew = 1;
                   }
                   syscall_mem_alloc(0, va, PTE_V | PTE_R);
                   ide_read(0, blockno * SECT2BLK, (void *)va, SECT2BLK);
           }
           if (blk) {
                   *blk = (void *)va;
           }
           return 0;
   }
   //将第blockno块在内存中的虚拟缓存中的数据写回磁盘中
   void write_block(u_int blockno)
   {
           u_int va;
           //检测当前块是否存储有效的虚拟内存缓存块
           if (!block_is_mapped(blockno)) {
                   user_panic("write unmapped block %08x", blockno);
           }
   		//向磁盘中写入内存中缓存区的信息
           va = diskaddr(blockno);
           ide_write(0, blockno * SECT2BLK, (void *)va, SECT2BLK);
       	//为缓存区的映射增加标记位表示修改过
           syscall_mem_map(0, va, 0, va, (PTE_V | PTE_R | PTE_LIBRARY));
   }
   /*对第blockno索引的数据块在虚拟内存中建立缓存页
    *如果本来就存在缓存,则返回0,否则新建一页缓存,返回映射的虚拟地址
   */
   int map_block(u_int blockno)
   {
           u_int va;
           if (va = block_is_mapped(blockno)) {
                   return 0;
           }
       	va = disdiskaddr(blockno);
           return syscall_mem_alloc(0,va,PTE_V|PTE_R);
   }
   /*解除第blockno索引的数据块与内存中缓存块的映射(起始就是将对应虚拟内存解除与物理页的映射)
    *如果该虚拟内存缓存块是dirty(即被修改过的),则要先写回磁盘中去再解除映射
   */
   void unmap_block(u_int blockno)
   {
           int r;
           u_int va;
           if (va=block_is_mapped(blockno)) {
                   if (!block_is_free(blockno) && block_is_dirty(blockno)) {
                           write_block(blockno);
                   }
                   if (syscall_mem_unmap(0,va))
                           writef("unmap_block failed!");
           }
           user_assert(!block_is_mapped(blockno));
   }
   //释放blockno索引代表的块(实际上只做了将其在位图中对应位置的位置为1表示可用)
   void free_block(u_int blockno)
   {
         	//第一个是boot块不允许释放
           if (blockno==0) return;
           //将对应标记位置为1
           bitmap[blockno/32] = bitmap[blockno/32] && ~(1<<(blockno%32));
   }
   //在位图标记中寻找第一个空闲可用的块返回其索引值(在其使用前要将原数据写回)
   int alloc_block_num(void)
   {
           int blockno;
           for (blockno = 3; blockno < super->s_nblocks; blockno++) {
                   if (bitmap[blockno / 32] & (1 << (blockno % 32))) { 
                           bitmap[blockno / 32] &= ~(1 << (blockno % 32));
           //注意 : 我们在free_block中只是改变了bitmap中的标记位而已,所以真正要使用这个块之前要将其数据写回到磁盘中去(使用write_block来写回)
                           write_block(blockno / BIT2BLK);
                           return blockno;
                   }
           }
           return -E_NO_DISK;
   }
   /*分配一个可用的数据块返回其索引值,可用的含义如下:
    1.在位图中标记位为1
    2.已经将之间的数据写回磁盘中
    3.在虚拟内存中建立了缓存块映射
   */
   int alloc_block(void)
   {
           int r, bno;
           //找到一个空闲的块并将值写回磁盘
           if ((r = alloc_block_num()) < 0) {
                   return r;
           }
           bno = r;
       	//建立虚拟内存中的缓存
           if ((r = map_block(bno)) < 0) {
                   free_block(bno);
                   return r;
           }
           return bno;
   }
   ```

   **注意** :

   * 有关有效位
     1. `PTE_D` : 
     2. `PTE_LIBRARY` : 

   * 有关映射
     1. **磁盘块的索引与虚拟地址中的缓存块地址**二者的映射关系是**确定**的,即按索引从小到大,地址按页小到大来静态映射的
     2. **我们所说的`map`和`unmap`实质上是建立索引对应的虚拟页映射到一个有效的物理页**,即保证虚拟缓存块是有效的

   * **有关块的释放与获取**

     1. 我们在释放块时只是**做了将其在`bitmap`结构中的对应标记位设置为1**的简单操作

     2. 当要获取一块时,我们要:

        1. 在`bitmap`中找到一个标记为1(可用)的块
        2. **将其对应的缓存内容写回磁盘中**
        3. 在虚拟内存中建立缓存块

        **至此该数据块才真正可以被分配使用**

### 对文件系统的操作

注意区分对**一般文件和对目录文件**二者操作的不同

二者的数据结构组织相同,但是**数据的含义**不同

```c
//fs/fs.c
//读取超级块内容到缓存区中
void read_super(void)
{
        int r;
        void *blk;
        if ((r = read_block(1, &blk, 0)) < 0) {
                user_panic("cannot read superblock: %e", r);
        }
        super = blk;
        if (super->s_magic != FS_MAGIC) {
                user_panic("bad file system magic number %x %x", super->s_magic, FS_MAGIC);
        }
        if (super->s_nblocks > DISKMAX / BY2BLK) {
                user_panic("file system is too large");
        }
        writef("superblock is good\n");
}
//读取位图结构bitmap
void read_bitmap(void)
{
        u_int i;
        void *blk = NULL;
//nbitmap代表的是位图结构bitmap占用的数据块的个数(至少为1个)
        nbitmap = super->s_nblocks / BIT2BLK + 1;
//将bitmap读入到虚拟内存缓存中
        for (i = 0; i < nbitmap; i++) {
                read_block(i + 2, blk, 0);
        }
        bitmap = (u_int *)diskaddr(2);
 //检查bitmap中前两个块(boot和super)和bitmap占用的块必定是已被使用的   	
        user_assert(!block_is_free(0));
        user_assert(!block_is_free(1));
        for (i = 0; i < nbitmap; i++) {
                user_assert(!block_is_free(i + 2));
        }
        writef("read_bitmap is good\n");
}
void fs_init(void)
{
        read_super();
        check_write_block();
        read_bitmap();
}
//在文件控制块f中查询第fileno个指针指向的数据块的索引值,ppdiskbno指向保存该索引值的地址
//alloc为1代表在间接指针无效时允许新分配一页
int file_block_walk(struct File *f, u_int filebno, u_int **ppdiskbno, u_int alloc)
{
        int r;
        u_int *ptr;
        void *blk;
//指针值在直接指针范围内直接通过直接指针获得
        if (filebno < NDIRECT) {
                ptr = &f->f_direct[filebno];
        } else if (filebno < NINDIRECT) {
        //当指针值需要使用间接指针时
                if (f->f_indirect == 0) {
                        if (alloc == 0) {//不允许alloc
                                return -E_NOT_FOUND;
                        }
                        if ((r = alloc_block()) < 0) {//允许alloc则分配一块存储间接指针
                                return r;
                        }
                        f->f_indirect = r;
                }
			//blk保存(保存间接指针的虚拟页)的起始地址
                if ((r = read_block(f->f_indirect, &blk, 0)) < 0) {
                        return r;
                }
            //则ptr指向(保存fileno指针指向的数据块索引)的地址
            //即*ptr即为fileno指向的数据块的索引值
                ptr = (u_int *)blk + filebno;
        } else {
                return -E_INVAL;
        }
        *ppdiskbno = ptr;
        return 0;
}
//在f文件控制块中查找第fileno个指针指向的数据块对应的索引值赋值给diskbno(即建立文件控制块域与数据块关系)
//当alloc为1时若查找出来的索引无效,允许新分配一块
int file_map_block(struct File *f, u_int filebno, u_int *diskbno, u_int alloc)
{
        int r;
        u_int *ptr;
//在f文件控制块中找对应的数据索引
        if ((r = file_block_walk(f, filebno, &ptr, alloc)) < 0) {
                return r;
        }
//*ptr保存索引值
//如果索引无效
        if (*ptr == 0) {
                if (alloc == 0) {
                        return -E_NOT_FOUND;
                }
			//如果允许分配则新分配一页,并写入控制块结构对应位置
                if ((r = alloc_block()) < 0) {
                        return r;
                }
            //填入新分配的块的索引
                *ptr = r;
        }
        *diskbno = *ptr;
        return 0;
}
//释放文件控制块f的第filebno个指针指向的数据块
int file_clear_block(struct File *f, u_int filebno)
{
        int r;
        u_int *ptr;
        if ((r = file_block_walk(f, filebno, &ptr, 0)) < 0) {
                return r;
        }
        if (*ptr) {
                free_block(*ptr);
                *ptr = 0;
        }
        return 0;
}
//获得文件控制块f中第filebno个指针指向的数据块中的数据加载到虚拟内存中,并令blk指向该虚拟地址
//当filebno超过索引范围之后,则新分配一个
int file_get_block(struct File *f, u_int filebno, void **blk)
{
        int r;
        u_int diskbno;
        u_int isnew;
    //找到数据块的索引
        if ((r = file_map_block(f, filebno, &diskbno, 1)) < 0) {
                return r;
        }
    //加载该数据块到虚拟内存blk中
        if ((r = read_block(diskbno, blk, &isnew)) < 0) {
                return r;
        }
        return 0;
}
//???
int file_dirty(struct File *f, u_int offset)
{
        int r;
        void *blk;

        if ((r = file_get_block(f, offset / BY2BLK, &blk)) < 0) {
                return r;
        }

        *(volatile char *)blk = *(volatile char *)blk;
        return 0;
}
//获得目录文件dir之下名为name的文件,如果有则将该文件的文件控制块赋值给file(注意此时dir问name文件的直接父目录)
int dir_lookup(struct File *dir, char *name, struct File **file)
{
        int r;
        u_int i, j, nblock;
        void *blk;
        struct File *f;
    //nblock为该目录文件之下的数据块个数
        nblock = ROUND(dir->f_size,BY2BLK)/BY2BLK;
        for (i = 0; i < nblock; i++) {
                //获得一个保存文件控制块的数据块
                if (r=file_get_block(dir,i,&blk))
                        return r;
                f = (struct File*)blk;
                //遍历该块中的每一个文件控制块,比较文件名
                for (j=0;j<FILE2BLK;j++) {
                        if (strcmp(f[j].f_name,name)==0) {
                                f[j].f_dir = dir;
                                *file = &f[j];
                                return 0;
                        }
                }
        }
        return -E_NOT_FOUND;
}
//在目录文件dir之下寻找一个空闲的文件控制块,并将其赋值给file
//如果没有空闲的控制块,则新分配一个数据块用来存储文件控制块
int dir_alloc_file(struct File *dir, struct File **file)
{
        int r;
        u_int nblock, i , j;
        void *blk;
        struct File *f;
        nblock = dir->f_size / BY2BLK;
    //遍历每一个块
        for (i = 0; i < nblock; i++) {
                if ((r = file_get_block(dir, i, &blk)) < 0) {
                        return r;
                }
                f = blk;
            //遍历块中的每一个文件控制块
                for (j = 0; j < FILE2BLK; j++) {
                        if (f[j].f_name[0] == '\0') { 
                                *file = &f[j];
                                return 0;
                        }
                }
        }
    //当没找到之后新分配一个数据块
        dir->f_size += BY2BLK;
        if ((r = file_get_block(dir, i, &blk)) < 0) {
                return r;
        }
        f = blk;
        *file = &f[0];
        return 0;
}
//去除路径中的空格
char *skip_slash(char *p)
{
        while (*p == '/') {
                p++;
        }
        return p;
}
//难道我们不支持文件目录的嵌套吗???
//返回0代表找到了给文件
int walk_path(char *path, struct File **pdir, struct File **pfile, char *lastelem)
{
        char *p;
        char name[MAXNAMELEN];
        struct File *dir, *file;
        int r;

        // start at the root.
        path = skip_slash(path);
        file = &super->s_root;
        dir = 0;
        name[0] = 0;

        if (pdir) {
                *pdir = 0;
        }

        *pfile = 0;

        // find the target file by name recursively.
        while (*path != '\0') {
                dir = file;
                p = path;

                while (*path != '/' && *path != '\0') {
                        path++;
                }

                if (path - p >= MAXNAMELEN) {
                        return -E_BAD_PATH;
                }

                user_bcopy(p, name, path - p);
                name[path - p] = '\0';
                path = skip_slash(path);

                if (dir->f_type != FTYPE_DIR) {
                        return -E_NOT_FOUND;
                }

                if ((r = dir_lookup(dir, name, &file)) < 0) {
                        if (r == -E_NOT_FOUND && *path == '\0') {
                                if (pdir) {
                                        *pdir = dir;
                                }

                                if (lastelem) {
                                        strcpy(lastelem, name);
                                }

                                *pfile = 0;
                        }

                        return r;
                }
        }

        if (pdir) {
                *pdir = dir;
        }

        *pfile = file;
        return 0;
}
//根据path找到其代表的文件控制块并赋值给file
int file_open(char *path, struct File **file)
{
        return walk_path(path, 0, file, 0);
}
//根据path在目录下创建文件,并将新建的文件控制块赋值给file
int file_create(char *path, struct File **file)
{
        char name[MAXNAMELEN];
        int r;
        struct File *dir, *f;
        if ((r = walk_path(path, &dir, &f, name)) == 0) {
                return -E_FILE_EXISTS;
        }
        if (r != -E_NOT_FOUND || dir == 0) {
                return r;
        }
        if (dir_alloc_file(dir, &f) < 0) {
                return r;
        }
        strcpy((char *)f->f_name, name);
        *file = f;
        return 0;
}
//将文件控制块f代表的文件的大小截取为newsize(要求newsize < oldsize)(不安全的方法,只能被file_set_size使用)
void file_truncate(struct File *f, u_int newsize)
{
        u_int bno, old_nblocks, new_nblocks;
        old_nblocks = f->f_size / BY2BLK + 1;
        new_nblocks = newsize / BY2BLK + 1;
        if (newsize == 0) {
                new_nblocks = 0;
        }
        if (new_nblocks <= NDIRECT) {//当只需要直接指针时
                f->f_indirect = 0;
            //当新的大小比原大小大时,将多余的块释放
                for (bno = new_nblocks; bno < old_nblocks; bno++) {
                        file_clear_block(f, bno);
                }
        } else {
                for (bno = new_nblocks; bno < old_nblocks; bno++) {
                        file_clear_block(f, bno);
                }
        }
        f->f_size = newsize;
}
/*将文件的大小改变为newsize
 *如果newsize < oldsize则截断(使用file_truncate),设置f_size为new_size
 *如果newsize > oldsize则直接设置f_size为new_size即可
 *将???
*/
int file_set_size(struct File *f, u_int newsize)
{
        if (f->f_size > newsize) {
                file_truncate(f, newsize);
        }

        f->f_size = newsize;

        if (f->f_dir) {
                file_flush(f->f_dir);//??????
        }
        return 0;
}
//将文件控制块f中所有指针指向的数据块中的dirty的写回磁盘中(即同步磁盘与内存缓存中的数据)
void file_flush(struct File *f)
{
        u_int nblocks;
        u_int bno;
        u_int diskno;
        int r;

        nblocks = f->f_size / BY2BLK + 1;

        for (bno = 0; bno < nblocks; bno++) {
            //获得当前文件第bno个指针对应的索引,当不存在时不新建映射
                if ((r = file_map_block(f, bno, &diskno, 0)) < 0) {
                        continue;
                }
            //如果由索引获得的数据块是dirty的,则将其刷新回磁盘上
                if (block_is_dirty(diskno)) {
                        write_block(diskno);
                }
        }
}
//该操作是保证数据同步性,即将内存中dirty的快缓存写回到磁盘中
void fs_sync(void)
{
        int i;
        for (i = 0; i < super->s_nblocks; i++) {
                if (block_is_dirty(i)) {
                        write_block(i);
                }
        }
}
//关闭文件的操作其实就是把文件内容从内存缓存写回磁盘中
void file_close(struct File *f)
{
        file_flush(f);
        if (f->f_dir) {
                file_flush(f->f_dir);
        }
}
int file_remove(char *path)
{
        int r;
        struct File *f;
        if ((r = walk_path(path, 0, &f, 0)) < 0) {
                return r;
        }
        file_truncate(f, 0);
        f->f_name[0] = '\0';
        file_flush(f);
        if (f->f_dir) {
                file_flush(f->f_dir);
        }
        return 0;
}
```

**注意**

好像 :

我们所谓的对文件或者目录文件的操作都是针对于在内存缓存区的,而磁盘上的文件结构或者内容不会发生改变,所以我们在每一次的修改之前都会有个写回磁盘的操作???

## 用户接口

在我们实现文件系统之后,我们需要给用户提供**使用的接口(就是一些标准化的操文件系统的函数)**

### 文件描述符

我们引入一个新的数据结构,即文件描述符

1. 定义

   ```c
   //user/fd.h
   struct Fd
   {
   	u_int fd_dev_id;
   	u_int fd_offset;
   	u_int fd_omode;
   };
   ```

2. 其在内存中的构成

   ```c
   //user/fd.c
   #define MAXFD 32 //最多同时存在32个文件描述符结构
   #define FILEBASE 0x60000000
   #define FDTABLE (FILEBASE-PDMAP)
   ```

   此处应该有个图

   **注意 : 我们一个`Fd`结构体占据一页的大小**

3. 操作

   ```c
   //user/fd.c
   int dev_lookup(int dev_id, struct Dev **dev)
   {
   	int i;
   	for (i=0; devtab[i]; i++)
   		if (devtab[i]->dev_id == dev_id) {
   			*dev = devtab[i];
   			return 0;
   		}
   	writef("[%08x] unknown device type %d\n", env->env_id, dev_id);
   	return -E_INVAL;
   }
   //找一个还没有映射的页分配给fd结构体
   int fd_alloc(struct Fd **fd)
   {
   	u_int va;
   	u_int fdno;
   	for(fdno = 0;fdno < MAXFD - 1;fdno++)
   	{
   		va = INDEX2FD(fdno);
   		//页目录无效说明该页还没有映射
   		if(((* vpd)[va/PDMAP] & PTE_V)==0)
   		{
   			*fd = va;
   		return 0;
   		}
   		//页表项无效说明该页还没有映射
   		if(((* vpt)[va/BY2PG] & PTE_V)==0)		
   		{
   			*fd = va;
   			return 0;
   		}
       }
   	return -E_MAX_OPEN;
   }
   //去除fd所在页的映射,即删除这个文件描述符
   void fd_close(struct Fd *fd)
   {
   	syscall_mem_unmap(0, (u_int)fd);
   }
   //将第fdnum个文件描述符的地址(如果有效)赋值给fd
   int fd_lookup(int fdnum, struct Fd **fd)
   {
   	u_int va;
   	if(fdnum >=MAXFD)
   		return -E_INVAL;
   	va = INDEX2FD(fdnum);//获得第fdnum个fd文件描述符的地址
   	if(((* vpt)[va/BY2PG] & PTE_V)!=0)		
   	{//该地址有效则赋值给fd
   		*fd = va;
   		return 0;
   	}
   	return -E_INVAL;
   }
   
   u_int fd2data(struct Fd *fd)
   {
   	return INDEX2DATA(fd2num(fd));
   }
   
   int fd2num(struct Fd *fd)
   {
   	return ((u_int)fd - FDTABLE)/BY2PG;
   }
   int num2fd(int fd)
   {
   	return fd*BY2PG+FDTABLE;
   }
   
   int close(int fdnum)
   {
   	int r;
   	struct Dev *dev;
   	struct Fd *fd;
   
   	if ((r = fd_lookup(fdnum, &fd)) < 0
   	||  (r = dev_lookup(fd->fd_dev_id, &dev)) < 0)
   		return r;
   	r = (*dev->dev_close)(fd);
   	fd_close(fd);
   	return r;
   }
   
   void close_all(void)
   {
   	int i;
   
   	for (i=0; i<MAXFD; i++)
   		close(i);
   }
   
   int dup(int oldfdnum, int newfdnum)
   {
   	int i, r;
   	u_int ova, nva, pte;
   	struct Fd *oldfd, *newfd;
   	
   	if ((r = fd_lookup(oldfdnum, &oldfd)) < 0)
   		return r;
   	close(newfdnum);
   
   	newfd = (struct Fd*)INDEX2FD(newfdnum);
   	ova = fd2data(oldfd);
   	nva = fd2data(newfd);
   
   	if ((* vpd)[PDX(ova)]) {
   		for (i=0; i<PDMAP; i+=BY2PG) {
   			pte = (* vpt)[VPN(ova+i)];
   			if(pte&PTE_V) {
   				// should be no error here -- pd is already allocated
   				if ((r = syscall_mem_map(0, ova+i, 0, nva+i, pte&(PTE_V|PTE_R|PTE_LIBRARY))) < 0)
   					goto err;
   			}
   		}
   	}
   
   	if ((r = syscall_mem_map(0, (u_int)oldfd, 0, (u_int)newfd, ((*vpt)[VPN(oldfd)])&(PTE_V|PTE_R|PTE_LIBRARY))) < 0)
   		goto err;
   
   	return newfdnum;
   
   err:
   	syscall_mem_unmap(0, (u_int)newfd);
   	for (i=0; i<PDMAP; i+=BY2PG)
   		syscall_mem_unmap(0, nva+i);
   	return r;
   }
   
   int read(int fdnum, void *buf, u_int n)
   {
   	int r;
   	struct Dev *dev;
   	struct Fd *fd;
   	//writef("read() come 1 %x\n",(int)env);
   	if ((r = fd_lookup(fdnum, &fd)) < 0
   	||  (r = dev_lookup(fd->fd_dev_id, &dev)) < 0)
   		return r;
   	//writef("read() come 2 %x\n",(int)env);
   	if ((fd->fd_omode & O_ACCMODE) == O_WRONLY) {
   		writef("[%08x] read %d -- bad mode\n", env->env_id, fdnum); 
   		return -E_INVAL;
   	}
   	//writef("read() come 3 %x\n",(int)env);
   	r = (*dev->dev_read)(fd, buf, n, fd->fd_offset);
   	if (r >= 0)
   		fd->fd_offset += r;
   	//writef("read() come 4 %x\n",(int)env);
   	return r;
   }
   
   int
   readn(int fdnum, void *buf, u_int n)
   {
   	int m, tot;
   
   	for (tot=0; tot<n; tot+=m) {
   		m = read(fdnum, (char*)buf+tot, n-tot);
   		if (m < 0)
   			return m;
   		if (m == 0)
   			break;
   	}
   	return tot;
   }
   
   int write(int fdnum, const void *buf, u_int n)
   {
   	int r;
   	struct Dev *dev;
   	struct Fd *fd;
   	//writef("write comes 1\n");
   	if ((r = fd_lookup(fdnum, &fd)) < 0
   	||  (r = dev_lookup(fd->fd_dev_id, &dev)) < 0)
   		return r;
   //writef("write comes 2\n");
   	if ((fd->fd_omode & O_ACCMODE) == O_RDONLY) {
   		writef("[%08x] write %d -- bad mode\n", env->env_id, fdnum);
   		return -E_INVAL;
   	}
   //writef("write comes 3\n");
   	if (debug) writef("write %d %p %d via dev %s\n",
   		fdnum, buf, n, dev->dev_name);
   	r = (*dev->dev_write)(fd, buf, n, fd->fd_offset);
   	if (r > 0)
   		fd->fd_offset += r;
   //writef("write comes 4\n");
   	return r;
   }
   
   int seek(int fdnum, u_int offset)
   {
   	int r;
   	struct Fd *fd;
   	//writef("seek() come 1 %x\n",(int)env);
   	if ((r = fd_lookup(fdnum, &fd)) < 0)
   		return r;
   	//writef("seek() come 2 %x\n",(int)env);
   	fd->fd_offset = offset;
   	//writef("seek() come 3 %x\n",(int)env);
   	return 0;
   }
   
   
   int fstat(int fdnum, struct Stat *stat)
   {
   	int r;
   	struct Dev *dev;
   	struct Fd *fd;
   
   	if ((r = fd_lookup(fdnum, &fd)) < 0
   	||  (r = dev_lookup(fd->fd_dev_id, &dev)) < 0)
   		return r;
   	stat->st_name[0] = 0;
   	stat->st_size = 0;
   	stat->st_isdir = 0;
   	stat->st_dev = dev;
   	return (*dev->dev_stat)(fd, stat);
   }
   
   int stat(const char *path, struct Stat *stat)
   {
   	int fd, r;
   
   	if ((fd = open(path, O_RDONLY)) < 0)
   		return fd;
   	r = fstat(fd, stat);
   	close(fd);
   	return r;
   }
   ```

   



