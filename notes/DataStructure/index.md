---
date: 2022-12-12
title: DataStructure
---


# Data Structure

## 1 线性结构

### 1.1 列表List

### 1.2 堆栈Stack

### 1.3 队列Queue


## 2 树tree

### 2.1 二叉树

- **前中出树**
- n2 = n0 - 1
  - n2+n1+n0  = n2*2+n1 + 1
- 同构的概念

### 2.2 满二叉树、完全二叉树

### 2.3 搜索树BST

- 遍历
  - 深度优先遍历[前序、中序、后序]（递归、非递归）
  - 广度优先遍历（利用队列进行层序遍历）
- 搜索
  - 从上到下依次比较判断（~O(logn))
- 插入
  - 从上到下依次比较，判断后插入
- 删除
  - 先找到要删除的点，然后删除时考虑三种情况
    - 无子树
    - 一个子树
    - 两个子树

### 2.4 平衡二叉树AVL

- 遍历同理
- 搜索同理，且AVL相比BST的时间复杂度更稳定
- 插入：插入时需要涉及到平衡树的调整
  - 调整的四个方法：RR，LL，RL，LR
  - 调整的基本策略：**自插入的叶节点开始向上递归**，可以在上层传参时传入**“父节点的左右孩子”的地址**，即 `&(root->rchild)`以达到不用在结构体或类中定义父节点指针就能调整的实现
  - 一般来说`RL_rotate(ptr_AVLTree)`等价于先`LL_rotate(&((*ptr_AVLTree)->rchild))`再`RR_rotate(ptr_AVLTree)`，LR同理
  - 记得在调整时及时更新高度

```c
void AVL_insert(address_AVL_node *ptr_AVLTree, AVL_ElementType value)
{
    address_AVL_node temp_AVLTree = *ptr_AVLTree;
    if (!temp_AVLTree)
    {
        *ptr_AVLTree = newNode(value);
        // return;
    }
    else if (temp_AVLTree->data < value)
    {
        address_AVL_node *address_rchild = &(temp_AVLTree->rchild);
        AVL_insert(address_rchild, value);

        int lHeight = getHeight_read(temp_AVLTree->lchild);
        int rHeight = getHeight_read(temp_AVLTree->rchild);
        // judge height
        if ((lHeight - rHeight) > 1 || (lHeight - rHeight) < -1)
        {
            if (value > temp_AVLTree->rchild->data)
            {
                // RR
                RR_rotate(ptr_AVLTree);
            }
            else if (value < temp_AVLTree->rchild->data)
            {
                // RL
                //RL_rotate(ptr_AVLTree);
                LL_rotate(&((*ptr_AVLTree)->rchild));
                RR_rotate(ptr_AVLTree);
            }
        }
    }
    else if (temp_AVLTree->data > value)
    {
        address_AVL_node *address_lchild = &(temp_AVLTree->lchild);
        AVL_insert(address_lchild, value);

        int lHeight = getHeight_read(temp_AVLTree->lchild);
        int rHeight = getHeight_read(temp_AVLTree->rchild);
        // judge height
        if ((lHeight - rHeight) > 1 || (lHeight - rHeight) < -1)
        {
            if (value > temp_AVLTree->lchild->data)
            {
                // LR
                //LR_rotate(ptr_AVLTree);
                RR_rotate(&((*ptr_AVLTree)->lchild));
                LL_rotate(ptr_AVLTree);
            }
            else if (value < temp_AVLTree->lchild->data)
            {
                // LL
                LL_rotate(ptr_AVLTree);
            }
        }
    }

    (*ptr_AVLTree)->height_rel = MAX(getHeight_read((*ptr_AVLTree)->lchild), getHeight_read((*ptr_AVLTree)->rchild)) + 1;
}
```

- 删除的情况过于复杂（6种）

### 2.5 哈夫曼树Huffman(要使得带权路径长度WPL最小)

- 概念
  - 设二叉树有n个叶节点，每个叶节点带有权值wk，从根节点到叶节点的长度为lk，则每个叶子节点的带权路径长度之和就是WPL。因此也叫**最优二叉树(WPL最小的二叉树)**
  - $WPL = \sum\limits_{k=1}^{n}w_kl_k$
- 特点：
  - 1.最优编码——WPL最小
  - 2.无歧义编码——前缀码：数据仅仅存在于叶子节点中
  - 3.**没有度为1的节点**
  - 满足1，2一定满足3，满足2，3不一定满足1
  - **n个叶结点的哈夫曼树共有2n-1个节点**
  - 哈夫曼树的任意非叶节点的左右子树交换后仍然是哈夫曼树
  - 对于**同一组权值**，存在**不止一种不同构的哈夫曼树**
- 哈夫曼树的构造：
  - 每次把**权值最小的两个树**合并，如1,2,3,4,5->3,3,4,5->6,4,5->6,9->15
  - 实现问题1，如何选取两个最小的：如果采用每次选取后排序取前二的方法会导致时间复杂度太高，可以考虑用**最小堆**实现

### 2.6 堆heap(完全二叉树)

堆通常是一个可以被看做一棵完全二叉树的数组对象

- 堆的创建
  - 由于是完全二叉树，一般采用数组存储，且从`array[1]`开始
  - `index`的左右子树分别是`2*index`和`2*index+1`
- 堆的插入
  - 先查到数组末尾，然后依次向上比较`index=index/2`
  - 循环条件一般是`index>1 && _data[index/2]>insert_value`
  - 知道找到一个合适的坐标，然后把值放进去，期间不用每次循环要交换值，只需要把上层(根)赋给下层(子树)即可
  - 即：
    - 先将元素插入到堆的末尾，即最后一个孩子之后。
    - 插入之后如果堆的性质遭到破坏，就将新插入的节点顺着其的父亲往上调整到合适位置。直到调到符合堆的性质为止。
- 堆的删除
  - 删除删去第一个元素，即`array[1]`
  - 随后开始令`index=1`，确保满足堆条件（最大或最小）的先提下逐层向下找到合适的下标，把`array[size- -]`赋到其中，过程中同理只需要把下层(子树)赋给上层(根)即可
  - 过程中要理清每个根**有几个子节点**
- 堆的建立（说明）
  - 如建立最大堆，即将已经存在的N个元素按照最大堆的要求存放在一个一维数组中
  - 两种方法
    - 方法1：一个个`insert`（插入）进去，这样时间的复杂度是O(n logn)
    - 方法2：先将N个元素按输入顺序存入，先满足**完全二叉树的结构特性**，然后再调整各个节点位置，以满足最大堆的**有序特性**。这一方法是线性时间复杂度[证明：[【数据结构】证明建堆的时间复杂度_柠檬叶子C的博客-CSDN博客](https://blog.csdn.net/weixin_50502862/article/details/122260229?ops_request_misc=&request_id=0b7741e3d46b46edb616fb53d3150768&biz_id=&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~koosearch~default-4-122260229-null-null.268^v1^control&utm_term=堆&spm=1018.2226.3001.4450)]
- 堆的建立：方法2的实现思路
  - 先将N个元素按输入顺序存入
  - 然后从最后一个有个子节点的根开始，按照数组下标的顺序往前依次执行下面的操作：
    - 由于是从下往上调整，所以每次调整的时候会发现左右子树都是按顺序排好的堆（抑或null），只需调整根就可以了
    - 调整根时的思路可以参考堆的删除的第二条：从根开始即找到一条合适的路径将根下放



## 3 优先队列Priority Queue

特殊的"队列", 出队顺序与入队顺序无关, 而是依照元素的**优先权(关键字)**大小出队

优先队列ADT是一种数据结构，它支持插入和删除最小/大值操作（返回并删除最小/大元素）或删除最大值操作（返回并删除最大元素）

- 插入
- 删除最小/大值

### 3.1 几种不同的实现方法

- 数组:
  - 插入: 插入到尾部~O(1)
  - 查找: 查找最大最小关键字~O(n)
  - 删除: 先查找[O(n)]再删除[O(n)], 同时还要考虑到移动元素~O(n)
- 链表:
  - 插入: 总是插入到头部~O(1)
  - 查找: 查找最大最小关键字~O(n)
  - 删除: 先查找[O(n)]再删除[O(1)], 但不用考虑移动元素~O(n)
- 有序数组:
  - 插入: 找到合适的位置~O(n)或O(logn), 移动元素插入~O(n)
  - 删除: 删去最后一个元素~O(1)
- 有序链表:
  - 插入: 找到合适的位置~O(n), 移动元素插入~O(1)
  - 删除: 删去最后一个元素~O(1)

### 3.2 用树实现(堆)

用搜索树实现的好处是无论是插入还是删除都是O(logn). 但是如果每次都要删除最大的值, 若是用搜索树会使得树的右节点逐渐变少, 直至树斜

**堆的有序性:** 1. 首先是一个**完全二叉树**, 2. 其次树中的任意节点是其子树所有根节点的最大值(或最小值)

- 最大堆(MaxHeap)
- 最小堆(MinHeap)

显然, 堆从根节点到任意节点都是有序的, 这对插入时的处理有指导作用

## 4 并查集

- 集合**并**、**查**某个元素属于什么集合

- 可以用**树结构**表示集合，树的每个节点代表一个集合元素。 一个树表示一个集合

- 已知一个节点，找其父节点是谁：父节点表示法——每个节点都指向其父节点

- 采用数组的存储形式

  <img src="C:\Users\rouge\AppData\Roaming\Typora\typora-user-images\image-20230825161858064.png" alt="image-20230825161858064" style="zoom:33%;" />

- 查找操作：

  - 先遍历一遍数组找到要查找元素的下标
  - 沿着元素的父节点下标往前找，知道小于0时停止

- 合并操作

  - 找到两个元素所在的集合的根节点的数组中的下标
  - 如果不相同则将一个根的父节点改为另一个根的下标
  - **改进：**可以将表示根节点的负数parent改成-n，其中n是其集合元素的个数，在合并时将小的树合并到大的树上

## 5 图

### 5.1 Intro

- 六度空间理论
- 从陈家庄到李家村走哪一条路最快——找最短路径
- 怎么修公路使得村村通的花费最少——最小生成树的问题

### 5.2 什么是图（Graph）

- 表示**多对多**的关系
- 包含：
  - 一组**顶点**：常用**V(Vertex)**表示顶点集合
  - 一组**边**：通常用**E(Edge)**表示边的集合，因为表示点与点之间的关系，因此常用顶点对表示：
    - **圆括号表示无向边**：(v, w) $\in$E，其中v, w $\in$ V（双向道路）
    - **尖括号表示有向边**：<v, w>表示从**v指向w**的边（单行线）
- 不考虑**重边**和**自回路**：
  - 不能有重边是指：**两个顶点之间默认只有一条无向边**
  - 不能有自回路是指：一个有向边一定是从一个顶点指向另一个顶点，**不可能指向自己**

### 5.3 抽象数据类型定义

- **<u>类型名称</u>**：图（Graph）
- **<u>数据对象集</u>**：G(V, E) 由一个**非空的有限顶点集合V**和一个有限边集合E组成（**至少有一个顶点**）
- **<u>操作集</u>**（部分）：对于任意图G $\in$ Graph，以及v $\in$ V，e $\in$ E
  - 建立并返回空图
  - 将v插入G
  - 将e插入G
  - 从v出发深度优先遍历图G（DFS）
  - 从v出发广度优先遍历图G （BFS）
  - 计算图G中顶点v到其他顶点的最短距离
  - 计算图G的最小生成树
  - …………

### 5.4 表示一个图

#### 邻接矩阵G\[N\]\[N\]——N个顶点从0到N-1编号

**G\[ i \]\[ j \] = 1 (若<v~i~, v~j~>是G中的边，注意是有向边)**

**G\[ i \]\[ j \] = 0 (否则)**

对于**网络（边有权重的图）**：把G\[ i \]\[ j \] 的值**从1改变定义为<v~i~, v~j~>的权重**即可

![image-20230827100430236](C:\Users\rouge\AppData\Roaming\Typora\typora-user-images\image-20230827100430236.png)

发现：

- 对角线上全是0——不能有**自回路**
- 关于对角线是对称的——因为这是一个无向图——同时也说明这个无向图的部分空间被浪费了——***对于无向图，怎么可以省一半空间呢***
- **<u>解决方法：用一个长度为N(N+1)/2的1为数组A存储：</u>**
  - **{G~00~, G~10~, G~11~, G~20~, G~21~, G~22~, G~30~, G~31~, ……, G~n-1~ ~n-1~}中的G~i~ ~j~在A中对应的下标为：**
  - **i上面元素的总个数\[ i * (i + 1) / 2 \]加上列号j**: 即：
  - ==<u>**\[ i * (i + 1) / 2 + j \]**</u>==

#### 邻接表G\[N\]——N个链表

- 方便找任一顶点的所有“邻接点”
- 不同与邻接矩阵的表示法（邻接矩阵要求图要足够稠密才合算），邻接表的表示要求图要足够稀疏才合算
  - 需要N个头指针+2E个结点（每个结点至少两个域）
- 方便计算任意一个顶点的**度**？
  - 无向图：Yes
  - 有向图：只方便**出度**；需要构建“逆邻接表”（存指向自己的边）来方便计算“入度”
- **<u>不方便</u>**检查任意一对顶点间是否存在边

#### 边列表

### 5.5 Search

#### Depth  First Searth(DFS)深度优先搜索

**\[回溯，堆栈stack\]**： 视力范围内所有灯都点亮后原路返回直至起点

```cpp
void DFS(Vertex V){
	visited(v) = true;
	for(V的每一个邻接点W){
		if(!visited(W))
			DFS(W);
	}
}
```

- **时间复杂度**
  - 用**邻接矩阵**存储：~**O(N^2^)**
  - 用**邻接表**存储：~**O(N+E)**

#### Breadth First Searth(BFS)广度优先搜索

**\[队列Queue\]：**先将第一个点入队，然后出队是把与之相连的邻接点一一入队，然后在一一出队，每次出队时同理把与出队元素相连的邻接点一一入队，往复循环。注意不要再将已经遍历过的元素再次入队即可

```cpp
void BFS(Vertex V){
	Queue.入队(V);
	Visited(V) = true;
	while(!Queue.IsEmpty()){
		V  = Queue.出队;
		for(V的每一个邻接点W){
			if(!Visited(W)){//如果W未被访问过
				Visited(W) = true;
				Queue.入队(W);
			}
		}
	}
	return;
}
```

发现上面的伪代码是入队时访问（考虑是否可以出队时访问）

**时间复杂度**

- 用**邻接矩阵**存储：~**O(N^2^)**
- 用**邻接表**存储：~**O(N+E)**

#### 为什么需要两种遍历

- 都是用于连的图或部分的遍历，dfs路径优先，bfs距离优先
- DFS目的地远的岔路少的比较好，BFS目的地近的而且岔路多的比较好
- DFS适合大爆搜，BFS适合求最短路径，最小步数的问题

#### 图不连通怎么办

##### 对于无向图

- **连通**：如果顶点V和W之间存在一条（无向）**路径**，则称V和W是连**通**的
- **路径**：顶点V到W的路径是**一系列顶点的集合**\{V, v1,v2,v3, …, vn, W\}，其中任意一对相邻的顶点间都有图中的边。如果顶点V到W之间的所有顶点都不相同，则称**简单路径**
- **路径的长度**：路径的长度是路径中的**边数**。如果带权，则路径的长度指的是所有边的**权重的和**
- **回路**：起点等于终点的**路径**
- **连通图**：图中任意两个顶点都**连通**
- **完全图**：假设一个图有n个顶点，那么如果**任意两个顶点之间都有边**的话，该图就称为**完全图**
- **连通分量**：无向图的**极大**连通子图——（无向图的概念）
  - 极大顶点数：再加一个顶点就不连通时顶点的个数
  - 极大边数：包含子图中所有顶点相连的所有边
  - <img src="C:\Users\rouge\AppData\Roaming\Typora\typora-user-images\image-20230827174256390.png" alt="image-20230827174256390" style="zoom:30%;" />

##### 对于有向图

- **强连通**：有向图中顶点V和W之间存在双向路径，则称V和W是强连通的
- **强连通图**：有向图中任意两顶点均强连通
- **强连通分量**：有向图的极大强连通子图
- <img src="C:\Users\rouge\AppData\Roaming\Typora\typora-user-images\image-20230827174642492.png" alt="image-20230827174642492" style="zoom:33%;" />
- 弱连通图：去掉边方向信息后是连通的，但加上边方向信息后又不是强连通的

### 5.6 最短路径问题：

#### 无权图的单源最短路径

![image-20230831203749179](C:\Users\rouge\AppData\Roaming\Typora\typora-user-images\image-20230831203749179.png)

- 主要采用BFS的框架，按照**递增**的顺序找出各个顶点的最短路
- 不再采用`_isvisited[]`数组来判断是否被访问过，而是采用`_dist[i]`数组来表示从源点到`vertex=i`点的最短路径长度，如果尚未被BFS访问到可以用一个异常值（-1，正无穷，负无穷）来表示
- 同时用`_path[W] = V`来表示从源点到点W的最短路径中的上一个节点是V，这样可以通过不断向前推都得到一条相反的路径，最后只需通过栈来倒序即可

```cpp
void Unweighted(Vertex S){
    queue.add(S);
    _dist[S] = 0;
    path[S] = ERROR;
    while(!queue.empty){
        V = queue.pop_back();
        for(V的每一个邻接点W){
            if(_dist[W]==ERROR){
                _dist[W] = _dist[V] + 1;
                path[W] = V;
                queue.add(W);
            }
        }
    }
}
```

#### 有权图的单源最短路径——“Dijkstra算法”

- 相比于无权图，不一定经过的顶点数最少
- 按照**递增**的顺序找出各个顶点的最短路
- ==<u>**Dijkstra算法**</u>==
- *补充：避免<u>负值圈 negative-cost cycle</u>（一直转圈赚钱）*![image-20230831204929408](C:\Users\rouge\AppData\Roaming\Typora\typora-user-images\image-20230831204929408.png)
- **把顶点一个一个往集合里面收，*令S={源点s+已经确定了最短路径的顶点vi}***
- 定义一个距离的数组`dist[]`，初始化时起点s的dist[s]=0，与s直接相连的邻接点w的dist[w] = Weight<s,w>，而与s没有直接的路相通时应定义为正无穷（考虑到下面要判断大小）  
- 同时用`_path[W] = V`来表示从源点到点W的最短路径中的上一个节点是V，这样可以通过不断向前推都得到一条相反的路径，最后只需通过栈来倒序即可
- ***不能解决有负边的情况***

```cpp
void Dijkstra(vertex s){
	while(1){
		vi = set(V-S) 中dist[·]最小值者;
        if(上面这样的vi不存在了，即set(V-S)=Ø)
            break;
		S[vi] = true;
		for[vi的每一个邻接点W & W∉set(S)]{
			if(dist[vi]+Weight<vi,w> 小于 dist[W]){
				dist[W] = dist[vi]+Weight<vi,w>;//更新dist[W]的值
				path[W] = vi;
			}
		}
	}
}
```

- 时间复杂度
  - 取决于两步：
    - 3——vi = set(V-S) 中dist[·]最小值者；
    - 9——dist[W] = dist[V]+Weight<v,w>;//更新dist[W]的值
  - 法1直接扫描所有未收录顶点
    - 寻找vi = set(V-S) 中dist[·]最小值者需要花费**【O(|V|】**
    - 更新dist[W]的值直接更新就好**【O(1)】**
    - 总体时间复杂度—–**【O(|V|^2^+|E|)】**
    - 对于稠密图效果更好
  - 法2：将整个`dist[]`数组存入最小堆中
    - 寻找vi = set(V-S) 中dist[·]最小值者需要花费**【O(log|V|)】**
    - 但是更新dist[W]的值就需要**【O(log|V|)】**的时间复杂度
    - 总体时间复杂度——**【O(|V|log|V|+|E|log|V|)】**
    - 对于稀疏图效果更好

#### 多源最短路算法

##### 法1：直接将单源最短路算法调用|V|遍

- **时间复杂度 【T = O(|V|^3^+|E|*|V|】**
- 对于稀疏图效果好

##### 法2：“Floyd算法”

- **时间复杂度【T = O(|V|^3^)】**
- 对于稠密图效果更好

```cpp
void Floyd(){
	for(int i=0; i<N; i++){
		for(int j=0; j<N; j++){
        	D[i][j] = G[i][j];//G[][]是邻接矩阵
        	path[i][j] = -1;
		}
	}
	for(int k=0; k<N; k++){
		for(int i=0; i<N; i++){
			for(int j=0; j<N; j++){
				if(D[i][k]+D[k][j] < D[i][j]){
					D[i][j] = D[i][k]+D[k][j];
					path[i][j] = k;
				}
			}
		}
	}
}
```

### 5.7 最小生成树问题

#### Prim算法

- 稠密图

#### Kruskal算法

- 稀疏图

### 5.8 拓扑排序
