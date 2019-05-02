[TOC]
#TensorFlow：Large-Scale Machine Learning on Heterogeneous Distributed System
## 0.Abstract
TensorFlow是一个可以表达众多机器学习算法的接口，也是一种执行这些算法的实现；
只需稍加改动就可以在众多异构系统中执行；


## 1.Introduction
Google Brain项目早期创建了第一代分布式可扩展训练和预测系统DistBelief；
在DistBelief的基础上，TensorFlow是第二代大规模机器学习模型的实现；
* TensorFlow使用stateful dataflow graph来表达计算；
* TensorFlow允许客户端轻松地表达各种并行，通过备份和各计算设备相互协作更新一系列共享参数或其他的状态来并行执行一个核心的dataflow graph模型；
* 宽松的同步，参数更新的灵活的一致性；
* 支持众多的异构硬件平台；


## 2.Programming Model and Basic Concepts
* 一个TensorFlow的计算过程被描述为由许多node组合而成的有向图，这个图代表了一个dataflow计算。允许node维持和更新持久化的状态，以及实现图内的分支和循环控制；
* 在一个TensorFlow的数据流图中，每个node有零个或多个输出，也有零个或多个输出，并代表某个操作(operation)的实例；
* 在数据流图中普通边上流动的值称为tensor，是任意维度的数组；
* 特殊的边被称为control dependencies，这些边上没有数据流动，只用来描述操作的依赖关系，用来表示后一个operation必须在前一个operation完成之后才能执行；

### Operations and Kernels
* 一个operation拥有一个名字，并代表一个计算(如矩阵乘法、加法)的抽象；
* 一个operation也有很多属性(如不同元素的多态计算，add of two tensors of type float/int32)，所有的属性必须在数据流图被构建的时候被提供或被推断，从而使得TensorFlow可以实例化一个node来执行这个operation；
* 一个kernel是对一种可以运行在专门硬件(如CPU、GPU)上的operation的专门的实现；
* TensorFlow使用一种注册机制来维护operation和kernel集合，用户可以通过链接新的operation或注册新的kernel来扩展这个集合；

### Session
* 用户通过创建一个Session来和TensorFlow交互，可以使用Session接口的Extend来对现有的数据流图进行拓展；
* Session支持的另一个主要操作是Run，它接受需要被计算的变量以及提供给数据流图的必要的数据来完成计算。调用Run后，TensorFlow会自动按照所定义的数据流图，并在遵守依赖关系的前提下完成计算；

### Variables
* 通常一个数据流图会被计算多次，大多数的tensor在数据流图的一次执行后就会消失。而一个Variable是一个特殊的operation，它可以返回一个指向在运算过程中持续存在的tensor的引用。这些引用可以被传递给其他操作来对tensor进行修改。在机器学习任务中，通常使用Variable来保存参数，并在数据流图的运算过程中不断更新；


## 3.Implementation
* 在一个TensorFlow系统中，client通过Session和TensorFlow的master进程交互，master进程将任务分配给不同的worker进程，而每个worker进程负责在一个或多个device(如CPU cores和GPU cards)上执行计算；
* TensorFlow有local和distributed的实现版本，local版本所有的运算都在同一台机器的(同一个操作系统)进程中执行(该机器可能有多个CPU或GPU)，distributed版本中client、master、workers可以在不同机器上的不同进程中；
  ![](http://ww1.sinaimg.cn/large/006aTs3ily1g2jtuqhpf8j31b40gyaew.jpg)

### Devices
* devices是TensorFlow的计算核心
* 每个worker负责一个或多个devices，每个device有一个device类型和名称。如/job:localhost/device:cpu:0或/job:worker/task:17/device:gpu:3，其中0和3是这个worker中该device的index；
* TensorFlow实现了CPU和GPU的device接口，用户也可以通过注册机制添加新的device实现；
* 每个device负责管理device内存的分配和释放，并且管理kernel的执行；

### Tensors
* Tensor的实现是一个强类型的多维数组；
* TensorFlow使用引用计数器来管理内存，当某个实例没有指向它的引用时就销毁这个实例；

### 3.1 Single-Device Execution
* TensorFlow最简单的执行模型是：一个Worker中的一个Device；
* 数据流图中的node按照定义的相互依赖关系执行，TensorFlow会保存每个node所依赖的且没有执行完毕的node的个数；
* 当一个node执行完毕后，所有依赖于它的nodes的count都减一；
* 当所有依赖的node执行完毕之后，该node会被加入一个就绪队列中，在这个就绪队列中的node的执行顺序是不确定的；

### 3.2 Multi-Device Execution
* 对于拥有多个运算device的系统，TensorFlow需要解决两个难题：
    * 决定在哪个device上执行某个node的计算任务
    * 管理由这个决定带来的device之间的数据交换通信
    
#### 3.2.1 Node Placement
* 对于一个给定的数据流图，TensorFlow会使用设备分配算法(placement algorithm)将计算任务映射到可用的设备上。设备分配算法将代价模型作为输入，代价模型包含了每个node中输入和输出张量的大小以及该node估计的计算时间；
* 设备分配算法模拟数据流图的计算过程并使用贪心策略(greddy heuristic)来为每个node分配运算设备；
* 设备分配算法首先从数据流图的源头开始对每个node的计算过程进行模拟，当某个node需要计算资源时，设备分配算法会将运行该计算的预计时间最短的可用设备分配给该节点。对于拥有多个可用计算设备的node，分配算法会使用贪心策略考虑将计算分配到不同设备后所需要的计算时间+设备间数据通信的成本。总之，分配算法会将执行某计算操作最快的可用设备分配个node；从上游一路往下游分配；

#### 3.2.2 Cross-Device Communication
* 当设备分配算法结束后，数据流图会被划分为多个子图，每个子图对应一个计算设备。
* 位于不同运算设备的任务x和y之间的通信被新建的Send node和Receive node所接管；
* 插入Send和Receive节点后，TensorFlow使依赖于特定张量的操作使用同一个Receive node，而不是每一个操作都拥有一个Receive node，这样可以是接收端设备之分配一次内存，并可以所需张量的数据仅被传输一次；
  ![](http://ww1.sinaimg.cn/large/006aTs3ily1g2jtrfn9t3j30pi0ciwgl.jpg)
* 这样处理设备通信的方法可以使得分配在不同设备上的node实现去中心化到各worker。因为Send和Receive解决了workers和devices之间的数据同步问题，所以master进程仅需对每个worker进程发出Run指令，而不需要管理位于不同运算设备上计算任务之间的通信；

### 3.3 Distributed Execution
* TensorFlow在分布式系统上的执行和在多个设备上的执行方式类似，在设备分配算法执行完成后，每个数据流图被分配到某个设备上，跨worker进程间的Send和Receive node通信使用远程通信机制(如TCP、RDMA)在不同机器间传输数据；

#### Fault Tolerance
* 在分布式系统运行过程中，错误可能在很多地方被检测到，TensorFlow主要依赖于
    * Send和Receive之间的通信错误
    * master进程对每个worker进程的周期性检查
* 当某个错误被监测到时，整个数据流图的计算将被中断并从头开始。注意，因为Variable是在计算过程中持续存在的张量，所以TensorFlow将每个Variable与一个Save节点连接，Save节点会定期checkpoint Variable的状态。当错误发生时，TensorFlow可以从Save保存的最近的状态恢复；


## 4.Extensions
### 4.1 Gradient Computation
