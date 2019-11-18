
## ext2/3和ext4保存数据方式的区别


ext2和ext3的i_block的定义都了类似，如下：
```
struct ext2_inode{
        ........
	__le32 i_block[EXT2_N_BLOCKS];
        ........
}
```
```c
struct ext4_inode{
        ........
	__le32 i_block[EXT4_N_BLOCKS];
        ........
}
```
*EXT2_N_BLOCKS*和*EXT4_N_BLOCKS*的默认值都是15，以*EXT4_N_BLOCKS*为例：
```c
#define  EXT4_NDIR_BLOCKS    12
#define  EXT4_IND_BLOCK      EXT4_NDIR_BLOCKS
#define  EXT4_DIND_BLOCK      (EXT4_IND_BLOCK + 1)
#define  EXT4_TIND_BLOCK      (EXT4_DIND_BLOCK + 1)
#define  EXT4_N_BLOCKS      (EXT4_TIND_BLOCK + 1)
```

ext2/3中i_block的15项分为下面四种：
- 前12项： 直接存放块的位置
- 第13项目：指向块，但是块里面不放数据，而是放数据的位置，称为间接块
- 第14项目：间接块的间接块
- 第15项目：间接块的间接块的间接块
如下图：
![](https://user-images.githubusercontent.com/12036324/68989363-995bfc80-0880-11ea-90ec-e5beb19704d0.png)

所以就是最多能存储`12 * 4 + 4096  + 4096 * 4096 + 4096 * 4096*4096 ~= 64G`
但是这样散着放，对于大文件来说要多次读写才能找到对应的人块，访问速度会变慢。



我们可以知道i_block的size是60byte（int32为4byte，一共15位）。
ext4新增了一个Extends的概念：
*Extends*可用于存放连续的数据块，增加读写性能，减少文件碎片。
![](https://user-images.githubusercontent.com/12036324/68989633-9531de00-0884-11ea-8968-16d98b7dea28.png)
i_block的分为：一个*Extend Header*和四个*Extend Entry*，他们每个都为12byte，五个加起来正好60byte。为什么是12byte，看定义：
```c
struct ext4_extent_header {
  __le16  eh_magic;  /* probably will support different formats */
  __le16  eh_entries;  /* number of valid entries */
  __le16  eh_max;    /* capacity of store in entries */
  __le16  eh_depth;  /* has tree real underlying blocks? */
  __le32  eh_generation;  /* generation of the tree */
};
```


下面位数据节点：

```c
struct ext4_extent {
  __le32  ee_block;  /* first logical block extent covers */
  __le16  ee_len;    /* number of blocks covered by extent */
  __le16  ee_start_hi;  /* high 16 bits of physical block */
  __le32  ee_start_lo;  /* low 32 bits of physical block */
};
```
一个extent能存储128M，这是因为ee_len是16位，但是第一位（最高位）在预分配特性中用来标识这个extent是否被初始化，如果块大小为4K的话，那么一个extent最大可以表示：`2 ** 15 * 4 / 1024 = 128M`

索引节点的结构为：
```c
struct ext4_extent_idx {
  __le32  ei_block;  /* index covers logical blocks from 'block' */
  __le32  ei_leaf_lo;  /* pointer to the physical block of the next *
         * level. leaf or next index could be there */
  __le16  ei_leaf_hi;  /* high 16 bits of physical block */
  __u16  ei_unused;
};
```
除了根节点，其他的节点都保存在一个块4K里面，4K扣除ext4_extent_header的12个byte，剩下还能够放340项，每个extent最大能表示128M的数据。340个extent最多能达到42.5G。

## 参考：
- [Linux.ext4文件系统.inode和extent](https://blog.csdn.net/stringNewName/article/details/73740155)
