# 一、**数据结构**

## **Channel**
```go
type hchan struct {  
   qcount   uint           // total data in the queue  
   dataqsiz uint           // size of the circular queue  
   buf      unsafe.Pointer // points to an array of dataqsiz elements  
   elemsize uint16  
   closed   uint32  
   elemtype *_type // element type  
   sendx    uint   // send index  
   recvx    uint   // receive index  
   recvq    waitq  // list of recv waiters  
   sendq    waitq  // list of send waiters  
   lock mutex  
}
```
- 没有缓冲区的管道：读写都会阻塞，直到有协程从管道写读数据
- 有缓冲区的管道：读时如果没有数据会阻塞，直到有协程写入；写时缓冲区已满也会阻塞，直到有协程读出数据
- 对于值为nil的管道，无论读写都会阻塞，而且是永久阻塞
- 内置函数close()可以关闭管道，关闭后向管道写入会出发Panic，但是可以读

#### **总结：**
- Channel实际上是个结构体hchan，里面有很多成员变量，比如元素类型、大小、接受队列、发送队列等等。
- 向channel发送数据的过程：先看recvq是否为空（不为空就直接唤醒recvq中的协程G，将数据写入G）。为空的话再看环形队列buf是否有空位（有空位就插入队尾），没空位就将当前的写成G插入sendq中。
- 读channel的过程：先看sendq是否为空（不为空的情况再分是否有缓冲区，有缓冲区就读取buf队首元素，然后将sendq的第一个协程G的数据插入到buf队尾并唤醒。没有缓冲区就从sendq取一个G，读取数据并唤醒），为空的话再看环形队列buf是否为空，即qcount>0?（不为空的话读取队首元素），为空的话讲当前协程加入到recvq中阻塞，等待被唤醒

## **Slice**
```go
type slice struct {  
   array unsafe.Pointer  
   len   int  
   cap   int  
}
```
- 切片和原数组或切片共享底层空间，修改切片会影响原来的数组或切片。
- Append可能会触发扩容：容量小于1024,2倍。大于等于1024，1.25倍
- 扩展表达式
	- Array := [10]int
	- S := Array[5:7:7]

## **Map**

```go
type hmap struct {  
   count     int // 当前保存的元素个数  
   flags     uint8  
   B         uint8  // 桶数组的大小  
   noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details  
   hash0     uint32 // hash seed  
   buckets    unsafe.Pointer // 长度为2^B  
   oldbuckets unsafe.Pointer // 旧桶，用于扩容  
   nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)  
   extra *mapextra // optional fields  
}
```

- 使用make初始化map时可以指定容量，减少内存分配次数。
- 读写冲突会触发panic
- 查找元素：
	- 根据key计算hash值，根据低8位确定桶位置，根据高8位从tophash中查询，没查询到就差溢出的bucket
- 添加元素：
	- 过程和查找相同，找不到就插入，找到了就更新
桶
```go
type bmap struct {  
   tophash [_bucketCnt_]uint8  //_bucketCnt__=8_  
   虚拟成员：data 存储key value

虚拟成员：overflow 指向下一个Bucket  
}
```
Map的扩容机制
**负载因子**：key数量/bucket数量
负载因子>6.5扩容或者overflow的数量（即溢出桶的数量）达到2的min(15,B)次方
**增量扩容**：
oldbucket指向原桶，buckets指向新的桶（数量为原来2倍），等oldbucket元素全部搬迁到buckets中后，释放oldbucket。
### **等量扩容**

极端情况下，大量增删后出现空桶过多，重新排列。也叫rehash

## **Struct**

类似java的class，但struct没有继承，只有组合。有隐式声明的字段，如

```go
Type Cat struct{
 Animal //等同于Animal Animal
}
```

## **Iota**

表示从声明开始计算的行数
## **String**

```go
type stringStruct struct {  
   str unsafe.Pointer //首地址  
   len int //长度  
}
```

### **为什么字符串不允许修改**

string只包含指针地址，这样传递很轻量

## **Sync.Map**

```go
type Map struct {  
   mu Mutex  
   read atomic.Value // readOnly 允许并发读  
   dirty map[any]*entry //负责新数据写入  
   misses int  
}
```

Var m sync.Map

M.Store load delete

Sync.Map的零值是空的map，不是nil

map[any]*entry

#### **查询数据**

先查询read表，查到返回，没查到根据amened判断是否要查dirty表，如果amened为true，查询dirty表，misses次数加一。miss次数达到dirty的数据个数时，dirty会覆盖read表

### **新增数据**

Read表中是否有数据，有的话取出用原子操作更新。没有的话，将read表数据复制到dirty,amened置为true，再向dirty新增数据。

（read和dirty的共同元素共享底层空间entry）

读多写少的环境，sync.Map性能优于Map

## 二、**控制结构**

### **select**

type scase struct {  
   c    *hchan         // chan  
   elem unsafe.Pointer // data element  
}

只能作用于管道

Case语句包含两个变量时，第二个变量表示是否读到了数据

### **for-range**

## 三、**协程**

## 四、**内存管理**

## 五、**并发控制**

### **Channel**

可以通过channel来控制多个协程，缺点是需要创建大量的channel，不优雅

### **WaitGroup**

Add 增加工作协程计数

Done 减少工作协程计数

Wait 增加等待协程计数

1. 工作协程计数变为负数时，会触发panic

2. Add数量大于Done时，会永久阻塞，Go会检测到死锁，触发panic

### **Context**

1. 空的context

type Context interface {  
   Deadline() (deadline time.Time, ok bool)  
   Done() <-chan struct{}  
   Err() error  
   Value(key any) any  
}

2. cancelCtx

type cancelCtx struct {  
   Context  
   mu       sync.Mutex            // protects following fields  
   done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call  
   children map[canceler]struct{} // set to nil by the first cancel call  
   err      error                 // set to non-nil by the first cancel call  
}

Cancel（）关闭自己及后代

3. timerCtx

DeadlineContext

和cancel一样，区别在于多了个定时器，到时间自动执行cancel()

TimeoutContext

4. valueCtx

可以通过子context查询到父节点的value值

### **Mutex**

type Mutex struct {  
   state int32 //互斥锁的状态，表示是否被锁定  
   sema  uint32 //信号量  
}

### **RWMutex**

type RWMutex struct {  
   w           Mutex  // held if there are pending writers  
   writerSem   uint32 // 写阻塞等待的信号量，最后一个读会释放信号  
   readerSem   uint32 // 读阻塞等待的信号量，写锁的协程执行完会释放  
   readerCount int32  //记录读的个数  
   readerWait  int32  // 记录读阻塞时读的个数  
}

**写操作时如何阻塞写操作的？**

互斥锁

**写操作如何阻塞读操作**

持有写锁时，会将readerCount减去2^30,后续读进程发现小于0，变等待

**读操作如何阻塞写操作**

Readercount不为0时，阻塞写操作

**为什么写锁定不会被饿死**

写操作到来时，会把readerCount复制到readerWait中，读操作结束后，两个变量都会递减，readerWait为0时，唤醒写操作

## 六、**反射**

Interface保存变量值和变量类型

type iface struct {  
   tab  *itab //保存变量类型以及方法集  
   data unsafe.Pointer //变量值位于堆栈的指针  
}

reflect.ValueOf() //获取值  
reflect.TypeOf() //获取类型

反射三定律：

1. 反射可以将interface类型变量转换为反射对象

2. 反射可以将反射对象还原成interface类型变量

v := reflect.ValueOf(a)  
b := v.Interface()

3. 反射对象可以修改，value值必须是可设置的

a := 3.4  
v := reflect.ValueOf(&a)  
v.Elem().SetFloat(5.65)  
fmt.Print(v.Elem().Interface())

## 七、**测试**

## 八、**异常处理**

### **Error**

Go1.13之前有两种创建error的方法

errors.New()  
fmt.Errorf()

早期版本fmt.Errorf底层就是调用errors.New，只不过多了拼接字符串的功能。

这个阶段判断err的值和类型方式如下

Err == os.ErrPermission

E, ok := err.(*os.PathError)

这个版本有个问题，error一层一层的往上抛并附加信息，最后信息都掺杂在了一起，不好分辨。

Go1.13推出了链式error  warpError

type wrapError struct {  
   msg string  
   err error  
}  
  
func (e *wrapError) Error() string {  
   return e.msg  
}  
  
func (e *wrapError) Unwrap() error {  
   return e.err  
}

fmt.Errorf方法增加了两个动词，%v生成errorString(普通error)，%w生成wrapError

这样生成的warpError可以添加本层的msg信息

Unwrap获取上一层的Error

errors.Is逐层判断err是否是某个值

Errors.As逐层判断err是否属于某种类型

### **defer**

defer只能作用于函数或者函数调用

三条规则：

1. **延迟函数的参数在defer出现时就已经确定了**

参数是地址值的情况，地址值不会变，但是地址对应的变量值可能会变化

2. **后进先出，先出现的defer后执行**

3. **延迟函数可能操作主函数的具名返回值**

注意，必须是具名！return不是一个原子操作，return i实际是分成两步，先将i存入栈中作为返回值，再执行跳转。defer就可以在跳转前修改i的值。具体看P241。

总结：func test() (result int)

只有这种情况，且defer中操作result（其他变量i 字面值等不会改变）才会影响返回值

实现原理

编译器优先使用**open-code，其次stack-allocater，最后heap-allocater，**因为存堆上会频繁获取释放内存，影响效率。

**Go1.13之前，heap-allocater 存储在堆上的defer**

Defer结构体包含了栈地址、程序计数器、函数地址等。编译器会把每个defer实例暂存到goroutine数据结构中，等函数快结束时才调用。

Deferproc()将defer函数存入到goroutine的_defer链表中

Deferreturn()将Defer从链表中取出

1. 编译成deferproc()

2. 运行时执行deferproc()，生成_defer实例

3. 插入goroutine的_defer头部

4. 在return前插入deferreturn函数

**Go1.13 stack-allocater 存储在栈上的defer**

区别在于deferprocStack()取代了deferproc()，编译器直接在栈上预留_defer空间

**Go 1.14 open-code 开放编码的defer**

直接推翻上述机制，编译时完成预处理，直接将defer语句翻译成相应的执行代码

但是以下场景会失效：

1. 禁用了编译器优化 -gcflags=”-N -l”

2. defer出现在了循环语句中

3. 单个函数defer出现8次以上，或者return和defer个数乘积超过15

### **Panic**

如果产生了panic，会立即跳转到defer，如果没被recover, 向上传递，直到程序崩溃

Defer中无论嵌套了多少个panic都没关系，只不过在协程链表中增加一个panic实例。

底层是gopanic函数

### **Recover**

只能用于defer后面

返回值就是Panic的参数

1. Recover只能处理本协程的panic

2. 处理完成后无法回到产生panic的位置继续执行

原理

Gorecover()函数，给panic实例标记recovered状态（p.recovered=true）

执行recover的defer函数是被runtime.gopanic()执行的，defer结束以后，gopanic函数会检查panic实例的recovered状态，发现被恢复了，则gopanic将结束当前panic流程，将程序流程恢复正常。

Recover()函数没有参数，runtime.gorecover(argp uintptr)却有参数，该参数实际上就是defer函数的地址，_panic实例中也保存了defer的参数地址，如果相等，则说明recover()被defer函数直接调用。

## 九、**定时器**

### **一次性定时器Timer**

启动timer并不是另外启动一个协程

**timer.NewTimer(d)**创建Timer相当于把一个计时任务交给系统守护协程，该协程管理所有的Timer。到时间后，向Timer的管道发送当前时间作为事件。

**timer.Stop()**停止计时器（Stop）相当于通知系统守护协程移除该定时器。

**timer.Reset(d)**重置定时器（Reset）相当于先停止计时器，再启动

type Timer struct {  
   C <-chan Time  
   r runtimeTimer  
}

C是管道，上层根据此管道接受事件

r定时器，面向底层

func NewTimer(d Duration) *Timer {  
   c := make(chan Time, 1)  
   t := &Timer{  
      C: c,  
      r: runtimeTimer{  
         when: when(d),  
         f:    sendTime,  
         arg:  c,  
      },  
   }  
   startTimer(&t.r)  
   return t  
}

NewTimer只是创建了一个runtimeTimer实例，然后交给startTimer处理

c有一个缓冲区，所以向c写入数据不会阻塞

func sendTime(c any, seq uintptr) {  
   select {  
   case c.(chan Time) <- Now():  
   default:  
   }  
}

有个default是因为留给后续的Ticker使用,如果管道有数据没被取走，那么会直接退出，此次事件会被丢弃

### **周期性定时器Ticker**

type Ticker struct {  
   C <-chan Time // The channel on which the ticks are delivered.  
   r runtimeTimer  
}

和Timer几乎一样

for range ticker.C

ticker使用完后要释放，否则会资源泄露

### **runtimeTimer**

Go1.10之前，所有的runtimeTimer都保存在一个全局的堆中

Go1.10-1.13 runtimeTimer被拆分到多个全局的堆中，一般有64个桶来存储

Go1.14+ runtimeTimer保存在每个处理器P中，消除了专门的系统协程

## 十、**语法糖**

## 十一、**泛型**

泛型函数就是把函数的参数和返回值**泛化**

### 1. **函数泛化**

func SumIntsOrFloats[K comparable, V int64 | float64](m map[K]V) V {

2. **类型泛化**

type Vector[T any] []T  
func main() {  
   var arr Vector[int]  
}

**类型集合**

## 十二、**依赖管理**

## 十三、**编程陷阱**