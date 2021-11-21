```c
static int
mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
  char *a, *last;
  pte_t *pte;

  a = (char*)PGROUNDDOWN((uint)va);//get starting address
  last = (char*)PGROUNDDOWN(((uint)va) + size - 1);//get ending address (which is starting address if size=1)
  for(;;){//for each page...
    if((pte = walkpgdir(pgdir, a, 1)) == 0)//get i1 row address (using walkpgdir)
      return -1;//如果walkpgdir返回零说明寻找PTE失败，出错，返回-1
    if(*pte & PTE_P)//按位与取*pte最低位表示是否已在页表中
      panic("remap");//如果已在页表中，触发页表重填
    *pte = pa | perm | PTE_P;//按位或，为当前PTE赋值，给予物理页号（pa），标注在页表中存在（PTE_P）
    if(a == last)//全部分配完成，退出循环
      break;
    a += PGSIZE;//PGSIZE=4096，即PTE第十三位加1，即PPN加1
    pa += PGSIZE;
  }
  return 0;
}

static pte_t *
walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
  pde_t *pde;
  pte_t *pgtab;

  pde = &pgdir[PDX(va)];//PDX(va)将va右移22位，获得二级页表页目录号
  if(*pde & PTE_P){//在页表中
    pgtab = (pte_t*)P2V(PTE_ADDR(*pde));
  } else {
    if(!alloc || (pgtab = (pte_t*)kalloc()) == 0)
      return 0;
    // Make sure all those PTE_P bits are zero.
    memset(pgtab, 0, PGSIZE);
    // The permissions here are overly generous, but they can
    // be further restricted by the permissions in the page table
    // entries, if necessary.
    *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
  }
  return &pgtab[PTX(va)];
}
```

`mappages`（1679）做的工作是在页表中建立一段虚拟内存到一段物理内存的映射。它是在页的级别，即一页一页地建立映射的。对于每一个待映射虚拟地址，`mappages` 调用 `walkpgdir` 来找到该地址对应的 PTE 地址。然后初始化该 PTE 以保存对应物理页号、许可级别（`PTE_W` 和/或 `PTE_U`）以及 `PTE_P` 位来标记该 PTE 是否是有效的（1691）。

每个 PTE 都包含一些标志位，说明分页硬件对应的虚拟地址的使用权限。PTE_P 表示 PTE 是否陈列在页表中：如果不是，那么一个对该页的引用会引发错误（也就是：不允许被使用）。PTE_W 控制着能否对页执行写操作；如果不能，则只允许对其进行读操作和取指令。PTE_U 控制着用户程序能否使用该页；如果不能，则只有内核能够使用该页。

`walkpgdir`（1654）模仿 x86 的分页硬件为一个虚拟地址寻找 PTE 的过程（见图表2-1）。`walkpgdir` 通过虚拟地址的前 10 位来找到在页目录中的对应条目（1659），如果该条目不存在，说明要找的页表页尚未分配；如果 `alloc` 参数被设置了，`walkpgdir` 会分配页表页并将其物理地址放到页目录中。最后用虚拟地址的接下来 10 位来找到其在页表中的 PTE 地址（1672）。

* **mappages()函数中PGROUNDDOWN()函数的作用是什什么？为何需要调用该函数？**

  `PGROUNDUP` 和 `PGROUNDDOWN` 是将地址四舍五入到 `PGSIZE` 的倍数的宏。这些通常用于获取页面对齐的地址。 `PGROUNDUP` 会将地址四舍五入为 `PGSIZE` 的较高倍数，而 `PGROUNDDOWN` 会将其四舍五入为 `PGSIZE` 的较低倍数。

  让我们举一个例子，如果 `PGROUNDUP` 在 `PGSIZE` 4KB 地址为 620 的系统上被调用:

    *PGROUNDUP(620) ==> ((620 + (4096 -1)) & ~(4096)) ==> 4096*

    *地址 620 向上舍入为 4096。*

    *同样对于 `PGROUNDDOWN` 考虑:*

    *PGROUNDDOWN(4100) ==> (4100 & ~(4096-1)) ==> 4096*

    *地址 4100 向下舍入为 4096*

  使用 PGROUNDDOWN(va) 将出错的虚拟地址向下舍入到页面边界

  

* **mappages()函数中“*pte = pa | perm | PTE_P;”代码的含义是什么？**

  按位或，为当前PTE赋值，给予物理页号（pa），标注在页表中存在（PTE_P）

  perm是什么？猜测是PTE中其他标志位（待查证）(查证结果：应该是的)  
  
  11/21 15:21：perm表示该页的权限：“同时赋予该页的权限perm” https://www.cnblogs.com/icoty23/p/10993861.html#_label1
  
  
  
* **walkpgdir()函数中“*pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;”代码的含义是什什么？**

  当pde不可用时（即 pde & PTE_P 为0时），上文 pgtab = (pte_t)kalloc() 创造了新的二级页表：pgtab；然后清空了二级页表页目录 memset(pgtab, 0, PGSIZE)；之后此处是将pde指向了目标二级页表页目录号，并设定此pde有效（PTE_P），可写入（PTE_W），用户态可用（PTE_U）。

  

* **virtual address与physical address的映射关系是什什么？对虚拟内存是如何实现的？(page swap)**

  ```c
  #define V2P(a) (((uint) (a)) - KERNBASE)//虚地址到实地址
  
  #define P2V(a) ((void *)(((char *) (a)) + KERNBASE))//实地址到虚地址
  ```

  实地址到虚地址加了一个偏移量KERNBASE，此偏移量表示内核虚地址起始地址；虚地址到实地址反之。（静态重定位）

  （啥叫如何实现？上面写的算是回答了这个问题吗？）

  11/21 15:19（大概明白了问题什么意思，需要细看kalloc函数，待补充）

  

* **进阶：linux代码中virtual address与physical address是如何映射的？如何实现了了虚拟内存(page swap)？请结合相关源码[mm/vmscan.c]进行分析。**
  （待补充）

  

### **walkpgdir注释待补充**


