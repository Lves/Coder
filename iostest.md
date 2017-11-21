# 源码分析笔记

[TOC]

### UITableView+FDTemplateLayoutCell

主要是一个缓存cell高度的控件，作者孙源博客讲解地址[http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)

技术点：

- 采用一个多维数组缓存cell的高度

- 缓存删除策略，当调用reloadview的时候清楚缓存

  实现方案是Hook住TableView的reloadData等主要方法，调用之前清楚原来的缓存


### ReactiveCocoa

ReactiveCocoa包含三部分

#### Result

Result是一个枚举类型，主要包含一下两种类型

```swift
case success(T)
case failure(Error)
```

里边还包含了analysis函数，参数接受两个Block

```swift
func analysis<U>(ifSuccess: (Value) -> U, ifFailure: (Error) -> U) -> U
```

#### ReactiveCocoa

#### ReactiveSwift

- Reactive.swift

  Reactive是一个带有泛型类型的结构体，结构体中base是泛型类型，ReactiveExtensionsProvider协议中包含这个结构体。

  ```swift
  public struct Reactive<Base> {
  	public let base: Base
  }
  ```

  ​

- Event.swift

  Event是一个枚举,表示Signal的状态

  ```swift
  public enum Event<Value, Error: Swift.Error> {
  	case value(Value)
  	case failed(Error)
  	case completed
  	case interrupted
  }
  ```

- Observer.swift

  Observer是一个类，里边有Action类型变量，Action是一个Block

  ```swift
  public typealias Action = (Event<Value, Error>) -> Void
  ```

  Observer有一系列send函数，函数作用就是执行action




### Alamofire

- 每一个Request都有一个delegate: TaskDelegate

- 每一个TaskDelegate都有一个串行队列queue，且队列是挂起状态的

  ```swift
  init(task: URLSessionTask?) {
          self.task = task

          self.queue = {
              let operationQueue = OperationQueue()

              operationQueue.maxConcurrentOperationCount = 1
              operationQueue.isSuspended = true //挂起状态
              operationQueue.qualityOfService = .utility

              return operationQueue
          }()
      }
  ```

- SessionManager负责创建和管理Request对象和他的`NSURLSession`。SessionManager有一个`open let delegate: SessionDelegate`SessionDelegate管理所有的网络回调

- 当调用`Alamofire.request(urlRequest)`时调用堆栈

  - Alamofire.request(urlRequest)
  - SessionManager.default.request(urlRequest)
  - DataRequest: request.resume()
  - URLSessionTask: task.resume()

- 当SessionDelegate（实现了URLSessionTaskDelegate方法）收到网络回调时会调用DataTaskDelegate（集成TaskDelegate）的代理

  ```swift
  delegate.urlSession(session, task: task, didCompleteWithError: error)
  ```

- DataTaskDelegate会把悬挂的队列启动queue.isSuspended = false

  ```swift
  @objc(URLSession:task:didCompleteWithError:)
      func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
          if let taskDidCompleteWithError = taskDidCompleteWithError {
              taskDidCompleteWithError(session, task, error)
          } else {
             .....

              queue.isSuspended = false
          }
      }
  ```

- 当调用responseJSON时就是向TaskDelegate的队列中添加处理返回值的任务。调用堆栈如下：

  ```swift
  delegate.queue.addOperation {
              let result = responseSerializer.serializeResponse(
                  self.request,
                  self.response,
                  self.delegate.data,
                  self.delegate.error
              )

              var dataResponse = DataResponse<T.SerializedObject>(
                  request: self.request,
                  response: self.response,
                  data: self.delegate.data,
                  result: result,
                  timeline: self.timeline
              )

              dataResponse.add(self.delegate.metrics)

              (queue ?? DispatchQueue.main).async { completionHandler(dataResponse) }
          }

  ```

  ​


### copy、mutableCopy

- 容器的copy和mutablecopy都是浅拷贝，只是拷贝结果是可变还是不可变区别
  - [array copy]   不可变数组                                        // 指针copy,容器指向同一个
  - [array mutableCopy]   可变数组                            // 指针copy,容器单层浅copy
  - [mutableArray mutableCopy]   可变数组             // 指针copy,容器单层浅copy
  - [mutableArray copy]   不可变数组                         // 指针copy,容器单层浅copy
- 字符串拷贝
  - NSString [str copy]字符串
    - copy不会创建新的对象
    - mutableCopy会创建新的对象
  - NSMutableString [str copy]字符串
    - copy、mutableCopy都会创建新的对象


### UITableview的优化方法

（缓存高度，异步绘制，减少层级，hide，避免离屏渲染）

### Delegate、Notification使用场景

#### 区别
    1. 代理可以获得接受者的返回值，通知只是简单的通知对方不会拿到接受后的结果
    2. 通知可以是一对多、多对多，多对一，只要注册了就可以获得相应的通知；代理则是一对n的，设置delege后需要实现相应的代理方法；
    3. 通知使用时需要借助第三方通知中心来实现消息传递，而代理则不需要；
#### 使用场景

- Delegate
  1. 回调方法
    日常开发中的网络请求和页面加载后的回调一般会使用delegate或者block.采用deleagte一般会有多个接受对象，而block一般是一对一的。
  2. UI相应或者自定义UI控件
    自定义控件需要使用者提供数据源或者相应通知使用者时会使用delegate，像TableView的delegate和datasource。
- 通知
  1. 一般在跨层之间或者两个毫不相关的模块间通信会使用通知，可以减少代码的耦合度。
  2. 如果多个地方相同的变化需要通知一个对象时，采用通知

### 分类和集成使用场景

#### 必须使用继承的情况

- 需要扩展的方法与原方法同名，但是还需要使用父类的方法。
- 需要扩展属性，一般选择集成。当然分类通过运行时也可以添加关联属性。

#### 分类使用情况

- 分类可以实现把方法分组的不同的单独文件中。

  - 减少单个文件的体积
  - 可以把不同的功能组织到不同的category里
  - 可以按需加载不同的category

- 针对系统提供的类扩展一些工具方法，而不改变原有类。

- 把framework的私有方法公开

- 模拟多继承

  利用OC的消息转发实现多继承，在分类中实现`methodSignatureForSelector`

  和`forwardInvocation`。 

  ​

  下面以UIViewController的多继承NSString的UTF8String函数为例，

  ```objective-c
  @implementation UIViewController (Mutable)
  - (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
      NSMethodSignature *sig = [super methodSignatureForSelector:aSelector];
      if (!sig) {
          sig = [self.title methodSignatureForSelector:aSelector];
      }
      return sig;
  }
  - (void)forwardInvocation:(NSInvocation *)anInvocation{
      SEL selector = [anInvocation selector];
      if ([self.title respondsToSelector:selector]) {
          [anInvocation invokeWithTarget:self.title];
      }
  }
  @end
  ```


 #### 分类的原理

- 分类也是一个结构体

  ```c
  struct objc_category {
      char * _Nonnull category_name                            
      char * _Nonnull class_name                               
      struct objc_method_list * _Nullable instance_methods     
      struct objc_method_list * _Nullable class_methods        
      struct objc_protocol_list * _Nullable protocols          
  }                                                            
  ```

  - 从category的定义也可以看出category的可为（可以添加实例方法，类方法，甚至可以实现协议）和不可为（原则上讲它只能添加方法，不能添加属性(成员变量)，不过可以通过运行时添加关联属性）
  - 分类中的可以写@property, 但不会生成`setter/getter`方法, 也不会生成实现以及私有的成员变量（编译时会报警告）; 

- 运行时做的事情

  - 把category的实例方法、协议以及属性添加到类上
  - 把category的类方法和协议添加到类的元类上
  - 如果多个分类中都有和原有类中同名的方法, 那么调用该方法的时候执行谁由编译器决定；编译器会执行最后一个参与编译的分类中的方法。
  - category的方法没有“完全替换掉”原来类已经有的方法，也就是说如果category和原来类都有methodA，那么category附加完成之后，类的方法列表里会有两个methodA
  - category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休，殊不知后面可能还有一样名字的方法。



### ARC下内存管理场景

- Block防止循环引用


- for循环中占用内存大，合理使用`@autoreleasepool{}`
- 合理使用NSTimer,如果Timer是ViewController的成员变量，需要在dealloc中调用invalidate和设置nil;
- 通知使用时记得页面销毁时注销通知
- 理解copy和mutableCopy
- 代理使用weak修饰
- block和字符串使用copy修饰符


### KVO底层实现原理

- KVO是基于runtime机制实现的
- 当某个类的属性对象第一次被观察时，系统就会在运行期动态地创建该类的一个派生类，在这个派生类中重写基类中任何被观察属性的setter 方法。派生类在被重写的setter方法内实现真正的通知机制
- 如果原类为Person，那么生成的派生类名为NSKVONotifying_Person
- 每个类对象中都有一个isa指针指向当前类，当一个类对象的第一次被观察，那么系统会偷偷将isa指针指向动态生成的派生类，从而在给被监控属性赋值时执行的是派生类的setter方法
- 键值观察通知依赖于NSObject 的两个方法: willChangeValueForKey: 和 didChangevlueForKey:；在一个被观察属性发生改变之前， willChangeValueForKey:一定会被调用，这就 会记录旧的值。而当改变发生后，didChangeValueForKey:会被调用，继而 observeValueForKey:ofObject:change:context: 也会被调用。

>  补充：KVO的这套实现机制中苹果还偷偷重写了class方法，让我们误认为还是使用的当前类，从而达到隐藏生成的派生类





### weak实现原理概括

Runtime维护了一个weak表，用于存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，Key是所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

文章：http://www.jianshu.com/p/13c4fb1cedea

https://www.desgard.com/weak/



### ARC下，不显式指定任何属性关键字时，默认的关键字都有哪些

1. 对应基本数据类型默认关键字是   atomic,readwrite,assign

  2. 对于普通的 Objective-C 对象   atomic,readwrite,strong



### 什么时候会报unrecognized selector的异常？

objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，如果，在最顶层的父类中依然找不到相应的方法时，程序在运行时会挂掉并抛出异常unrecognized selector sent to XXX 。但是在这之前，objc的运行时会给出三次拯救程序崩溃的机会：

1. Method resolution

objc运行时会调用`+resolveInstanceMethod:`或者 `+resolveClassMethod:`，让你有机会提供一个函数实现。如果你添加了函数，那运行时系统就会重新启动一次消息发送的过程，否则 ，运行时就会移到下一步，消息转发（Message Forwarding）。

2. Fast forwarding

如果目标对象实现了`-forwardingTargetForSelector:`，Runtime 这时就会调用这个方法，给你把这个消息转发给其他对象的机会。 只要这个方法返回的不是nil和self，整个消息发送的过程就会被重启，当然发送的对象会变成你返回的那个对象。否则，就会继续Normal Fowarding。 这里叫Fast，只是为了区别下一步的转发机制。因为这一步不会创建任何新的对象，但下一步转发会创建一个NSInvocation对象，所以相对更快点。 

3. Normal forwarding

这一步是Runtime最后一次给你挽救的机会。首先它会发送`-methodSignatureForSelector:`消息获得函数的参数和返回值类型。如果`-methodSignatureForSelector:`返回nil，Runtime则会发出`-doesNotRecognizeSelector:`消息，程序这时也就挂掉了。如果返回了一个函数签名，Runtime就会创建一个NSInvocation对象并发送`-forwardInvocation:`消息给目标对象。



### 典型的内存空间布局

从低地址到高地址依次为：代码区、只读常量区、全局区/数据区、BSS段、堆区、栈区。

- 代码区：存放可执行指令。

- 只读常量区：存放字面值常量、具有常属性且被初始化的全局和静态局部变量（如：字符串字面值、被const关键字修饰的全局变量和被const关键字修饰的静态局部变量）。

- 全局区/数据区：存放已初始化的全局变量和静态局部变量。

- BBS段：存放未初始化的全局变量和静态局部变量，并把它们的值初始化为0。

- 堆区：存放动态分配的内存。

- 栈区：自动变量和函数调用时需要保存的信息（逆向分析的重点）

  > 代码区和只读常量区一般统称为代码段
  >
  >  栈区和堆区之间相对生长的，堆区的分配一般按照地址从小到大进行，而栈区的分配一般按照地址从大到小进行分配。


   

### 事件传递响应链

http://www.cocoachina.com/ios/20160113/14896.html







继续研究的：

https://zhuanlan.zhihu.com/p/22834934

https://www.zhihu.com/question/19604641











面试github整理

https://github.com/ChenYilong





































## 网络协议

### TCP

尽管 T C P 和 U D P 都使用相同的网络层( I P )， T C P 却向应用层提供与 U D P 完全不同的服务。T C P 提供一种面向连接的、可靠的字节流服务。

面向连接意味着两个使用 T C P 的 应 用 ( 通 常 是 一 个 客 户 和 一 个 服 务 器 ) 在 彼 此 交 换 数 据之前必须先建立一个 T C P 连 接 。 这 一 过 程 与 打 电 话 很 相 似 ， 先 拨 号 振 铃 ， 等 待 对 方 摘 机 说“喂”，然后才说明是谁。

在一个 T C P 连接中，仅有两方进行彼此通信。在第 1 2 章介绍的广播和多播不能用于 T C P 。



T C P 通过下列方式来提供可靠性:

- 应用数据被分割成 T C P 认为最适合发送的数据块。这和 U D P 完 全 不 同 ， 应 用 程 序 产 生 的

  数据报长度将保持不变。由 T C P传递给 I P 的信息单位称为报文段或段( s e g m e n t )(参见

  图 1 - 7 )。在 1 8 . 4 节我们将看到 T C P 如何确定报文段的长度。

- 当 T C P发 出 一 个 段 后 ， 它 启 动 一 个 定 时 器 ， 等 待 目 的 端 确 认 收 到 这 个 报 文 段 。 如 果 不 能

  及时收到一个确认，将重发这个报文段。在第 2 1 章我们将了解 T C P 协 议 中 自 适 应 的 超 时

  及重传策略。

- 当 T C P 收到发自 T C P 连 接 另 一 端 的 数 据 ， 它 将 发 送 一 个 确 认 。 这 个 确 认 不 是 立 即 发 送 ，

  通常将推迟几分之一秒，这将在 1 9 . 3 节讨论。

- T C P 将保持它首部和数据的检验和。这是一个端到端的检验和，目的是检测数据在传输

  过程中的任何变化。如果收到段的检验和有差错， T C P 将 丢 弃 这 个 报 文 段 和 不 确 认 收 到

  此报文段(希望发端超时并重发)。

- 既然 T C P 报文段作为 I P 数据报来传输，而 I P 数据报的到达可能会失序，因此 T C P 报文段

  的到达也可能会失序。如果必要， T C P 将 对 收 到 的 数 据 进 行 重 新 排 序 ， 将 收 到 的 数 据 以

  正确的顺序交给应用层。

- 既然 I P 数据报会发生重复， T C P 的接收端必须丢弃重复的数据。

- T C P 还能提供流量控制。 T C P 连接的每一方都有固定大小的缓冲空间。 T C P 的 接 收 端 只允许另一端发送接收端缓冲区所能接纳的数据。这将防止较快主机致使较慢主机的缓冲

  区溢出。

#### TCP首部

![tcp_001](./img/tcp_001.png)




​			
​		
​	序号用来标识从 T C P 发端向 T C P 收 端 发 送 的 数 据 字 节 流 ， 它 表 示 在 这 个 报 文 段 中 的 的 第 一个数据字节。如果将字节流看作在两个应用程序间的单向流动，则 T C P 用 序 号 对 每 个 字 节 进行 计 数 。 序 号 是 3 2 b i t 的 无 符 号 数 ， 序 号 到 达 2 3 2 - 1 后又从 0 开始。

当建立一个新的连接时， S Y N标志变 1。序号字段包含由这个主机选择的该连接的初始序号 I S N ( I n i t i a l S e q u e n c e N u m b e r )。该主机要发送数据的第一个字节序号为这个 I S N加 1 ，因为S Y N 标 志 消 耗 了 一 个 序 号 ( 将 在 下 章 详 细 介 绍 如 何 建 立 和 终 止 连 接 ， 届 时 我 们 将 看 到 F I N 标志也要占用一个序号)。

既然每个传输的字节都被计数，确认序号包含发送确认的一端所期望收到的下一个序号。因 此 ， 确 认 序 号 应 当 是 上 次 已 成 功 收 到 数 据 字 节 序 号 加 1 。只有 A C K 标 志 ( 下 面 介 绍 ) 为 1 时确认序号字段才有效。

T C P为 应 用 层 提 供 全 双 工 服 务 。 这 意 味 数 据 能 在 两 个 方 向 上 独 立 地 进 行 传 输 。 因 此 ， 连
接的每一端必须保持每个方向上的传输数据序号。


​			
​		
​	





​

​




