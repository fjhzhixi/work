---
title: OS-lab3
date: 2019-04-13 08:02:53
tags: 
- OS
- env
categories: 
- OS
---

# env进程结构管理

先给大家安利一个神器：`Understand`，有了它，妈妈再也不担心我源码看吐了

首先神图镇楼

![](OS-lab3\mm.png)

## 基本常数定义

### 最大进程数目

```c
// env.h
#define LOG2NENV	10
#define NENV		(1<<LOG2NENV)
```

即最多有2^10即1024个进程

### 进程状态

```c
#define ENV_FREE	0
#define ENV_RUNNABLE		1
#define ENV_NOT_RUNNABLE	2
```

## 基本数据结构

### env进程管理块

```c
// env.h
struct Env {
	struct Trapframe env_tf; 	   // 保存进程切换前后的寄存器值（即所谓的保存上下文）
	LIST_ENTRY(Env) env_link;	   // 在空闲进程队列env_free_list中链接下一项的指针
	u_int env_id;                  // 进程id号（每个进程的唯一标识）
	u_int env_parent_id;           // 父进程的id号 
	u_int env_status;              // 进程的状态
	Pde  *env_pgdir;               // 进程的页目录的内核中的起始虚拟地址 
	u_int env_cr3;				  // 进程的页目录的起始物理地址（即cr3寄存器中存储的值）
	LIST_ENTRY(Env) env_sched_link;// 在调度队列env_sched_list中链接下一项的指针
	u_int env_pri;				  // 进程的优先级，定义为进程可用的时间片个数
    // 之后的lab4再补充以下结构
	u_int env_ipc_value;           
	u_int env_ipc_from;           
	u_int env_ipc_recving;        
	u_int env_ipc_dstva;		
	u_int env_ipc_perm;		
	u_int env_pgfault_handler;    
	u_int env_xstacktop;          
	u_int env_runs;		
}；
// 数据结构展开
struct Trapframe { 
	unsigned long regs[32];
	unsigned long cp0_status;
	unsigned long hi;
	unsigned long lo;
	unsigned long cp0_badvaddr;
	unsigned long cp0_cause;
	unsigned long cp0_epc;	//中断异常时执行的指令地址
	unsigned long pc;		//在进程切换回来时应该开始执行的地址
};
#define	LIST_ENTRY(type)						\
struct {								\
	struct type *le_next;	/* next element */			\
	struct type **le_prev;	/* address of previous next element */	\
}
```

### env_free_list

```c
// env.c
static struct Env_list env_free_list;
```

该结构的作用将**待分配**的空闲`env`控制块串成链表形式，该链表中的`env`控制块都是**`ENV_FREE`状态**

### env_sched_list

```c
// env.c
static struct Env_list env_sched_list[2];
```

该结构的作用是将已经处于**可运行状态**的`env`控制块串成链表形式以便调度程序来调度运行（**本质上就是分配给时间片**），该链表中的`env`控制块都是**`ENV_RUNNABLE`状态**

具体使用详见进程调度算法

### 进程的状态转移

![](OS-lab3\env_status.png)

注 ： **对用户而言只能调用`create`,不能使用`alloc`所以在其看来进程一旦创建便处于可执行状态了**

？？或者在`env_alloc`中设置为`ENV_NOT_RUNNABLE`,把设置`status = ENV_RUNNABLE`放在`create`层次其实更好？？

**此处有个坑(测试杀我) ： 评测函数env_check()中会调用env_alloc()，所以会创建一堆只建立了页表但是没有加载可执行文件的残疾进程，所以如果你在调度时只是简单的判断是否是`RUNNABLE`状态，则会让这些进程运行，然后就会报`TOO LOW`的错误**

## 对进程的操作

### 初始化控制结构

#### 构建控制块数组

1. 函数实现

   **该函数的流程与物理内存控制块`pages`的构建流程基本一样**

```c
pmap.c
void mips_vm_init() {
	// code
	envs = (struct Env *)alloc(NENV * sizeof(struct Env), BY2PG, 1);
    n = ROUND(NENV * sizeof(struct Env), BY2PG);
    boot_map_segment(pgdir, UENVS, n, PADDR(envs), PTE_R);
}  
```

2. 图示分析

   ![](OS-lab3\envs.png)

#### 初始化控制结构

1. 函数实现

```c
// env.c
void env_init(void)
{
	int i;
    // 初始化空闲进程块链表
	LIST_INIT(&env_free_list);
    // 初始化待调度进程控制块链表
	LIST_INIT(&env_sched_list[0]);
	LIST_INIT(&env_sched_list[1]);
	// 初始时所有的进程控制块都是空闲的
	for(i=NENV-1;i>=0;i--){
		envs[i].env_status = ENV_FREE;
		LIST_INSERT_HEAD(&env_free_list,&envs[i],env_link);
	}
}
```

### 创建一个进程

```c
//env.c
void env_create_priority(u_char *binary, int size, int priority)
{
	struct Env *e;
	//分配进程控制块
	if(env_alloc(&e,0)<0){
		printf("Sorry,env can't create because alloc env failed!\n");
		return;
	}
    //设计进程优先级，在我们的系统中即是一次可以使用的时间片个数
	e->env_pri = priority；
    //加载二进制镜像
	load_icode(e,binary,size);
    //将加载完之后的进程加入等待被调度的链表中
	LIST_INSERT_HEAD(&env_sched_link[0], &e, env_sched_link);
}
```



#### 函数调用分析

![](OS-lab3\create_call.png)

#### 流程分析

##### 分配进程所需资源

即函数`env_alloc()`的作用，分配进程运行的必需资源，详细分析如下：

1. 函数流程 ：

   1. 从进程控制块空闲链表中取出一个
   2. **使用`snv_setup_vm`分配空间资源**
   3. 填写控制块中的各项数值

   ```c
   //env.c
   int env_alloc(struct Env **new, u_int parent_id)
   {
   	int r;
   	struct Env *e;
       // 从空闲进程块中分配一个控制块
   	if((e=LIST_FIRST(&env_free_list))==NULL){
   		printf("Sorry,alloc env failed!\n");
   		return -E_NO_FREE_ENV;
   	}
       // 为进程分配运行必需的空间资源
   	env_setup_vm(e);
       // 填写控制块的项
   	e->env_parent_id = parent_id;
   	e->env_status = ENV_RUNNABLE;
   	e->env_runs = 0;
   	e->env_id = mkenvid(e);			    //生成唯一的进程标识符
       e->env_tf.cp0_status = 0x10001004;
   	e->env_tf.regs[29] = USTACKTOP;		//填写栈寄存器
   	// 将使用过的控制块从空闲链表中移除
   	*new = e;
   	LIST_REMOVE(e,env_link);
   	return 0;
   }
   ```

2. 重点说明 ：

   1. `env_setup_vm()`函数 ：

      该函数作用是建立进程的页目录结构

      注意点 ：

      * 对进程页目录的初始化（一部分来自拷贝了内核页目录）
      * **自映射机制的构建**

      ```c
      //env.c
      static int env_setup_vm(struct Env *e)
      {
      	int i, r;
      	struct Page *p = NULL;
      	Pde *pgdir;
          // 给进程的页目录分配一个物理页
      	if ((r = page_alloc(&p)) < 0) {
      		panic("env_setup_vm - page_alloc error\n");
      		return r;
      	}
      	p->pp_ref++;
      	pgdir = (Pde *)page2kva(p);//将分配的物理页映射到kseg0中的一个虚拟地址方便之后填写
        	/* 之后的工作是对这个进程的页表中的项进行填写
        	 * 注意填写要区分
        	 * 1.没有在内核页表中完成映射的虚拟空间（UTOP向下：用户堆栈区) : 清空初始化
        	 * 2.已经在内核页表中映射好的虚拟空间（UTOP向上：ENVS,PAGES,USER VPT, kseg） ： 拷贝内核中对应页表项
        	 * 注：ENV,PAGES的映射是在mips_vm_init中通过boot_map_segment实现的
        	*/
      	for (i = 0; i < PDX(UTOP); i++) {
      		pgdir[i] = 0;
          }
          
      	for (i = PDX(UTOP); i <= PDX(~0); i++) {
      		pgdir[i] = boot_pgdir[i];
      	}
          // 填写控制块信息
      	e->env_pgdir = pgdir;	//进程页目录的起始虚拟内核地址
      	e->env_cr3   = PADDR(pgdir); //进程页目录的起始物理地址
      	// 之后的两条语句完成自映射机制
          e->env_pgdir[PDX(VPT)]   = e->env_cr3 | PTE_V;//将（进程页目录的起始物理地址）以（只读的权限）写入页目录中（内核空间的页目录起始虚拟地址VPT）对应的项
          e->env_pgdir[PDX(UVPT)]  = e->env_cr3 | PTE_V | PTE_R;//将（进程页目录的起始物理地址）以（读写的权限）写入页目录中（用户空间的页目录起始虚拟地址UVPT）对应的项
      	return 0;
      }
      ```

   2. `mkenvid()`函数

      该函数的作用是生成唯一的进程标识符

      * 生成标识符 ：标识符分为两段

        * 使用`next_env_id`静态递增变量来**保证不同进程的进程号一定不同**

          占据高位段

        * 使用`idx`即该进程控制块在`envs[]`的数组下标段来**保证可以从进程号获得`env`控制块**

          占据低11位段（因为只有1024项）

      ```c
      //env.c
      u_int mkenvid(struct Env *e)
      {
      	static u_long next_env_id = 0;
      	u_int idx = e - envs;
          //拼接两部分
      	return (++next_env_id << (1 + LOG2NENV)) | idx;
      }
      int envid2env(u_int envid, struct Env **penv, int checkperm)
      {
      	struct Env *e;
          //当前第一个进程还未创建其他进程
      	if (envid == 0) {
      		*penv = curenv;
      		return 0;
      	}
          //依据低11位来取出index获得env结构体
      	e = &envs[ENVX(envid)];
          //该进程未使用，
          //或者唯一进程标识符不同，
          //则取进程失败
      	if (e->env_status == ENV_FREE || e->env_id != envid) {
      		*penv = 0;
      		return -E_BAD_ENV;
      	}
          //取出进程不是由当前进程创建的，即父子关系不对，则取出失败
          if (checkperm && e != curenv && e->env_parent_id != curenv->env_id){
              *penv = 0;
              return -E_BAD_ENV;
          }
      	*penv = e;
      	return 0;
      }
      ```

   3. `e->env_tf.cp0_status = 0x10001004;`语句：

      待施工....。

##### 加载二进制映像

即函数`load_icode(struct Env *e, u_char *binary, u_int size)`的作用 ：

1. 函数流程

   ```c
   //env.c
   static void load_icode(struct Env *e, u_char *binary, u_int size)
   {
   	struct Page *p = NULL;
   	u_long entry_point;
   	u_long r;
       u_long perm;
       
   	if(page_alloc(&p)<0){
   		printf("Sorry,alloc page failed!\n");
   		return;
   	}
   	//为用户栈空间分配一页物理内存并完成从[USTACKTOP-BY2PG, USTACKTOP]到分配物理空间的二级页结构映射
       //即构建进程的运行栈空间
   	perm = PTE_V|PTE_R;
   	page_insert(e->env_pgdir,p,USTACKTOP-BY2PG,perm);
       //加载二进文件
       r = load_elf(binary,size,&entry_point,e,load_icode_mapper);
   	if(r<0){
   		printf("Sorry.load entire image failed!\n");
   		return;
   	}
   	//
   	e->env_tf.pc = entry_point;
   }
   ```

   

2. 重点说明

   1. `load_elf()`函数 ：

      * 参数含义 ：

        * `binary` ：待加载的二级制`ELF`文件的起始虚拟地址
        * `size` ： `ELF`文件的字节数目
        * `entry_point` ：程序的入口地址
        * `*map`：函数指针，在此处即是调用`load_icode_mapper()`

      * 函数作用 ：

        **解析`ELF`文件头信息**并把它加载到指定的虚拟内存区域，并建立对物理空间的二级页表映射

      ```c
      //kernel_elfloader.c
      int load_elf(u_char *binary, int size, u_long *entry_point, void *user_data,
      			 int (*map)(u_long va, u_int32_t sgsize,
      						u_char *bin, u_int32_t bin_size, void *user_data))
      {
          //解析ELF文件头的信息（具体结构在下文）
      	Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;
      	Elf32_Phdr *phdr = NULL;
      	u_char *ptr_ph_table = NULL;
      	Elf32_Half ph_entry_count;
      	Elf32_Half ph_entry_size;
      	int r;
      	//判断ELF文件是否合法
      	if (size < 4 || !is_elf_format(binary)) {
      		return -1;
      	}
      	//获得segment结构头的起始地址指针
      	ptr_ph_table = binary + ehdr->e_phoff;
          //获得segment段的数目
      	ph_entry_count = ehdr->e_phnum;
          //获得一个segment结构头的大小
      	ph_entry_size = ehdr->e_phentsize;
      	//遍历所有segment结构头
      	while (ph_entry_count--) {
      		phdr = (Elf32_Phdr *)ptr_ph_table;
              //如果该segment的类型是加载类型
      		if (phdr->p_type == PT_LOAD) {
                  //将该segment段加载到其头结构指出的虚拟地址空间并建立二级页表映射
      			r = map(phdr->p_vaddr, phdr->p_memsz,
      					binary + phdr->p_offset, phdr->p_filesz, user_data);
      			if (r < 0) {
      				return r;
      			}
      		}
      		//指下一个segment结构头
      		ptr_ph_table += ph_entry_size;
      	}
      	// 程序入口地址
      	*entry_point = ehdr->e_entry;
      	return 0;
      }
      ```

   2. `load_icode_mapper()`函数 ：

      * 参数含义 ：

        * `va` : 该段要加载到的虚拟地址
        * `sgsize` : 该段在内存中的大小
        * `bin` : 该段在`ELF`文件中的内容的起始地址
        * `bin_size` : 该段在文件中的大小

      * 函数作用 ：

        **将`ELF`文件中的一个`segment`加载到指定的虚拟地址处并建立好对应的二级页表映射（主要通过`page_inset`建立），若内存中大小比在文件中大，则多余的位用0补**

      ```c
      //env.c
      static int load_icode_mapper(u_long va, u_int32_t sgsize,
      							 u_char *bin, u_int32_t bin_size, void *user_data)
      {
      	struct Env *env = (struct Env *)user_data;
      	struct Page *p = NULL;
      	u_long i;
      	int r;
          //获得va在其所在页中的偏移offset
      	u_long offset = va - ROUNDDOWN(va, BY2PG);
      	//为二进制文件分配存储的物理页
      	for (i = 0; i < bin_size; i += BY2PG) {
      		if(page_alloc(&p)<0){
      			printf("Sorry,alloc page failed!\n");
      			return -E_NO_MEM;
      		}
      		p->pp_ref++;
              //void bcopy(const void *src, void *dst, size_t len)
              //从src（虚拟源地址）复制len个字节到dst(虚拟目标地址)
              //下面的两个copy不好理解，具体看下图
              //总的来说，就是比较剩下待拷贝的数据大小与页的大小来决定拷贝的数据量，然后逐页拷贝
              //对第一个分配的页，可能页不从首地址开始，所以要特殊考虑
      		if(i==0)
      			bcopy(bin,(char *)page2kva(p)+offset,((BY2PG-offset)<bin_size-i)?(BY2PG-offset):(bin_size - i));
      		else
      			bcopy(bin+i-offset,(char *)page2kva(p),(BY2PG<bin_size-i+offset)?BY2PG:(bin_size-i+offset));
              //拷贝完之后p对应物理页中存储二进制数据
              //然后以读写的权限通过该进程的页表将va+i（即当前一页二进制数据应该加载到虚拟地址）映射到p对应的物理页上
      		r = page_insert(env->env_pgdir,p,va+i,PTE_V|PTE_R);
      		if(r<0){
      			printf("Sorry,insert a page is failed!\n");
      			return -E_NO_MEM;
      		}
      	}
      	//当该segment在内存中的大小大于在文件中时
          //给多余的地址空间赋值为0（在page_alloc()中就已经清0） 
      	while (i < sgsize) {
      		if(page_alloc(&p)<0){
      			printf("Sorry,alloc page failed!\n");
      			return -E_NO_MEM;
      		}
      		p->pp_ref++;
      		r = page_insert(env->env_pgdir,p,va+i,PTE_V|PTE_R);
      		if(r<0){
      			printf("Sorry,alloc page failed!\n");
      			return -E_NO_MEM;
      		}
      		i+=BY2PG;
      	}
      	return 0;
      }
      ```

   3. **图示**

      * `ELF`文件结构示意图

        ![](OS-lab3\ELF.png)

        ```c
        // ELF文件头结构
        typedef struct {
        	unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
        	Elf32_Half	e_type;			/* Object file type */
        	Elf32_Half	e_machine;		/* Architecture */
        	Elf32_Word	e_version;		/* Object file version */
        	Elf32_Addr	e_entry;		/* Entry point virtual address */
        	Elf32_Off	e_phoff;		/* Program header table file offset */
        	Elf32_Off	e_shoff;		/* Section header table file offset */
        	Elf32_Word	e_flags;		/* Processor-specific flags */
        	Elf32_Half	e_ehsize;		/* ELF header size in bytes */
        	Elf32_Half	e_phentsize;	/* Program header table entry size */
        	Elf32_Half	e_phnum;		/* Program header table entry count */
        	Elf32_Half	e_shentsize;	/* Section header table entry size */
        	Elf32_Half	e_shnum;		/* Section header table entry count */
        	Elf32_Half	e_shstrndx;		/* Section header string table index */
        } Elf32_Ehdr;
        ```

        ```c
        //一个segment结构头的结构，其管理一份segment段
        typedef struct {
        	Elf32_Word	p_type;			/* Segment type */
        	Elf32_Off	p_offset;		/* Segment file offset */
        	Elf32_Addr	p_vaddr;		/* Segment virtual address */
        	Elf32_Addr	p_paddr;		/* Segment physical address */
        	Elf32_Word	p_filesz;		/* Segment size in file */
        	Elf32_Word	p_memsz;		/* Segment size in memory */
        	Elf32_Word	p_flags;		/* Segment flags */
        	Elf32_Word	p_align;		/* Segment alignment */
        } Elf32_Phdr;
        ```

      * 加载一个`segment`的示意图

        ![](OS-lab3\segment1.png)

        ![](OS-lab3\segment2.png)

        ![](OS-lab3\segment3.png)

### 进程运行

主要使用`env_run(e)`方法运行线程e ：

* 函数流程 ：
  1. 从内存中取出当前线程保存上下文的地方
  2. 将上下文信息存入当前线程的控制块的`env_tf`结构中以便他切走
  3. 设置`e`为当前进程，并加载`e`进程的信息

```c
//env.c
#define TIMESTACK 0x82000000
//由当前进程切换到e进程执行
void env_run(struct Env *e)
{
    //old即为当前进程进程的上下文存储的地方
	struct Trapframe *old = (struct Trapframe *)(TIMESTACK-sizeof(struct Trapframe));
	if(curenv){
        //将当前进程的上下文保存其进程控制块的的env_tf结构体变量中
		bcopy(old,&(curenv->env_tf),sizeof(struct Trapframe));
        /* 填写存储进程再次切换回来时应该开始执行的指令地址
         * 我们的进程切换时由于分配时间片使用完之后导致的时钟中断
         * cp0_epc存储的是产生中断时的执行指令地址
         * 所以它就是我们进程再次切回时应该执行的指令，将其保存在env_tf。pc中
        */
		curenv->env_tf.pc = old->cp0_epc;
	}
	//将当前进程切换到e
	curenv = e;
	curenv->env_runs ++;
	//（即切换页目录）加载要运行进程的页目录（env_cr3中保存了页目录的起始物理地址)
	lcontext(KADDR(curenv->env_cr3));	
    //加载当前要运行的进程的上下文
	env_pop_tf(&(curenv->env_tf),GET_ENV_ASID(curenv->env_id));	
}
```

* 分析`lcontext(KADDR(curenv->env_cr3))`

  该函数的所用其实是加载要运行的进程的页目录虚拟地址到`mCONTEXT`这个全局变量中，其实也是构造进程运行环境的一部分

  让我们先追踪溯源 ：

  1. `lcontext`的函数定义：

     ```c
     //env_asm.S
     LEAF(lcontext)
     		.extern	mCONTEXT
     		//该操作即是将该进程的页目录的内核虚拟地址存入mCONTEXT中
     		sw		a0,mCONTEXT//a0代表第一个参数，即为KADDR(curenv->cr3)
     		jr	ra
     		nop
     END(lcontext)
     ```

  2. 出现了变量`mCONTEXT`我们继续来追踪溯源 ：

     * 定义处 ：

       ```c
       //start.S第45行
       			.globl mCONTEXT
       mCONTEXT:
          			.word 0//即初值为0的一个字大小的全局变量
       ```

     * 使用处 ：我们目前只关注在这之前改变他值的操作，暂时不关心使用

       ```c
       //pmap.c第160行
       pgdir = alloc(BY2PG, BY2PG, 1);
       mCONTEXT = (int)pgdir;//mCONTEXT中保存的是内核中的页目录的起始内核虚拟地址
       ```

* 分析`env_pop_tf(&(curenv->env_tf),GET_ENV_ASID(curenv->env_id));`

  * 函数定义

    ```c
    //env_asm.S第15行
    LEAF(env_pop_tf)
    /*传入参数
     *$a0 : 当前进程的env->env_tf起始地址，即上下文信息
     *$a1 : 当前进程的唯一标识（经过一定处理）//我也不知道他为什么要处理。。。
     */
    .set	mips1          
    		//1:	j	1b
    	nop    
    		move	k0,a0 
    		mtc0	a1,CP0_ENTRYHI
    		// 是设置中断位，从禁止中断（即之前在时钟中断进入的核心态）到允许中断
    		mfc0	t0,CP0_STATUS                    
    		ori		t0,0x3                          
    		xori	t0,0x3                          
    		mtc0	t0,CP0_STATUS                    
    		//加载进程控制块的储存上下文信息的tf中的值到对应的寄存器中去	
    		lw	v1,TF_LO(k0)                                       
    		mtlo	v1                               
    		lw	v0,TF_HI(k0)                     
    		lw	v1,TF_EPC(k0)                    
    		mthi	v0                               
    		mtc0	v1,CP0_EPC  
    		lw	$31,TF_REG31(k0)                 
    		...........                
    		lw	$1,TF_REG1(k0)
    		lw	k1,TF_PC(k0)
    		lw	k0,TF_STATUS(k0)                 
    		nop
            //跟新CP0协处理器的状态
    		mtc0	k0,CP0_STATUS
    		j	k1
    		rfe
    		nop
    END(env_pop_tf)
    ```

    

### 进程调度

进程调度主要依赖于函数`sched_yield()`以及数据结构`env_sched_list[2]` :

* 调度原理
* 调度算法
* 实现
* 图示

