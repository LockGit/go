# Go  
```
一些小项目，算法，以及有学习价值的问题记录：
```
* 数据结构与算法（汇总，一些算法的golang实现，本项目algorithm文件夹下)
* 中英文翻译互转 (抓的有道chrome扩展数据包)
* 命令行下的dns信息工具 (golang简单实现类似dig的功能)
* 实现socks5代理来访问看上去不能访问的资源 （网络边界突破，实现类似lcx,ngrok的功能……）
* 两种cache淘汰策略 lru.go (nginx,redis,squid中都有lru的身影）
* 内存对齐在golang中的表现 (证明cpu读取内存是按块读取的)
* golang shell tools (golang实现的交互式shell，实现一个简易版的交互终端)
* golang实现一致性hash (从单节点走向分布式节点)
* golang实现类似nginx基于权重的upstream的轮询调度
* etcd watch机制的补充 (微服务)
* 微服务系统中的优雅升级
* 讨论rpc && 调用链追踪
* 敏感词的检测 (tire树,ac自动机)
* 布隆过滤器 && hash表 && Btree
* golang的map,interface,defer,return,goroutine,slice
* golang的两个nil可能不相等
* golang的gc机制
* diff算法（寻找diff的过程抽象成表示为图的搜索，类似git diff的差异比较，复杂!)

### Chinese and English translation tools in the command line（命令行下中英文翻译工具）
```
dict is a command line tool by go
```
![](https://github.com/LockGit/Go/blob/master/img/dict.gif)

## 一键安装(OneKey Install)
```
wget https://raw.githubusercontent.com/LockGit/Go/master/go-dict/dict.go && go build -o /usr/local/bin/dict dict.go 
```

## 使用(Usage)
### 英译中(English To Chinese)
```bash
➜  ~ dict I love you
您的输入: I love you
*******英汉翻译:*******
我爱你。
*******网络释义:*******
I Love You : 我爱你
I really love you : 真的爱你
I Do love you : 我是爱你的
```

### 中译英(Chinese To English)
```bash
➜  ~ dict 我爱你
您的输入: 我爱你
*******英汉翻译:*******
I love you
*******网络释义:*******
我爱你 : I Love You
我也爱你 : I Love You Too
我就爱你 : The Arrangement
```

### 命令行下的dns信息工具
#### 一键安装(OneKey Install)
```
wget https://raw.githubusercontent.com/LockGit/Go/master/fdns/FindDnsRecordFast.go && go build -o /usr/local/bin/fdns FindDnsRecordFast.go 
```

#### 使用(Usage)
![](https://github.com/LockGit/Go/blob/master/img/fdns.gif)

### 实现socks5代理来访问看上去不能访问的资源
```
假设有如下网络情况：
a----->b<------>c

a可以访问b
b不能访问a
b与c之间可以互相访问
实现c访问a上的服务

具体实现：
proxy_server.go 为服务端，proxy_client.go 为客户端
在b上运行：go run proxy_server.go 8000 7999 （其中8000,7999为自定义监听的端口）
在a上运行：go run proxy_client.go b的ip:8000  (其实是a主动连接b的8000端口，为后续的流转发铺垫）
然后c使用b ip的7999端口作为socks5代理即可访问a上的内容
更多场景，可自行探索
```

### 两种cache淘汰策略 lru.go
```
1,如果数据最近被访问过，那么将来被访问的几率也更高（访问后，把当前节点调整到链表头，新加入的Cache直接加到链表头中）
2,如果访问的频率越高，将来被访问的几率更高。（需要一个计数器，计数器排序后，调整链表位置，淘汰无用Cache）
go run lru.go
```

### 内存对齐在golang中的表现
```
思考：声明一个结构体的时候，结构体的内部成员顺序是否会对内存的占用产生影响？
首先了解一下:bool,int8,uint8,int32,uint32,int64,uint64,byte,string 类型占用多少字节。
fmt.Printf("bool size: %d\n", unsafe.Sizeof(bool(true)))
fmt.Printf("int8 size: %d\n", unsafe.Sizeof(int8(0)))
fmt.Printf("uint8 size: %d\n", unsafe.Sizeof(uint8(0)))
fmt.Printf("int32 size: %d\n", unsafe.Sizeof(int32(0)))
fmt.Printf("uint32 size: %d\n", unsafe.Sizeof(uint32(0)))
fmt.Printf("int64 size: %d\n", unsafe.Sizeof(int64(0)))
fmt.Printf("uint64 size: %d\n", unsafe.Sizeof(uint64(0)))
fmt.Printf("byte size: %d\n", unsafe.Sizeof(byte(0)))
fmt.Printf("string size: %d\n", unsafe.Sizeof("lock"))
得到如下：
bool size: 1
int8 size: 1
uint8 size: 1
int32 size: 4
uint32 size: 4
int64 size: 8
uint64 size: 8
byte size: 1
string size: 16

那么下面这个结构体理论上共占用 1+4+1+8+1=15 byte
type Test struct {
	a bool  // 1 byte
	b int32 // 4 byte
	c int8  // 1 byte
	d int64 // 8 byte
	e byte  // 1 byte
}

执行如下：
test := Test{}
fmt.Println(unsafe.Sizeof(test))
//output 32 
最终输出为占用32个字节,这与前面所预期的结果完全不一样,这充分地说明了先前的计算方式是错误的!

更改结构体成员顺序：
type Test2 struct {
	e byte  // 1 byte
	c int8  // 1 byte
	a bool  // 1 byte
	b int32 // 4 byte
	d int64 // 8 byte
}

执行：
test2 := Test2{}
fmt.Println(unsafe.Sizeof(test2))
//output 16 
最终输出为占用16个字节！两次输出结果不一样。

```
分析:
CPU读取内存是一块一块读取的，块的大小可以为 2、4、6、8、16 字节等大小。

针对第一个输出32byte的实验：

成员变量 | 类型 | 偏移量 | 自身占用
---|---|---|---
a | bool | 0 | 1
字节对齐 | 无 | 1 | 3
b | int32 | 4 | 4 
c | int8 | 8 | 1
字节对齐 | 无 | 9 | 7
d | int64 | 16 | 8
e | byte | 24 | 1
字节对齐 | 无 | 25 | 7
总占用大小 | - | - | 32

针对第二个输出16byte的实验：

成员变量 | 类型 | 偏移量 | 自身占用
---|---|---|---
e | byte  | 0 | 1
c | int8  | 1 | 1
a | bool  | 2 | 1
字节对齐 | 无 | 3 | 1
b | int32 | 4 | 4
d | int64 | 8 | 8
总占用大小 | - | - | 16

以上过程由于字节对齐原因，
也就是说CPU读取内存是一块一块读取的，而不是一次读取一个offset，所以造成了两次结果不一致。

### golang shell tools
+ [run a golang shell in the command](https://github.com/LockGit/Go/blob/master/shell.go)
```
使用golang实现的shell工具
```


### golang实现一致性hash (从单节点走向分布式节点)
+ ![](https://github.com/LockGit/Go/blob/master/img/hash.png)
```
解决数据倾斜问题，引入虚拟节点，hash环
虚拟节为一个真实节点对应多个虚拟节点。虚拟节点扩充了节点的数量，解决了节点较少的情况下数据容易倾斜的问题。  

另：
实际场景中有很多基于权重的选择节点算法，比如nginx的加权轮询，参看本项目下的round_node中的算法实现
1，每个节点用它们的当前值加上自己的权重。
2，选择当前值最大的节点为选中节点，并把它的当前值减去所有节点的权重总和。            
```

### etcd watch机制的补充 (微服务)
```
将etcd作为服务发现和配置中心已成为常态，我们可能经常需要在应用中起一个协程watch etcd的某个kv变化,用于尽可能实时的通知当前主进程。
但是在实际应用场景中我们发现经常在协程watch一段时间后，etcd中的value发生了变化，但是应用并没有watch到value的变化。
通过查看etcd源码的watcher interface大致可以得知:
1,正常情况下，起监听的协程如果cotenxt是context.Background/TODO，WatchChan管道不会关闭的，除非etcd集群出了问题不可用。
2,起监听的协程如果cotenxt使用了cetcd.WithRequireLeader包装的上下文,这个时候etcd的WatchChan管道会因为etcd集群leader的变更而导致WatchChan通道关闭，那么此时的watch机制自然失效。
理论上在接收到WatchChan通道关闭信号的时候，应用程序应该再次调用watch,也就是用for{}包装下，我们微服务系统又引入了go-micro的Config包，有二次修改搞了一些自定义的东西，粗略观察是没有这种for{}包装的
鉴于此，又由于网络或者磁盘IO等原因，我们自己应用中WatchChan管道的关闭都会导致watch失效,所以我们可以在微服务启动后，起一个goroutine定时轮询etcd,拿到最新的kv。
在WatchChan管道关闭watch失效时，将定时轮询得到的最新kv作为watch机制失效的补充。（定时不用太频繁，我设置的是1分钟，一些配置的1分钟误差可以接受，对于etcd集群来说也不会有太大的压力）

还有一种watch优化机制是在应用和etcd集群之间增加一个grpc代理，但是这个grpc代理会有cache，弄不好就会出问题，使用时也需小心。
```
+ ![](https://github.com/LockGit/Go/blob/master/img/etcd-watch.png)


### 敏感词的检测 (tire树,ac自动机)
```
这个在我的Py项目中也有介绍，这是风控系统的一层基础设施，简而言之,trie树利用字符串的公共前缀来降低查询时间的开销以达到提高效率的目的，带上AC自动机版的
会更强劲。在实际应应用中，我们将敏感词微服务化，用于粗略过滤一些简单的敏感信息，敏感词的词库在不断增加，应用中会起一个go协程增量更新新增的词。但更高级的
可能会基于机器学习来做，现实中，tire树版基本够用。
```

### 布隆过滤器 && hash表 && Btree
```
判断元素是否存在某个集合中：
hash表用于定位一个元素是否存在的复杂可以认为是O(1),主要取决去hash算法，如果出现hash冲突，一般会在对应hash值后面挂上一个链表，这里有一个问题，如果数据
足够多，那么hash表也会变的越来越大，hash冲突的可能性也会变大，一些hash算法本身可能会有rehash的过程来扩容hash表的大小。对于少量的数据，hash表是一个不错
的选择。

布隆过滤器的也可以用于判断一个元素是否存在与一个集合中，布隆过滤器与hash表有着本质数据结构区别,布隆过滤器会存在一定的误差，主要原因为hash冲突，所以选择合适的hash函数尤为重要。
网上有bloom filter的golang版实现，https://github.com/willf/bloom，这个库采用了murmurhash,murmurhash实现了尤其对大块的数据，具有较高的平衡性与低碰撞率，Redis在实现字典时也借鉴了这种方法。
布隆过滤器是一个很长的二进制向量和一系列随机映射函数，对于判断一个元素是否存在，布隆过滤器会先对key执行多次不同hash函数得到不同的hash值,在分别查找对应hash值的标志位。全部满足，则元素存在。由于hash冲突的原因，所以布隆过滤器说某个集合中不存在某个元素，那么一定不存在，说某个元素在,可能会被误判。
高版本的redis还支持bloom filter扩展，启动redis-server前加载对应so文件，则redis便很快可以提供一个布隆过滤器的功能，而相关细节均被屏蔽，对于使用者而言只需要设置下相关参数，误差率即可。

Btree，二叉树，这种树形结构应用也很广泛，比如mysql索引就采用了这种数据结构，而数据库本身有很多范围查找的场景，使用这种结构就尤其符合。与之相关的还有红黑树，多用于内存排序。
这些树或者树的变种均为解决数据平衡，提高查找效率而演变出来的结构。
```

### golang的map,interface,defer,return,goroutine,slice
```
1,结构体可以比较，但是包含map的结构体不可比较。
2,当程序运行到defer函数时，不会执行函数实现，但会将defer函数中的参数代码进行执行。
3,使用 i.(type) ,i必须是interface{}
4,map需要初始化,最好用make([]int,x)完成初始化
5,defer、return、返回值三者的执行逻辑应该是：return最先执行，return负责将结果写入返回值中；接着defer开始执行一些收尾工作；最后函数携带当前返回值退出。
6,真正实现并行至少需要2个操作系统硬件线程并至少有两个goroutine时才能实现真正的并行,每个goroutine在一个单独的系统线程上执行指令。
7,slice存在对array的引用，对slice的修改在slice未扩容前会影响到array，当slice发生扩容行为之后，这时候内部就会重新申请一块内存空间，
将原本的元素拷贝一份到新的内存空间上。此时其与原本的数组就没有任何关联关系了，再进行修改值也不会变动到原始数组。
```
```go
func caseOne() *int {
	var i int
	defer func(i *int) {
		time.Sleep(time.Second * 2)
		*i = 3
	}(&i)
	return &i
}

func caseTwo() int {
	var i int
	defer func(i int) {
		time.Sleep(time.Second * 2)
		i = 3
	}(i)
	return i
}
func main() {
	fmt.Println(*caseOne())
	fmt.Println(caseTwo())
}
```

```golang的两个nil可能不相等
接口(interface) 是对非接口值(例如指针，struct等)的封装，内部实现包含 2 个字段，类型 T 和 值 V。一个接口等于 nil，当且仅当 T 和 V 处于 unset 状态（T=nil，V is unset）。

两个接口值比较时，会先比较 T，再比较 V。
接口值与非接口值比较时，会先将非接口值尝试转换为接口值，再比较。
func main() {
	var p *int = nil
	var i interface{} = p
	fmt.Println(i == p) // true
	fmt.Println(p == nil) // true
	fmt.Println(i == nil) // false
}
上面这个例子中，将一个 nil 非接口值 p 赋值给接口 i，此时，i 的内部字段为(T=*int, V=nil)，i 与 p 作比较时，将 p 转换为接口后再比较，因此 i == p，p 与 nil 比较，直接比较值，所以 p == nil。

但是当 i 与 nil 比较时，会将 nil 转换为接口 (T=nil, V=nil)，与i (T=*int, V=nil) 不相等，因此 i != nil。因此 V 为 nil ，但 T 不为 nil 的接口不等于 nil。
```


```golang的gc机制
最常见的垃圾回收算法有标记清除(Mark-Sweep) 和引用计数(Reference Count)，Go 语言采用的是标记清除算法。并在此基础上使用了三色标记法和写屏障技术，提高了效率。
标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记（Mark）和清除（Sweep）两个阶段：

标记阶段 — 从根对象出发查找并标记堆中所有存活的对象；
清除阶段 — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表。
标记清除算法的一大问题是在标记期间，需要暂停程序（Stop the world，STW），标记结束之后，用户程序才可以继续执行。为了能够异步执行，减少 STW 的时间，Go 语言采用了三色标记法。

三色标记算法将程序中的对象分成白色、黑色和灰色三类。

白色：不确定对象。
灰色：存活对象，子对象待处理。
黑色：存活对象。
标记开始时，所有对象加入白色集合（这一步需 STW ）。首先将根对象标记为灰色，加入灰色集合，垃圾搜集器取出一个灰色对象，将其标记为黑色，并将其指向的对象标记为灰色，加入灰色集合。重复这个过程，直到灰色集合为空为止，标记阶段结束。那么白色对象即可需要清理的对象，而黑色对象均为根可达的对象，不能被清理。

三色标记法因为多了一个白色的状态来存放不确定对象，所以后续的标记阶段可以并发地执行。当然并发执行的代价是可能会造成一些遗漏，因为那些早先被标记为黑色的对象可能目前已经是不可达的了。所以三色标记法是一个 false negative（假阴性）的算法。

三色标记法并发执行仍存在一个问题，即在 GC 过程中，对象指针发生了改变。比如下面的例子：

1
A (黑) -> B (灰) -> C (白) -> D (白)
正常情况下，D 对象最终会被标记为黑色，不应被回收。但在标记和用户程序并发执行过程中，用户程序删除了 C 对 D 的引用，而 A 获得了 D 的引用。标记继续进行，D 就没有机会被标记为黑色了（A 已经处理过，这一轮不会再被处理）。

1
2
3
A (黑) -> B (灰) -> C (白) 
  ↓
 D (白)
为了解决这个问题，Go 使用了内存屏障技术，它是在用户程序读取对象、创建新对象以及更新对象指针时执行的一段代码，类似于一个钩子。垃圾收集器使用了写屏障（Write Barrier）技术，当对象新增或更新时，会将其着色为灰色。这样即使与用户程序并发执行，对象的引用发生改变时，垃圾收集器也能正确处理了。

一次完整的 GC 分为四个阶段：

1）标记准备(Mark Setup，需 STW)，打开写屏障(Write Barrier)
2）使用三色标记法标记（Marking, 并发）
3）标记结束(Mark Termination，需 STW)，关闭写屏障。
4）清理(Sweeping, 并发)
```

### leetcode with golang 
```
cat *.go |grep Time -A 1 |grep -v Time
 * 给定两个二进制字符串，返回它们的和（也是一个二进制字符串）。 例如， a = "11" b = "1" 返回"100"。
--
 * 将大于10的数的各个位上的数字相加，若结果还大于0的话，则继续相加，直到数字小于10为止
--
 * 将获得两个非空链接列表，表示两个非负整数。 数字以相反的顺序存储，并且它们的每个节点包含单个数字。 添加两个数字并将其作为链接列表返回。
--
 * 找出字符串中所有的变位词,重复出现的
--
 * 高度平衡二叉树判断
--
 * 中序遍历二叉树
--
 * 反转二叉树
--
 * 层序遍历二叉树
--
 * 在二叉树中找一条路径，使得该路径的和最大。该路径可以从二叉树任何结点开始，也可以到任何结点结束。
--
 * 返回所有根到叶节点的路径
--
 * 二叉树的后序遍历
--
 * 二叉树前序遍历
--
 * 二叉树锯齿形层次遍历
--
 * 等价于 求 m 与 n 二进制编码中 同为1的前缀. [5,7] , 5 & 6 & 7 = 4
--
 * 爬N阶的楼梯，每次可以爬1阶或者2阶，一共有多少种爬法? (递归，动态规划)
--
 * 回溯算法|递归实现
--
 * 给一个数组，其中数组在下标i处的值为A[i]，坐标(i,A[i])和坐标(i,0)构成一条垂直于坐标轴x的直线。现任取两条垂线和x轴组成四边形容器。问其中盛水量最大为多少？
--
 * 求小于给定非负数n的质数个数
--
 * git diff 算法
--
 * 给定字符串T和S，求S的子串中有多少与T相等。
--
 * 动态规划
--
 * 求一个整数的阶乘末尾有多少个0
--
* 从一个未经排序的数组中找出第k大的元素。注意是排序之后的第k大，而非第k个不重复的元素。
--
 * 在旋转排序数组中搜索(指定的目标)
--
 * 二分查找
--
 * 给定一个数组[a,b,c,d,e],一个target值，找出满足a+b+c+d==target的不重复集合
--
 * 无向图复制 DFS,BFS
--
 * 群组错位词分类
--
 * 实现一个http代理
--
 * 实现字符串位置查找
--
 *  同构字符串
--
 * 跳跃游戏---动态规划
--
 * 求一个字符串最后一个字符的长度
--
 * 字符串最长公共前缀
--
 * 从一个数组中找出出现半数以上的元素
--
 * 动态规划---最大面积
--
 * 求连续的数组值，加和最大
--
* 寻找连续数组中缺失的数
--
 * 将数组中的0移动数组末尾
--
 * 海明重量---位操作
--
 * 实现一个正则匹配算法
--
 * 一个有序数组，返回去除重复元素后数组的长度。不允许申请新的空间。
--
 * 给定一个数组和一个值，在这个数组中移除指定值和返回移除后新的数组长度。不要为其他数组分配额外空间,O(1)复杂度
--
 * golang反转链表
--
 * 反转数组
--
 * 一个list中只有一个数出现了1次，其他数2次，找出这个出现一次的数字
--
 * 在一个数组中，找到3个元数之和等于0的元素列表组合
--
 * 给一个整数数组，找到三个数的和与给定target的值距离最短的那个和
--
 * 两数之和
--
 * 是否是丑陋数，所谓丑陋数就是其质数因子只能是2,3,5。
```
![](https://github.com/LockGit/Go/blob/master/img/leetcode.gif)


### http-proxy.go 使用go实现一个http代理
```
执行：go run http-proxy.go -h 查看帮助

执行：go run http-proxy.go --debug=true  打开调式模式

默认监听：8080 端口，把浏览器的代理设置成127.0.0.1 8080 端口，那么浏览器所访问资源将会走go代理脚本
```
![](https://github.com/LockGit/Go/blob/master/img/http-proxy.png)
```
* 如果是https类型，CONNECT方法 （要求http协议>=1.1)
    * 客户端通过CONNECT方法请求代理服务器创建一条到达任意目的服务器和端口的TCP链接，代理服务器仅对客户端和服务器之间的后续数据进行盲转发（只是转发，不关心，也不懂发送的内容是什么）。
* 如果是http类型，直接代理转发

代理https时的大致过程如下：
1）客户端通过HTTP协议发送一条CONNECT方法的请求给代理服务器，告知代理服务器需要连接的主机和端口。
CONNECT www.xxx.com:443 HTTP/1.1
User-agent: Mozilla/5.0

2）代理服务器一旦建立了和目标主机（上例的www.xxx.com:443）TCP连接，就会回送一条HTTP 200 Connection Established应答给客户端。
example: HTTP/1.1 200 Connection Established

3）此时隧道就建立起来了。客户端通过该HTTP隧道发送的所有数据都会被代理服务器（通过之前建立起来的与目标主机的TCP连接)原封不动的转发给目标服务器。目标服务器发送的所有数据也会被代理服务器原封不动的转发给客户端。

4) 后续可完善认证体系，在vps上部署代理
```

### 数据结构与算法（汇总)
* 10个数据结构 && 10个算法
```
10个数据结构：
    数组、链表、栈、队列、散列表、二叉树、堆、跳表、图、Trie树
10个算法：
    递归、排序、二分查找、搜索、哈希算法、贪心算法、分治算法、回溯算法、动态规划、字符串匹配算法
```
![](https://github.com/LockGit/Go/blob/master/img/algo.jpeg)
* 算法复杂度
```
复杂度也叫渐进复杂度，包括时间复杂度和空间复杂度，用来分析算法执行效率与数据规模之间的增长关系，
可以粗略地表示，越高阶复杂度的算法，执行效率越低。常见的复杂度并不多，
从低阶到高阶有：O(1)、O(logn)、O(n)、O(nlogn)、O(n2 )

空间复杂度:
假设 arr=new int[n]; 这个空间复杂度=n
```
* 基本结构-数组
```
数组具有随机访问特效，连续内存空间，大多数编程语言的数组下标都是0开始，
计算数组内存中地址公式：
a[k]_address = base_address + k * type_size
假设从1开始表示第一个元素，那么这个公式变化为：a[k]_address = base_address + (k-1) * type_size 
其中k-1对于cpu多了一次减法运算，不过也不一定非要从0开始，Matlab就是非0开始，pyhton还支持负数下标，大部分语言沿用了c语言习惯
二维数组内存寻址：
对于 m * n 的数组，a [ i ][ j ] (i < m,j < n)的地址为：
address = base_address + ( i * n + j) * type_size
公式推算：
对于二维数组 x[][](长度为a1*a2)来说，求x[i][j]的时候（不会考虑i j越界的情况），
要到i的时候，一定走完了i*a2的长度，
在x[i][0]往后找j个长度就是x[i][j]，所以会从初始地址增加 （i*a2+j）个单位长度。
一维数组：（a1）x[i]_address = base_address + i * type_size
二维数组：（a1*a2）x[i][j]_address = base_address + ( i * a2 + j ) * type_size
三位数组：（a1*a2*a3）x[i][j][k]_address = base_address + ( i * a2*a3 + j * a3 + k ) * type_size
```
* 基本结构-链表
```
单链表，双向链表，循环链表，双向循环链表
比如gochat中维护房间内用户的数据结构就是双向链表
链表在内存中并不是连续存储，所以对CPU缓存不友好，没办法有效预读。
对于数组来说，存储空间是连续的，所以在加载某个下标的时候可以把以后的几个下标元素也加载到CPU缓存这样执行速度会快于存储空间不连续的链表存储。
本项目中内存对齐在golang中的表现也间接说明了，CPU读取内存是一块一块读取的。
1，空链表
2，一个节点
3，两个节点
4，头节点，尾部节点处理
```
* 基本数据结构-栈
```
先进后出，内存中的堆栈和数据结构堆栈不是一个概念，可以说内存中的堆栈是真实存在的物理区（说明:一般内存里会有虚拟地址的概念），数据结构中的堆栈是抽象的数据存储结构。
内存空间在逻辑上分为三部分：代码区、静态数据区和动态数据区，动态数据区又分为栈区和堆区。
代码区：存储方法体的二进制代码。高级调度（作业调度）、中级调度（内存调度）、低级调度（进程调度）控制代码区执行代码的切换。
静态数据区：存储全局变量、静态变量、常量，比如在java中常量包括final修饰的常量和String常量，系统自动分配和回收。
栈区：存储运行方法的形参、局部变量、返回值，由系统自动分配和回收。
堆区：new一个对象的引用或地址存储在栈区，指向该对象存储在堆区中的真实数据。
```
* 基本数据结构-队列
```
基于数组的顺序队列
基于链表的链式队列
基于数组的循环队列(基于数组的循环队列，利用CAS原子操作，可以实现非常高效的并发队列。这也是循环队列比链式队列应用更加广泛的原因)
阻塞队列和并发队列

CAS原子:
在入队前，获取tail位置，入队时比较tail是否发生变化，如果否，则允许入队，反之，本次入队失败。出队则是获取head位置，进行cas(乐观锁）
```
* 递归
```
什么场景用递归解决：
一个问题的解可以分解为几个子问题的解
这个问题与分解之后的子问题，除了数据规模不同，求解思路完全一样
存在递归终止条件（递归出口）

栈的调用越来越深，存在重复计算。（函数调用会使用栈来保存临时变量。每调用一个函数，都会将临时变量封装为栈帧压入内存栈，等函数执行完成返回时，才出栈）
尾递归优化，改成迭代也可以

是否所有的递归问题都可以改成迭代循环？
是的，因为递归本身就是借助栈来实现的，只不过我们使用的栈是系统或者虚拟机本身提供的，我们没有感知罢了。
如果我们自己在内存堆上实现栈，手动模拟入栈、出栈过程，这样任何递归代码都可以改写成看上去不是递归代码的样子。
但是这种思路实际上是将递归改为了“手动”递归，本质并没有变，徒增了实现的复杂度。
如果改成迭代，规则也不一定能轻易想出。
```
* 排序
```
类型：稳定排序，不稳定排序
原地排序：
冒泡排序（稳定）， O(n^2)
插入排序（稳定）， O(n^2)
选择排序（不稳定），O(n^2) 
快速排序（不稳定），O(nlogn)
希尔排序 (不稳定)，O(n^(1.3—2))（缩小增量排序，最后的数组也是使用插入排序完成整体有序）
---
非原地排序：
归并排序（稳定），O(nlogn)
计数排序（稳定），O(n+k)k是数据范围
桶排序（稳定），O(n)
基数排序（稳定），O(dn) d是纬度 

golang标准库中的Sort.Ints()用的是快排+希尔排序+插排，数据量大于12时用快排，小于等于12时用6作为gap做一次希尔排序，然后走一遍普通的插排（插排对有序度高的序列效率高）。
其中快排pivot的分区点选择做了很多基于首中尾值的很复杂的变种。
其中 b-a>12 && maxDepth==0时使用了堆排序后直接返回
for b-a > 12 { // Use ShellSort for slices <= 12 elements
		if maxDepth == 0 {
			heapSort(data, a, b)
			return
		}
   ......
   ......
}
```
* 查找
```
二分查找，O(logn) 对数时间复杂度
如果在42亿个数据中用二分查找一个数据，最多需要比较32次。要求是数据必须有序。
* 二分查找的变体
 * 假设查找的元素中存在重复元素
 * 1，查找第一个值等于给定值的元素
 * 2，查找最后一个值等于给定值的元素
 * 3，查找第一个大于等于给定值的元素
 * 4，查找最后一个小于等于给定值的元素
* 有序数组是一个循环有序数组,求“值等于给定值”的二分查找
```
* 跳表
```
跳表使用空间换时间的设计思路，通过构建多级索引来提高查询的效率，实现了基于链表的“二分查找”。
跳表是一种动态数据结构，支持快速地插入、删除、查找操作，时间复杂度都是 O(logn)。
应用场景：（Redis用跳表来实现有序集合，准确说应该是由一个双hashmap构成的字典和跳跃表实现）
虽然跳表的代码实现并不简单，但是作为一种动态数据结构，比起红黑树来说，实现要简单多了。
所以很多时候，为了代码的简单、易读，比起红黑树，更倾向用跳表，在范围查找上相比红黑树更容易。
```
* 散列表
```
支持快速地查询、插入、删除操作；
内存占用合理，不能浪费过多的内存空间；
性能稳定，极端情况下，散列表的性能也不会退化到无法接受的情况。

散列表的设计：
一个合适的散列函数；
定义装载因子阈值，并且设计动态扩容策略；
选择合适的散列冲突解决方法。（通常有链表法开放寻址法，链表法更常用）

在golang中，map就是一种类型底层是基于散列表实现,具体实现在runtime包中.
散列冲突也是使用链表法解决，不过这里的一个节点不是普通意义上的节点而是一个容量为8的桶，
单桶数量超过容量时再创建新的桶，这样的设计可以有效避免链表的缺点，如存储指针，不能利用CPU缓存等。
支持动态扩容，容量不够时会申请一个原来两倍大小的容量。

rehash时需要重新计算位置,有一些是渐进式reash，慢慢把老的散列表中数据迁移到新的散列表中。
```
* 二叉树
```
种类：
1.树、二叉树
2.二叉查找树
3.平衡二叉树、红黑树
4.递归树

二叉树的每个节点最多有两个子节点，分别是左子节点和右子节点。
二叉树中，有两种比较特殊的树，分别是满二叉树和完全二叉树。满二叉树又是完全二叉树的一种特殊情况。
每个节点都有左右两个子节点，这种二叉树就叫做满二叉树。
叶子节点都在最底下两层，最后一层的叶子节点都靠左排列，并且除了最后一层，其他层的节点个数都要达到最大，这种二叉树叫做完全二叉树。
完全二叉树把最底一层去掉，就是一颗满二叉树。
高度：从0开始
深度：从0开始
层数：从1开始
二叉树既可以用链式存储，也可以用数组顺序存储。数组顺序存储的方式比较适合完全二叉树，其他类型的二叉树用数组存储会比较浪费存储空间。
如果节点 X 存储在数组中下标为 i 的位置，下标为 2 * i 的位置存储的就是左子节点，下标为 2 * i + 1 的位置存储的就是右子节点。
反过来，下标为 i/2 的位置存储就是它的父节点。
前、中、后，遍历的时间复杂度都是 O(n)
前序：root->left-right （root第一个位置)
中序：left->root->right （root在中间位置）
后序：left->right->root （root最后位置）
层序：从上而下一层一层，依次放入一个stack中，出栈，打印，left入栈，right入栈

二叉查找树
在树中的任意一个节点，其左子树中的每个节点的值，都要小于这个节点的值，而右子树节点的值都大于这个节点的值。
二叉树删除操作，针对要删除节点的子节点个数的不同，需要分三种情况来处理。
第一种情况：如果要删除的节点没有子节点，我们只需要直接将父节点中，指向要删除节点的指针置为 null。
第二种情况：如果要删除的节点只有一个子节点（只有左子节点或者右子节点），我们只需要更新父节点中，指向要删除节点的指针，让它指向要删除节点的子节点就可以了。
第三种情况：如果要删除的节点有两个子节点，可以找到这个节点的右子树中的最小节点（也可以用左子树中的最大节点来顶替要删除的节点位置），把它替换到要删除的节点上。然后再删除掉这个最小节点，因为最小节点肯定没有左子节点（如果有左子结点，那就不是最小节点）
```
* 红黑树
```
红黑树特性：两端黑，叶为空，间隔红，黑相同
红黑树的基本思想是用标准的二叉查找树（完全由2-结点构成）和一些额外的信息（替换3-结点）来表示2-3树。红黑树就是用红链接表示3-结点的2-3树.
红黑树是对2-3查找树的改进，它能用一种统一的方式完成所有变换。其中黑节点代表了近似平衡高度，所有黑节点的定义都是为了保证这一点。

平衡二叉树的严格定义是：二叉树中任意一个节点的左右子树的高度相差不能大于1。
完全二叉树、满二叉树都是平衡二叉树，非完全二叉树也有可能是平衡二叉树。
红黑树只是做到了近似平衡，并不是严格的平衡，所以在维护平衡的成本上，要比AVL树要低。
它从根节点到各个叶子节点的最长路径，有可能会比最短路径大一倍。

红黑树需要满足几个要求：
1，根节点是黑色的；
2，每个叶子节点都是黑色的空节点（NIL），也就是说，叶子节点不存储数据；
3，任何相邻的节点都不能同时为红色，也就是说，红色节点是被黑色节点隔开的；
4，每个节点，从该节点到达其可达叶子节点的所有路径，都包含相同数目的黑色节点；

为什么需要这种树：
为了解决普通二叉查找树在数据更新的过程中，复杂度退化的问题而产生的。
红黑树的高度近似 log2n，所以它是近似平衡，插入、删除、查找操作的时间复杂度都是 O(logn)。

对比前面的几种数据结构
散列表：插入删除查找都是O(1), 是最常用的，但其缺点是不能顺序遍历以及扩容缩容的性能损耗。适用于那些不需要顺序遍历，数据更新不那么频繁的。
跳表：插入删除查找都是O(logn), 并且能顺序遍历。缺点是空间复杂度O(n)。适用于不那么在意内存空间的，其顺序遍历和区间查找非常方便。
红黑树：插入删除查找都是O(logn), 中序遍历即是顺序遍历，稳定。缺点是难以实现，去查找不方便。其实跳表更佳，但红黑树已经用于很多地方了。
```
* 堆
```：
堆是一个完全二叉树；（除了最后一层，其他层的节点个数都是满的，最后一层的节点都靠左排列）
堆中每一个节点的值都必须大于等于（或小于等于）其子树中每个节点的值。（最大值堆，最小值堆）
用数组来存储完全二叉树是非常节省存储空间的。因为不需要存储左右子节点的指针，单纯地通过数组的下标，就可以找到一个节点的左右子节点和父节点。
堆化：
下往上和从上往下，顺着节点所在的路径，向上或者向下，对比，然后交换。
一个包含n个节点的完全二叉树，树的高度不会超过log2​n。堆化的过程是顺着节点所在路径比较交换的，所以堆化的时间复杂度跟树的高度成正比，也就是O(logn)。
插入数据和删除堆顶元素的主要逻辑就是堆化，所以，往堆中插入一个元素和删除堆顶元素的时间复杂度都是O(logn)。
堆排序：
时间复杂度：O(nlogn)，原地排序，非稳定性排序
堆排序的建堆过程的时间复杂度是O(n)，CPU对于顺序访问的数据更加友好，可以做缓存，但是堆的元素访问是跳跃式，所以对CPU cache而言，有点缺陷。

用数组存储堆：
如果数组第1个元素不使用，那么：
数组中下标为i的节点的左子节点就是下标为i∗2 的节点，右子节点就是下标为i∗2+1的节点，父节点就是下标为 i/2​ 的节点。

如果从数组第0个元素开始存储，那么：
如果节点的下标是i，那左子节点的下标就是i*2+1，右子节点的下标就是i*2+2，父节点的下标就是(i−1)/2​。

也可以从下标为0开始，只是这样做的话，计算左子节点时，会多一次加法运算。

应用场景：
优先级队列，top N (实际场景，我更喜欢redis有序集合(跳表))，中位数 ……
```
* 图
```
图和树一样都是非线性表数据结构，和树不同的是图是一种更加复杂的非线性表结构，无向图、有向图、带权图、顶点、边、度、入度、出度。
顶点的入度，表示有多少条边指向这个顶点；顶点的出度，表示有多少条边是以这个顶点为起点指向其他顶点。
图的两个主要的存储方式：邻接矩阵和邻接表。

邻接矩阵存储方法的缺点是比较浪费空间，但是优点是查询效率高，而且方便矩阵运算。
邻接表存储方法中每个顶点都对应一个链表，存储与其相连接的其他顶点。尽管邻接表的存储方式比较节省存储空间，但链表不方便查找，所以查询效率没有邻接矩阵存储方式高。
针对这个问题，邻接表还有改进升级版，即将链表换成更加高效的动态数据结构，比如平衡二叉查找树、跳表、散列表等。

比如本项目中上面提到的git diff 算法，寻找diff的过程，抽象成表示为图的搜索。
```
* 字符串匹配
```
字符存储匹配算法是各种编程语言都会提供的字符匹配函数的底层依赖，它可以分为单模式匹配和多模式匹配算法。
单模式匹配：BF算法和RK算法，RK算法是BF算法的改进，它巧妙借助了哈希算法，提升了匹配的效率。
BF算法是Brute Force的缩写，中文译作暴力匹配算法，也叫朴素匹配算法。
RK算法的全称是Rabin-Karp算法，是两位发明人的名字拼接，BF算法的升级版。通过哈希算法对主串中的n-m+1个子串分别求哈希值，然后逐个于模式串的哈希值比较大小，如果相等就说明有对应的模式串。

BM算法：
BM（Boyer-Moore）算法，一种非常高效的字符串匹配算法，有实验统计，它的性能是著名的KMP算法的3到4倍。
前面的BF，RK算法已经能够满足我们需求了，为什么发明BM算法？
是为了减少时间复杂度，但是带来的弊端是，优化代码变得复杂，维护成本变高。文本编辑器中字符串查找很多使用了BM算法。
BM算法核心思想是，利用模式串本身的特点，在模式串中某个字符与主串不能匹配的时候，将模式串往后多滑动几位，以此来减少不必要的字符比较，提高匹配的效率。
BM 算法构建的规则有两类，坏字符规则和好后缀规则。好后缀规则可以独立于坏字符规则使用。因为坏字符规则的实现比较耗内存，为了节省内存，我们可以只用好后缀规则来实现BM算法。

KMP算法：
i：始终是主串的下标。
j：始终是模式串的下标。
关键点是，在匹配过程中的任何时候，主串下标为i的字符永远和模式串下标为j的字符对齐。
从初始状态开始，i一直在增加，而j保持为0（没有任何字符匹配上），那么它的含义就是模式串向后移动了i位。同时因为j=0，所以此时模式串首和主串i下标对齐。

KMP vs BM -> 前缀匹配和后缀匹配
BM属于贪心算法，适应于实际应用。KMP是稳定算法，不在乎特例。一般的BM算法很容易构造m*n的特例，不过实际应用中比较少见。当然BM算法也可以改造成稳定的，构造模式比KMP需要多一些时间。

trie树+ac自动机
字典树，多叉树，Trie树最有优势的是查找前缀匹配的字符串，这种数据结构现实项目中已经使用到，用作敏感词匹配，或者类似搜索词提示的功能。
Trie树的本质是利用字符串之间公共前缀，将重复的前缀合并在一起，加了ac自动机版的效率更高，fail指针类比前面的bm,kmp算法中多走了几步一样，提高了匹配效率，
此算法是空间换时间的一种。
构建fail指针，按层构建，root的fail指向本身，root的第一层节点fail指针都指向root，后面fail指针指向当前节点父节点的fail指针(有可能指向root)
```
* 贪心算法
```
霍夫曼编码：
包含1000个字符的文件，每个字符占1个byte（1byte=8bits），存储这1000个字符就一共需要8000bits。
使用霍夫曼编码，可实现压缩率在20%~90%之间。霍夫曼编码不仅会考察文本汇总有多少个不同字符，还会考察每个字符出现的频率，根据频率的不同，选择不同长度的编码。
核心思想：
把出现频率比较多的字符，用稍微短一些的编码；出现频率比较少的字符，用稍微长一些的编码。
避免解压缩过程中的歧义，霍夫曼编码要求各个字符的编码之间，不会出现某个编码是另一个编码前缀的情况。
假设这a,b,c,d,e,f这6个字符出现的频率从高到低依次是：
```

| 字符 | 频率 | 编码  | 总共二进制位数 |
| ---- | ---- | ----- | -------------- |
| a    | 450  | 1     | 450            |
| b    | 350  | 01    | 700            |
| c    | 90   | 001   | 270            |
| d    | 60   | 0001  | 240            |
| e    | 30   | 00001 | 150            |
| f    | 20   | 00000 | 100            |

```
把每个字符看作一个节点，并把频率放到优先级队列中。从队列中取出频率最小的两个节点A,B，然后新建一个节点C，把频率设置为两个节点的频率之和，
并把这个新节点C作为节点A,B的父节点，最后再把C节点放入到优先级队列中。重复这个过程，直到队列中没有数据。
给每一条边画上一个权值，指向左子节点的边，统统标记为0，指向右子节点的边，统统标记为1，从根节点到叶节点的路径就是叶节点对应字符的霍夫曼编码。
```
* 分治算法
```
将原问题划分成n个规模较小而结构与原问题相似的子问题，递归地解决这些子问题，然后再合并其结果，得到原问题的解。
1.快速排序
2.归并排序
3.桶排序
4.基数排序
5.二分查找
6.利用递归树求解算法复杂度的思想
7.分布式数据库利用分片技术做数据处理
8.MapReduce模型处理思想
```
* 回溯算法
```
深度优先搜索、正则表达式匹配、编译原理中的语法分析、数独、八皇后、0-1背包、图的着色、旅行商问题、全排列。
```
* 动态规划
```
大部分动态规划能解决的问题，都可以通过回溯算法来解决，只不过回溯算法解决起来效率比较低，时间复杂度是指数级的。
动态规划算法，在执行效率方面，要高很多。尽管执行效率提高了，但是动态规划的空间复杂度也提高了，动态规划是一种空间换时间的算法思想。
贪心：一条路走到黑，就一次机会，只能哪边看着顺眼走哪边
回溯：一条路走到黑，无数次重来的机会
动态规划：拥有上帝视角，手握无数平行宇宙的历史存档，同时发展出无数个未来

适合动态规划求解的问题的特性：一个模型，三个特征
一个模型：
1，多阶段决策最优解，寻找一组决策序列，经过这组决策序列，能够产生最终期望求解的最优值。

三个特征：
1，最优子结构（问题的最优解包含子问题的最优解）
2，无后效性（在推导后面阶段的状态的时候，我们只关心前面阶段的状态值，不关心这个状态是怎么一步步推导出来的，某阶段状态一旦确定，就不受之后阶段的决策影响）
3，重复子问题（不同的决策序列，到达某个相同的阶段时，可能会产生重复的状态）

解决动态规划问题：
一般能用动态规划解决的问题，都可以使用回溯算法的暴力搜索解决。可以先用简单的回溯算法解决，然后定义状态，每个状态表示一个节点，然后对应画出递归树。
从递归树中，很容易可以看出是否存在重复子问题，以及重复子问题是如何产生的。以此来寻找规律，看是否能用动态规划解决。
找到重复子问题之后，有两种处理思路，第一种是直接用回溯加“备忘录”的方法，来避免重复子问题。从执行效率上来将，这和动态规划的解决思路没有差别。

状态转移表法：
先画出一个状态表，状态表一般都是二维的其中每个状态包含三个变量，行，列，数组值。根据决策的先后过程，从前往后，根据递推关系，分阶段填充状态表中的每个状态。
最后，将这个递推填表的过程，翻译成代码，就是动态规划代码。

状态转移方程法：
状态转移方程有点类似递归的解题思路，需要分析，某个问题如何通过子问题来递归求解，也就是所谓的最优子结构。
根据最优子结构，写出递归公式，也就是所谓的状态转移方程。根据状态转移方程，进行代码实现。一般情况下，有两种代码实现的方法，一种是递归加“备忘录”，另一种是迭代递推。

状态转移方程是解决动态规划的关键。
```

### diff.go 实现类似git中diff功能
```
执行: go run diff.go 1.md 2.md

```
![](https://github.com/LockGit/Go/blob/master/img/go-diff.png)

```
关于算法：
Myers算法，原作者1953年生于美国Idaho州，University of California at Berkeley教授，
美国科学院工程部成员，贡献了模式匹配和计算生物学的基本算法。
算法将寻找diff的过程，抽象成表示为图的搜索。

在网上找了一些关于该算法文档描述如下：
以两个字符串，src=ABCABBA，dst=CBABAC为例，根据这两个字符串我们可以构造下面一张图，横轴是src内容，纵轴是dst内容。
那么，图中每一条从左上角到右下角的路径，都表示一个diff。向右表示“删除”，向下表示”新增“，对角线则表示“原内容保持不动“。
图：
```
![](https://github.com/LockGit/Go/blob/master/img/diff.png)

```
最优路径如红线：
```
![](https://github.com/LockGit/Go/blob/master/img/diff2.png)

```
首先，定义参数d和k，d代表路径的长度，k代表当前坐标x - y的值。定义一个”最优坐标“的概念，
最优坐标表示d和k值固定的情况下，x值最大的坐标。x大，表示向右走的多，表示优先删除。
还是用上面那张图为例。我们从坐标(0, 0)开始，此时，d=0，k=0，然后逐步增加d，计算每个k值下对应的最优坐标。
因为每一步要么向右（x + 1），要么向下（y + 1），要么就是对角线（x和y都+1)，所以，当d=1时，k只可能有两个取值，要么是1，要么是-1。
当d=1，k=1时，最优坐标是(1, 0)。
当d=1，k=-1时，最优坐标是(0, 1)。
因为d=1时，k要么是1，要么是-1，当d=2时，表示在d=1的基础上再走一步，k只有三个可能的取值，分别是-2，0，2。
当d=2，k=-2时，最优坐标是(2, 4)。
当d=2，k=0时，最优坐标是(2, 2)。
当d=2，k=2时，最优坐标是(3, 1)。
以此类推，直到我们找到一个d和k值，达到最终的目标坐标(7, 6)。

算法过程：
迭代d，d的取值范围为0到n+m，其中n和m分别代表源文本和目标文本的长度（这里我们选择以行为单位）
每个d内部，迭代k，k的取值范围为-d到d，以2为步长，也就是-d，-d + 2，-d + 2 + 2…
使用一个数组v，以k值为索引，存储最优坐标的x值（这里使用hash也行，但是用数组效率更高一些，因为Go不支持使用负数做索引，所以需要创建一个自定义类型）
将每个d对应的v数组存储起来，后面回溯的时候需要用
当我们找到一个d和k，到达目标坐标(n, m)时就跳出循环
使用上面存储的v数组（每个d对应一个这样的数组），从终点反向得出路径

git真正用的是标准Myers算法的一个变体，标准的算法空间消耗很大。
在某些情况下，变体产生的diff会和标准算法有所不同。
也就是说，如果你按照上面的算法实现的程序，出来的结果和git diff的结果有所不同是正常的。
```