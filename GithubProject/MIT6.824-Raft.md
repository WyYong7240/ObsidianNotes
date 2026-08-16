---
tags:
  - Go
  - 分布式系统
  - MapReduce
  - MIT6_824
  - Raft
---

# Lab1：MapReduce

## MapReduce

> 在开始Lab1的实验之前，先介绍一下MapReduce，方便后面编写实验代码，让思路更加明了。

MapReduce是一种编程模型和相关的实现，用于处理和生成大型数据集。用户指定一个处理键/值对以生成一组中间键/值对的map函数，以及一个合并与同一中间键关联的所有中间值的reduce函数。

这种函数式风格编写的程序被自动并行化，并在大量的商用机器集群上执行。运行时系统负责对输入数据进行分区、在一组机器上调度程序的执行、处理机器故障以及管理所需的机器间通信等细节。这使得没有任何并行和分布式系统经验的程序员可以轻松地利用大型分布式系统的资源。

函数主要分为Coordinator协调器、Worker工作器、Map函数、Reduce函数。

其中，Map函数和Reduce函数都是Worker工作器的子功能，Coordinator作为协调器，负责接收需要处理的数据或文件，并创建nMap个Map任务。任务创建完成后，会有任意个数个Worker（可能分布在不同的节点上）向Coordinator请求任务（Worker可以接收Map任务和Reduce任务），而Coordinator作为任务的协调者，就要处理这些并行请求，同步或者说合理的分配任务给每个发送请求的Worker。Worker接收到任务后，首先判定任务类型，根据任务类型进行不同的处理。Worker在完成任务后，要通知Coordinator已经完成了任务，然后Coordinator做出相应改变，如果所有的Map任务都完成，coordinator就要判定进入Reduce阶段，并根据Map产生的中间文件创建nReduce个Reduce任务，然后Worker会继续向Coordinator请求任务，这时Coordinator分配的就是Reduce任务。当所有Reduce任务被完成后，Coordinator进入结束阶段，如果再有Worker请求任务，Coordinator应该告诉Worker让Worker退出。

### 编程模型与类型

* Map函数：`map(k1,v1) -> list(k2,v2)`

  Map函数由用户编写，其接收一个键值对，并产生一组中间键值对，MapReduce库将所有中间键通过哈希函数，分为nReduce个桶，并最终将每个桶传递给Reduce函数

* Reduce函数:`reduce(k2, list(v2)) -> list(v2)`

  Reduce函数接收一个中间键和该键的一组值，其将这些值合并在一起，形成一个可能更小的值集。

### 流程图与示例模型
<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202512190942159.png" alt="image.png" style="zoom:80%;" />

* 模型流程
  1. 现假设有x个File需要通过MapReduce处理，Master即Coordinator接收这x个文件，并按照预设大小，将这x个文件划分为nMap个Map任务，然后启动监听，给Worker分配任务。
  2. 现假设有3个Worker并行启动，同时向Coordinator请求任务，分别分配到1个Map任务，并开始处理。当Worker处理完Map任务后，需要向Coordinator通知该Map任务已经完成。需要注意的是，根据论文，每个Map任务都会产生nReduce个中间文件。
  3. Coordinator继续分配任务，在分配任务的同时，还要留意已经分配出去的任务是否已经超时，如果已经分配出去的任务Worker没有按时完成，应该让该任务被重新分配，直到所有Map任务完成。
  4. 当Coordinator发现所有的Map任务被完成，就判定进入Reduce阶段，根据Map操作产生的nReduce个中间文件，创建nReduce个任务，并继续给Worker分配Reduce任务。
  5. Worker继续向Coordinator请求任务，并得到Reduce任务，开始处理Reduce任务。Worker处理Reduce任务时，会读取每个Map操作产生的对应reduceID的共nMap个中间文件，每个Reduce任务都会产生一个结果文件。Worker完成Reduce任务后，也要通知Coordinator完成了该Reduce任务。
  6. Coordinator继续分配任务，在分配任务的同时，还要留意已经分配出去的任务是否已经超时，如果已经分配出去的任务Worker没有按时完成，应该让该任务被重新分配，直到所有Reduce任务完成。
  7. Coordinator发现所有的Reduce任务被完成后，进入结束阶段，进入结束阶段还不能立即退出，立即退出会导致还没有退出的Worker还在向Coordinator发送请求，然后发现拒绝连接最后退出。进入结束阶段，如果还有Worker向Coordinator请求任务，Coordinator应该向Worker通知所有任务完成，结束进程。

## 实验内容

> 你的任务是实现一个分布式 MapReduce 系统，它由协调器和工作进程两个程序组成。系统中只有一个协调器进程，以及一个或多个并行执行的工作进程。在实际系统中，工作进程通常会运行在多台不同的机器上，但为了方便实验，我们将所有工作进程运行在一台机器上。工作进程通过远程过程调用 (RPC) 与协调器通信。每个工作进程会在一个循环中，向协调器请求一个任务，从一个或多个文件中读取任务的输入，执行任务，将任务的输出写入一个或多个文件，然后再次向协调器请求新任务。协调器应该能够检测到某个工作进程是否在合理的时间范围内（本实验中，使用十秒）没有完成任务，并将相同的任务分配给另一个工作进程。

具体见链接https://pdos.csail.mit.edu/6.824/labs/lab-mr.html

建议看完该链接的所有内容后，再开始实验，链接下面有相应的提示和实现要求。

## 我的实现

### src/mr/coordinator.go

~~~go
package mr

import (
	"log"
	"net"
	"net/http"
	"net/rpc"
	"os"
	"sync"
	"sync/atomic"
	"time"
)

const TimeOut = 10

type Coordinator struct {
	// 任务列表
	MapTaskChan    chan MapTask    // Map任务通道，类似线程安全的队列
	ReduceTaskChan chan ReduceTask // Reduce任务通道，类似线程安全的队列

	// 分配任务列表
	DistributedTask sync.Map // 已经被分配的Map任务，用于coordinator检查任务是否超时
	// DistributedReduceTask sync.Map

	Phase        int64 // 阶段0对应map，1对应reduce，2对应完成任务
	FinishedTask int64 // 每个阶段已经完成的任务数量
	NMap         int   // 共有N个Map任务，共产生NMap * NReduce个中间文件
	NReduce      int
}

type MapTask struct {
	FileName  string // Map处理的文件名
	MapID     int    // 该文件对应的Map任务ID，用于标识输出的中间文件
	StartTime int64  // 任务开始时间，也即任务被分配时间
}

type ReduceTask struct {
	ReduceID  int   // 该Reduce任务处理的中间文件ID，每个Map操作都有对应ID的中间文件
	StartTime int64 // 任务开始时间，也即任务被分配时间
}

// Your code here -- RPC handlers for the worker to call.

// an example RPC handler.
//
// the RPC argument and reply types are defined in rpc.go.

// 当有多个Worker Call Coordinator.AssignTask RPC时，会同时有多个AssignTask运行，因此需要线程安全访问
func (c *Coordinator) AssignTask(args *WorkerRequest, reply *WorkerReply) error {
	mrCoordinatorPhase := atomic.LoadInt64(&c.Phase)

	// 如果是协调器处于Map阶段，给Worker分配Map任务
	if mrCoordinatorPhase == 0 {
		// 首先检查是否存在超时的任务
		c.DistributedTask.Range(func(key, value interface{}) bool {
			// 先做类型转换
			taskFileName, _ := key.(string)
			mapTask, _ := value.(MapTask)
			nowTime := time.Now().Unix()
			// 如果该任务超时了，就重新进入chan，并将其在被分配map中删除
			if nowTime-mapTask.StartTime > TimeOut {
				c.MapTaskChan <- mapTask
				c.DistributedTask.Delete(taskFileName)
			}
			return true
		})

		// 从通道中取出任务，如果任务为空，就阻塞
		// 说是不能阻塞，想象，当最后一个Map任务被分配出去，MapTaskChan就是空的，并且状态Map也是空的
		// 但是其他Worker请求任务，由于Coordinator还在Map阶段，就会在这里阻塞住，那么当最后一个Map任务完成，其他阻塞在这里的Worker应该如何？
		// 因此，不能在这里阻塞住，如果没有任务分配，就让Worker进入等待状态
		if len(c.MapTaskChan) == 0 {
			reply.TaskType = "wait"
		} else {
			takeAMapTask := <-c.MapTaskChan
			nowTime := time.Now().Unix()

			// 给Worker Map工作的信息
			reply.TaskType = "map"
			reply.File = takeAMapTask.FileName
			reply.NReduce = c.NReduce
			reply.DistributedTime = nowTime
			reply.TaskID = takeAMapTask.MapID

			// 将该任务被分配出去，记录在Map中
			takeAMapTask.StartTime = nowTime
			c.DistributedTask.Store(takeAMapTask.FileName, takeAMapTask)

			// fmt.Printf("Assgin %s Task, File %s, TaskID %d\n", reply.TaskType, reply.File, reply.TaskID)
		}
	} else if mrCoordinatorPhase == 1 {
		// 首先检查是否存在超时的任务
		c.DistributedTask.Range(func(key, value interface{}) bool {
			// 先做类型转换
			reduceTaskID, _ := key.(int)
			reduceTask, _ := value.(ReduceTask)
			nowTime := time.Now().Unix()
			// 如果该任务超时了，就重新进入chan，并将其在被分配map中删除
			if nowTime-reduceTask.StartTime > TimeOut {
				c.ReduceTaskChan <- reduceTask
				c.DistributedTask.Delete(reduceTaskID)
			}
			return true
		})

		// 从通道中取出任务，如果任务为空，就阻塞
		// 说是不能阻塞，想象，当最后一个Map任务被分配出去，MapTaskChan就是空的，并且状态Map也是空的
		// 但是其他Worker请求任务，由于Coordinator还在Map阶段，就会在这里阻塞住，那么当最后一个Map任务完成，其他阻塞在这里的Worker应该如何？
		// 因此，不能在这里阻塞住，如果没有任务分配，就让Worker进入等待状态
		if len(c.ReduceTaskChan) == 0 {
			reply.TaskType = "wait"
		} else {
			takeAReduceTask := <-c.ReduceTaskChan
			nowTime := time.Now().Unix()

			// 给Worker Reduce工作的信息
			reply.TaskType = "reduce"
			reply.TaskID = takeAReduceTask.ReduceID
			reply.NMap = c.NMap
			reply.DistributedTime = nowTime

			// 将该任务被分配出去，记录在Map中
			takeAReduceTask.StartTime = nowTime
			c.DistributedTask.Store(takeAReduceTask.ReduceID, takeAReduceTask)

			// fmt.Printf("Assgin %s Task, File %s, TaskID %d\n", reply.TaskType, reply.File, reply.TaskID)
		}
	} else if mrCoordinatorPhase == 2 {
		// 通知worker退出
		reply.TaskType = "done"
	}
	return nil
}

func (c *Coordinator) TaskFin(args *WorkerRequest, reply *WorkerReply) error {
	mrCoordinatorPhase := atomic.LoadInt64(&c.Phase)

	timeNow := time.Now().Unix()
	if mrCoordinatorPhase == 0 {
		// Worker向协调器报告工作状态，首先看这个任务是否Timeout

		value, ok := c.DistributedTask.Load(args.FileName)
		if !ok {
			// 如果没有从被分配Map中获取到对应任务，说明Coordinator已经发现该任务超时，并且将该任务重新加入chan了
			return nil
		}
		mapTask, _ := value.(MapTask)
		taskStartTime := mapTask.StartTime

		// 就算在已分配任务列表中找到了对应元素，也不一定记录的是分配该worker了，因此仍然需要比对当前任务的开始时间和该worker收到的被分配时间，如果不一致就不是分配给该worker的
		// 如果worker处理的任务的分配时间不等于该任务的起始时间，说明该任务被重新分配了，如果相等，还超时了那就按超时算
		if taskStartTime != args.DistributedTime || timeNow-taskStartTime > TimeOut {
			// fmt.Printf("Map Task TimeOut, TaskID: %d, FileName: %s\n", args.TaskID, args.File)
			return nil
		}

		// 如果任务没有超时，说明任务正常完成，在协调器中将该任务从已分配任务列表中删除，并将任务完成计数+1
		c.DistributedTask.Delete(args.FileName)
		finishedTask := atomic.LoadInt64(&c.FinishedTask)
		atomic.StoreInt64(&c.FinishedTask, finishedTask+1)

		// 判断是否完成了所有的Map任务, 如果完成了所有的Map任务，并且ReduceTaskChan还没有被放入任务
		// 判断ReduceTaskChan是为了防止每个TaskFin对ReduceTaskChan在完成Map任务时都添加任务
		if finishedTask+1 == int64(c.NMap) && len(c.ReduceTaskChan) == 0 {

			// fmt.Printf("MapTask Finished, Coordinator into Phase %d\n", c.Phase)
			// 完成所有Map任务后，就要向Reduce待完成任务中添加任务
			for i := 0; i < int(c.NReduce); i++ {
				reduceTask := ReduceTask{
					ReduceID:  i,
					StartTime: -1,
				}
				c.ReduceTaskChan <- reduceTask
			}

			// 将协调器转换状态
			atomic.StoreInt64(&c.Phase, 1)
			// 将阶段任务完成数量归零
			atomic.StoreInt64(&c.FinishedTask, 0)
		}
	} else if mrCoordinatorPhase == 1 {
		// Worker向协调器报告工作状态，首先看这个任务是否Timeout
		value, ok := c.DistributedTask.Load(args.ReduceID)
		if !ok {
			// 如果没有从被分配Map中获取到对应任务，说明Coordinator已经发现该任务超时，并且将该任务重新加入chan了
			return nil
		}
		reduceTask, _ := value.(ReduceTask)
		taskStartTime := reduceTask.StartTime

		// 就算在已分配任务列表中找到了对应元素，也不一定记录的是分配该worker了，因此仍然需要比对当前任务的开始时间和该worker收到的被分配时间，如果不一致就不是分配给该worker的
		// 如果worker处理的任务的分配时间不等于该任务的起始时间，说明该任务被重新分配了，如果相等，还超时了那就按超时算
		if taskStartTime != args.DistributedTime || timeNow-taskStartTime > TimeOut {
			// fmt.Printf("Map Task TimeOut, TaskID: %d, FileName: %s\n", args.TaskID, args.File)
			return nil
		}

		// 如果任务没有超时，说明任务正常完成，在协调器中将该任务从已分配任务列表中删除，并将任务完成计数+1
		c.DistributedTask.Delete(args.ReduceID)
		finishedTask := atomic.LoadInt64(&c.FinishedTask)
		atomic.StoreInt64(&c.FinishedTask, finishedTask+1)

		// 判断是否完成了所有的Reduce任务, 如果完成了所有的Reduce任务，就切换协调器阶段
		if finishedTask+1 == int64(c.NReduce) {
			// 将协调器转换状态
			atomic.StoreInt64(&c.Phase, 2)
		}
	}
	return nil
}

// start a thread that listens for RPCs from worker.go
func (c *Coordinator) server() {
	rpc.Register(c)
	rpc.HandleHTTP()
	//l, e := net.Listen("tcp", ":1234")
	sockname := coordinatorSock()
	os.Remove(sockname)
	l, e := net.Listen("unix", sockname)
	if e != nil {
		log.Fatal("listen error:", e)
	}
	go http.Serve(l, nil)
}

// main/mrcoordinator.go calls Done() periodically to find out
// if the entire job has finished.
func (c *Coordinator) Done() bool {
	ret := false

	// Your code here.
	// 如果协调器的Phase状态进入2，代表任务完成
	if c.Phase == 2 {
		ret = true
		// 同时删除中间文件夹和结果文件锁
		// os.RemoveAll("./intermediate")

		// 让coordinator结束任务后再提供一段时间的rpc，让所有worker都能接收到taskType="done"的通知
		time.Sleep(2 * time.Second)
	}

	return ret
}

// create a Coordinator.
// main/mrcoordinator.go calls this function.
// nReduce is the number of reduce tasks to use.
func MakeCoordinator(files []string, nReduce int) *Coordinator {
	c := Coordinator{
		MapTaskChan:    make(chan MapTask, len(files)),
		ReduceTaskChan: make(chan ReduceTask, nReduce),
		Phase:          0,
		NMap:           len(files),
		NReduce:        nReduce,
	}

	// 为Map任务通道添加任务
	for i, file := range files {
		mapTask := MapTask{
			FileName:  file,
			MapID:     i,
			StartTime: -1,
		}
		c.MapTaskChan <- mapTask
	}

	c.server()
	// fmt.Println("Coordinator Start Success!")
	return &c
}
~~~



### src/mr/rpc.go

~~~go
package mr

//
// RPC definitions.
//
// remember to capitalize all names.
//

import (
	"os"
	"strconv"
)

// Add your RPC definitions here.
type WorkerRequest struct {
	FileName        string // 当任务类型为Map时有意义，为处理完毕的文件名
	ReduceID        int    // 当任务类型为Reduce时有意义，为处理完毕的reduce桶
	DistributedTime int64  // 该任务被分配到worker的时间，用于判断worker是否超时
}

type WorkerReply struct {
	TaskType        string // 任务类型
	File            string // 当任务类型为Map时，有意义，为需要处理的文件名
	NReduce         int    // 当任务类型为Map时有意义，为需要将中间结果划分为的桶数
	TaskID          int    // 当任务类型为Map、Reduce时，有意义，Map:当前处理的是第几个Map任务；Reduce:当前处理的Reduce任务ID
	NMap            int    // 当任务类型为Reduce时，有意义，为需要接收的中间文件数
	DistributedTime int64  // 该任务被分配到Worker的时间
}

// Cook up a unique-ish UNIX-domain socket name
// in /var/tmp, for the coordinator.
// Can't use the current directory since
// Athena AFS doesn't support UNIX-domain sockets.
func coordinatorSock() string {
	s := "/var/tmp/5840-mr-"
	s += strconv.Itoa(os.Getuid())
	return s
}
~~~



### src/mr/worker.go

~~~go
package mr

import (
	"encoding/json"
	"errors"
	"fmt"
	"hash/fnv"
	"io"
	"log"
	"net/rpc"
	"os"
	"sort"
	"strconv"
	"time"
)

// Map functions return a slice of KeyValue.
type KeyValue struct {
	Key   string
	Value string
}

// for sorting by key.，包含两个字段key,value
type ByKey []KeyValue

// for sorting by key.，实现了sort.interface的三个接口方法
func (a ByKey) Len() int           { return len(a) }
func (a ByKey) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByKey) Less(i, j int) bool { return a[i].Key < a[j].Key }

// use ihash(key) % NReduce to choose the reduce
// task number for each KeyValue emitted by Map.
func ihash(key string) int {
	h := fnv.New32a()
	h.Write([]byte(key))
	return int(h.Sum32() & 0x7fffffff)
}

// main/mrworker.go calls this function.
func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {

	// Your worker implementation here.
	// 不断循环，worker不断向coordinator请求任务
	for {
		reply := WorkerReply{}
		// Woker先从协调器获取任务,如果调用rpc失败，说明coordinator已经退出，worker可以退出
		if ok := CallGetTask(&reply); !ok {
			// time.Sleep(time.Second)
			// continue
			log.Printf("Coordinator Missed!\n")
			return
		}
		// fmt.Printf("Woker Get %s Task, File %s, TaskID %d\n", reply.TaskType, reply.File, reply.TaskID)

		if reply.TaskType == "map" {
			if err := DoMap(&reply, mapf); err != nil {
				// 不应该使用Fatalf，这样会杀死worker进程，worker需要能够处理Map出错的情况，处理结果就是放弃该任务，重新请求任务
				// log.Fatalf("Map Task Failed, FileName: %s: %v", reply.File, err)
				log.Printf("Map Task Failed, FileName: %s: %v\n", reply.File, err)
				continue
			}
			// 完成Map任务后，向协调器发送完成任务信号
			args := WorkerRequest{
				FileName:        reply.File,
				DistributedTime: reply.DistributedTime,
			}
			// 如果调用rpc失败，等待1s后重试,但是就不是调用taskFin rpc了，重新请求任务,这个和请求任务感觉不一样，还是重试
			if ok := CallTaskFinished(&args); !ok {
				time.Sleep(time.Second)
				continue
			}
		} else if reply.TaskType == "reduce" {
			if err := DoReduce(&reply, reducef); err != nil {
				// log.Fatalf("Reduce Task Failed, ReduceID: %d: %v", reply.TaskID, err)
				log.Printf("Reduce Task Failed, ReduceID: %d: %v\n", reply.TaskID, err)
				continue
			}
			// 完成Reduce任务后，向协调器发送完成任务信号
			args := WorkerRequest{
				ReduceID:        reply.TaskID,
				DistributedTime: reply.DistributedTime,
			}
			// 如果调用rpc失败，等待1s后重试,但是就不是调用taskFin rpc了，重新请求任务
			if ok := CallTaskFinished(&args); !ok {
				time.Sleep(time.Second)
				continue
			}
		} else if reply.TaskType == "done" {
			// 如果coordinator说所有任务都完成了，worker退出
			// log.Printf("All Task Done!\n")
			return
		}
		// 针对Coordinator给出的TaskType=wait等待类型任务，不用单拎if分支处理，if分支没有wait就自动重试，达到了等待的效果

		// 一个工作完成后，睡眠1秒，避免与其他进程产生冲突
		time.Sleep(time.Second)
	}
}

// send an RPC request to the coordinator, wait for the response.
// usually returns true.
// returns false if something goes wrong.
func call(rpcname string, args interface{}, reply interface{}) bool {
	// c, err := rpc.DialHTTP("tcp", "127.0.0.1"+":1234")
	sockname := coordinatorSock()
	c, err := rpc.DialHTTP("unix", sockname)
	if err != nil {
		log.Println("dialing", err)
		return false
		// log.Fatal("dialing:", err)
	}
	defer c.Close()

	err = c.Call(rpcname, args, reply)
	if err == nil {
		return true
	}

	fmt.Println("calling", err)
	return false
}

// Woker向协调器请求任务
func CallGetTask(reply *WorkerReply) bool {
	args := WorkerRequest{}
	ok := call("Coordinator.AssignTask", &args, reply)
	if ok {
		// fmt.Printf("Woker Call Get Task Success!\n")
		return true
	} else {
		// fmt.Printf("Woker Call Get Task Failed!\n")
		return false
	}
}

func CallTaskFinished(args *WorkerRequest) bool {
	reply := WorkerReply{}
	ok := call("Coordinator.TaskFin", args, &reply)
	if ok {
		// fmt.Printf("Woker Call Finish Task Success!\n")
		return true
	} else {
		// fmt.Printf("Woker Call Finish Task Failed!\n")
		return false
	}
}

// Woker的Map操作
func DoMap(reply *WorkerReply, mapf func(string, string) []KeyValue) error {
	// 打开Map任务的文件
	file, err := os.Open(reply.File)
	if err != nil {
		// 嵌套函数中同样，不应该使用fatalf，而是放弃该任务，重新请求任务
		// log.Fatalf("MapTask Can't Open File %s\n", reply.File)
		log.Printf("MapTask Can't Open File %s\n", reply.File)
		return err
	}
	// 读取文件的内容，然后就可以关闭文件了
	content, err := io.ReadAll(file)
	if err != nil {
		// log.Fatalf("MapTask Read File %s Content Error!\n", reply.File)
		log.Printf("MapTask Read File %s Content Error!\n", reply.File)
		return err
	}
	file.Close()

	// 开始对文件内容进行Map操作,map操作返回<k,v> list
	// 以wc.go为例，将文本内容每个单词都构建为一个键值对，键是单词，值是自定义值，mapf返回键值对列表，没有做融合操
	intermediate := mapf(reply.File, string(content))

	// map阶段输出，需要将中间结果划分为nReduce个存储桶，以此将中间结果交给下面nReduce个reduce工作,
	// 将当前mapf返回的键值列表根据键划分为nReduce个工作
	nBuckets := make([][]KeyValue, reply.NReduce)
	for _, kv := range intermediate {
		// 使用键哈希获取该keyvalue的reduceID，并将其放入对应的桶内
		reduceId := ihash(kv.Key) % reply.NReduce
		nBuckets[reduceId] = append(nBuckets[reduceId], kv)
	}

	// 将Map结果分组后，选择将其以临时文件的方式存储在系统中，方便传给接下来的Reduce工作
	for i := 0; i < reply.NReduce; i++ {
		// 所有产生的中间文件都在创建的intermediate文件夹中，待reduce操作完成后，会删除该文件夹和中间文件
		// 根据Map Reduce设计，每个Map操作都会产生nReduce个中间文件，因此需要mapID来标识不同的Map操作产生的中间文件
		os.Mkdir("./intermediate", 0755)

		// 由于论文要求，每个map任务都输出nReduce个文件，而非所有map任务共享nReduce个中间文件
		// 根据实验要求，可以使用创建临时文件的方式防止worker崩溃后的内容被查看
		tempFile, err := os.CreateTemp("", "mr-reduce-temp")
		if err != nil {
			// log.Fatalf("MapWoker Can't Create Temp File: %v\n", err)
			log.Printf("MapWoker Can't Create Temp File: %v\n", err)
			return err
		}

		// 创建一个JSON编码器，编码器写入文件方式也是官方推荐的
		encoder := json.NewEncoder(tempFile)
		// 将第i个Reduce工作的内容编码并写入临时文件
		// 经过测试，将JSON数组整体写入，比较容易出现JSON非完整错误，将JSON一条条写入文件
		for _, inter := range nBuckets[i] {
			err = encoder.Encode(inter)
			if err != nil {
				// log.Fatalf("MapWoker Encoder Error: %v\n", err)
				log.Printf("MapWoker Encoder Error: %v\n", err)
				return err
			}
		}
		tempFile.Close()

		resultFileName := fmt.Sprintf("./intermediate/mr-%d-%d", reply.TaskID, i)
		os.Rename(tempFile.Name(), resultFileName)
	}
	return nil
}

// Woker的Reduce操作
func DoReduce(reply *WorkerReply, reducef func(string, []string) string) error {
	// 读取中间文件存储的键值对
	intermediate := []KeyValue{}
	reduceID := reply.TaskID

	// 读取该reduce操作所需要的所有中间文件的键值对内容
	for i := 0; i < reply.NMap; i++ {
		intermediateFileName := fmt.Sprintf("./intermediate/mr-%d-%d", i, reduceID)
		file, err := os.Open(intermediateFileName)
		if err != nil {
			// log.Fatalf("ReduceWoker Load intermediate File %s Failed: %v\n", intermediateFileName, err)
			log.Printf("ReduceWoker Load intermediate File %s Failed: %v\n", intermediateFileName, err)
			return err
		}
		// log.Printf("ReduceWorker Load intermediate File Success, TaskID: %d\n", reduceID)
		// 创建该中间文件的json解码器
		decoder := json.NewDecoder(file)
		for {
			// 由于解码器每次只能读取一个完整的json记录, 因此读取一整个文件的json内容，需要循环
			var fileKV KeyValue
			// 如果读到文件尾，就退出该读文件的循环
			if err := decoder.Decode(&fileKV); errors.Is(err, io.EOF) {
				break
			} else if err != nil {
				// log.Fatalf("ReduceWorker Read %s FileKV Failed: %v\n", intermediateFileName, err)
				log.Printf("ReduceWorker Read %s FileKV Failed: %v\n", intermediateFileName, err)
				return err
			}
			// 将该中间文件的键值对内容添加到reduce操作需要处理的中间值列表中
			intermediate = append(intermediate, fileKV)
			// log.Printf("ReduceWorker Read intermediate File Success, TaskID: %d\n", reduceID)
		}
	}

	// 将所有要处理的中间值键值对按键排序
	sort.Sort(ByKey(intermediate))
	// log.Printf("ReduceWorker Sort intermediate File Success, TaskID: %d\n", reduceID)

	// 由于论文要求，每个reduce任务都输出一个文件，而非所有reduce任务共享一个输出文件
	// 因此每个reduce任务都需要创建一个输出文件，根据实验要求，可以使用创建临时文件的方式防止worker崩溃后的内容被查看
	tempFile, err := os.CreateTemp("", "mr-reduce-temp")
	if err != nil {
		// log.Fatalf("ReduceWorker Can't Create Temp File: %v\n", err)
		log.Printf("ReduceWorker Can't Create Temp File: %v\n", err)
		return err
	}

	i := 0
	for i < len(intermediate) {
		j := i + 1
		// 找到所有与intermediate[i].key相同的键值对
		for j < len(intermediate) && intermediate[j].Key == intermediate[i].Key {
			j++
		}
		values := []string{}
		// 将键相同的键值对的value放到同一个List中，
		for k := i; k < j; k++ {
			values = append(values, intermediate[k].Value)
		}

		// log.Printf("ReduceWorker reduce from %d to %d, Key: %s\n", i, j, intermediate[i].Key)

		// 将该键值的键和同键值交给reduce函数处理
		output := reducef(intermediate[i].Key, values)

		fmt.Fprintf(tempFile, "%v %v\n", intermediate[i].Key, output)

		i = j
	}
	// 该reduce任务完成后，修改临时文件名，并关闭文件
	tempFile.Close()
	os.Rename(tempFile.Name(), "mr-out-"+strconv.Itoa(reduceID))
	return nil
}
~~~

## 测试脚本解析和测试结果

* 测试脚本`src/main/test-mr.sh`有如下测试项

  1. wc test

     即Word Count test，该test将学习者实现的MapReduce框架最终生成的mr-out-1~mr-out-10进行sort并统一输出，并与mrsequantial.go输出的mr-out-0进行对比，并行结果与串行结果一致即可通过该测试；

  2. indexer test

     即索引检测.该test主要记录每个word分别来自哪个文本，并检测并行结果的正确性，通常情况下wc.test能够通过后，该test应该同样可以通过；

  3. map parallelism test

     该test主要用于检测学习者实现的MapReduce框架中的worker在执行MapTask时是否能够并行运转

  4. reduce parallelism test

     与上个test相似，主要用于检查ReduceTask执行的并行性;

  5. job count test

     该test主要检测学习者实现的MapReduce框架是否执行的MapTask个数是否与规定个数相同(该Lab中为8个);

  6. early exit test

     test主要检测学习者实现的MapReduce框架是否在所有Task执行完毕之前就会有Master或者Worker退出的情况；

  7. **Crash Test**

     该test采用mrapp/crash.go文件实现worker的随机退出或者随机睡眠，用于检测学习者实现的MapReduce框架是否拥有足够强的Fault Tolerance机制，使之能够保证在部分Worker宕机的情况下仍能够产生正确的Output files

* 测试结果

  ~~~sh
  (base) root@master:~/Golang/code/go/src/MIT-Raft/src/main# ./test-mr.sh 
  *** Starting wc test.
  --- wc test: PASS
  *** Starting indexer test.
  --- indexer test: PASS
  *** Starting map parallelism test.
  --- map parallelism test: PASS
  *** Starting reduce parallelism test.
  --- reduce parallelism test: PASS
  *** Starting job count test.
  --- job count test: PASS
  *** Starting early exit test.
  --- early exit test: PASS
  *** Starting crash test.
  --- crash test: PASS
  *** PASSED ALL TESTS
  ~~~



## 实现过程中遇到的问题

### Reduce读取中间文件时出现unexpected EOF

在Map操作向中间文件写键值对时，有两种方式写键值对

1. 一次写入该中间文件对应的所有键值对，也就是写JSON数组

   ~~~go
   		// 创建一个JSON编码器
   		encoder := json.NewEncoder(tempFile)
   		// 将第i个Reduce工作的内容编码并写入临时文件
   		err = encoder.Encode(nBuckets[i])
   		if err != nil {
   			log.Fatalf("MapWoker Encoder Error: %v", err)
   		}
   		tempFile.Close()
   ~~~

   文件内容

   ~~~
   [{"Key":"bar","Value":"1"},{"Key":"baz","Value":"1"}]
   ~~~

2. 逐个写入键值对

   ~~~go
   		// 创建一个JSON编码器，编码器写入文件方式也是官方推荐的
   		encoder := json.NewEncoder(tempFile)
   		// 将第i个Reduce工作的内容编码并写入临时文件
   		// 经过测试，将JSON数组整体写入，比较容易出现JSON非完整错误，将JSON一条条写入文件
   		for _, inter := range nBuckets[i] {
   			err = encoder.Encode(inter)
   			if err != nil {
   				// log.Fatalf("MapWoker Encoder Error: %v\n", err)
   				log.Printf("MapWoker Encoder Error: %v\n", err)
   				return err
   			}
   		}
   		tempFile.Close()
   ~~~

   文件内容

   ~~~go
   {"Key":"bar","Value":"1"}
   {"Key":"baz","Value":"1"}
   ~~~

如第二种方法的注释所说， 第一种方法当遇上Crash时，可能会出现文件内容不完整的情况，例如最后一个`]`没有被写入，这样reduce在读取时，会出现unexpected EOF错误

### 不要将nMap * nReduce个中间文件，在Map写中间文件时，利用文件锁合并为nReduce个中间文件

* 官方论文 & Lab 的设计是：

  - **Map worker：**

    只写 **自己负责的中间文件**

    完全不和其他 worker 共享文件

  - **Reduce worker：**

    去读 **所有 Map 产生的、属于自己的那一份**

如果在Map阶段合并，会有如下风险

1. 多 worker 并发写

2. 文件锁竞争

3. 死锁 / 饥饿

4. 崩溃时文件处于半写状态

5. 很难保证幂等

6. 同时Lab1要求，同一个Map任务可以被执行多次，结果任然正确，如果合并写

   第一次执行写了一半，第二次执行再写一遍，结果直接重复

### 当有多个Worker调用coordinator的AssignTask RPC时，coordinator会一个个处理还是同时启动多个AssignTask线程？

多个 Worker 同时调用 `AssignTask` 时：

- Coordinator 的 `AssignTask` 会被“并发调用”
- 每个 RPC 请求运行在独立的 goroutine 中
- 不是排队串行执行的

### Worker必须不能Fatalf，必须能够处理不同异常情况

* 异常情况

  中间文件不存在、中间文件是截断的、JSON解析失败，RPC失败，Coordinator消失等情况

* 处理方法

  Worker放弃该次任务，重新进入一次向coordinator请求任务的循环

  **当然，如果Coordinator消失，我们可以假定任务已经完成，然后退出Worker进程**

### 分配Map任务时，MapID不可以通过Map任务数量-未分配数量来赋值

由于未分配数量会动态变化，如果一个任务超时，那么其第一次分配和第二次分配的ID可能会不一样，并且可能与其他的Map任务的Map ID相同，导致错误

### 使用Chan分配任务时，当Chan中没有未分配任务时，不可以阻塞

分配任务管道不能阻塞 

说是不能阻塞，想象，当最后一个Map任务被分配出去，MapTaskChan就是空的，并且状态Map也是空的 

**但是其他Worker请求任务，由于Coordinator还在Map阶段，就会在这里阻塞住，那么当最后一个Map任务完成，其他阻塞在这里的Worker应该如何？** 

因此，不能在这里阻塞住，如果没有任务分配，就让Worker进入等待状态



# Lab2 Key/Value Server

## KV Server介绍

* 实验链接：https://pdos.csail.mit.edu/6.824/labs/lab-kvsrv1.html

  每个客户端都使用 Clerk 与键/值服务器交互，该 *Clerk* 向服务器发送 RPC。 客户端可以向服务器发送两种不同的 RPC： `Put(key, value, version)` 和 `Get(key)` 。服务器 维护一个内存映射，该映射记录每个键的（值， 版本）元组。键和值均为字符串。版本号 记录密钥被写入的次数。 `Put(key, value, version)` *仅在 `Put` 的版本号与服务器上对应键的版本号匹配时，* 才会安装或替换映射中特定键的值。如果版本号匹配，服务器还会递增该键的版本号。如果版本号不匹配，服务器应返回 `rpc.ErrVersion` 。客户端可以通过调用 `Put` 并传入版本号 0 来创建一个新键（服务器存储的版本号将为 1）。如果 `Put` 的版本号大于 0 且该键不存在，服务器应返回 `rpc.ErrNoKey` 。

* 目标效果

  当你完成这个实验并通过所有测试后，你将拥有一个*线性化的*键/值服务。 客户来电的视角 `Clerk.Get` 和 `Clerk.Put` 。也就是说，如果客户端操作不是并发的，则每个客户端 `Clerk.Get` 和 `Clerk.Put` 都会观察到前一个操作序列所隐含的状态变化。对于并发操作，返回值和最终状态与按某种顺序逐个执行的操作相同。如果操作在时间上重叠，则它们是并发的：例如，如果客户端 X 调用 `Clerk.Put()` ，客户端 Y 调用 `Clerk.Put()` ，然后客户端 X 的调用返回。一个操作必须观察到在其开始之前已完成的所有操作的效果。有关更多背景信息，请参阅关于[线性化的](https://pdos.csail.mit.edu/6.824/papers/linearizability-faq.txt)常见问题解答。



## 具有可靠网络的键值服务器

* 实验要求

  第一个实现的目标其实很简单，就是实现client中关于调用server端的put和get RPC的代码逻辑，注意点就是要看该实验中如何调用RPC

  这个实验的启动和测试与第一个实验不一样，其Server和client的启动都不需要我们手动启动，server也不需要我们注册rpc，注意看代码上方的注释即可

  server端需要实现的，首先是存储KV键值对的存储结构，其次是各个put和get RPC互斥访问KV存储结构的代码逻辑

### /kvsrv1/server.go

~~~go
// 存储结构修改
type ValueVersionPair struct {
	Value   string
	Version rpc.Tversion
}

type KVServer struct {
	mu sync.Mutex

	// Your definitions here.
	// 由于KV Server默认带了一个同步锁，并且需要线性处理每个请求，因此不计划采用sync.map
	KVMap map[string]ValueVersionPair
}
func MakeKVServer() *KVServer {
	// 初始化键值服务器
	kv := &KVServer{
		KVMap: make(map[string]ValueVersionPair, 0),
	}
	// Your code here.
	return kv
}

// Get returns the value and version for args.Key, if args.Key
// exists. Otherwise, Get returns ErrNoKey.
func (kv *KVServer) Get(args *rpc.GetArgs, reply *rpc.GetReply) {
	// Your code here.
	// 先获取KVServer的锁
	kv.mu.Lock()
	defer kv.mu.Unlock()

	// log.Printf("Server Get Called, Args, Key %s\n", args.Key)

	// 判断是否存在对应的键值对,如果存在，正常返回，否则返回ErrNoKey
	if vvpair, ok := kv.KVMap[args.Key]; ok {
		reply.Err = rpc.OK
		reply.Value = vvpair.Value
		reply.Version = vvpair.Version
		// log.Printf("Server Get Done, Ok\n")

	} else {
		// log.Printf("Server Get Done, NoKey\n")
		reply.Err = rpc.ErrNoKey
	}
}

// Update the value for a key if args.Version matches the version of
// the key on the server. If versions don't match, return ErrVersion.
// If the key doesn't exist, Put installs the value if the
// args.Version is 0, and returns ErrNoKey otherwise.
func (kv *KVServer) Put(args *rpc.PutArgs, reply *rpc.PutReply) {
	// Your code here.
	// 先获取KVServer的锁
	kv.mu.Lock()
	defer kv.mu.Unlock()

	// log.Printf("Server Put Called, Args, Key %s, Value %s, Version %d\n", args.Key, args.Value, args.Version)
	// 如果Put的键在Map中存在
	if vvpair, ok := kv.KVMap[args.Key]; ok {
		// 判断Put的版本号是否一致, 如果版本号一致，允许修改Value，并Version+1
		if args.Version == vvpair.Version {
			kv.KVMap[args.Key] = ValueVersionPair{
				Value:   args.Value,
				Version: args.Version + 1,
			}
			reply.Err = rpc.OK
			// log.Printf("Server Get Done, OK\n")
		} else {
			// 如果版本号不匹配，返回ErrVersion
			reply.Err = rpc.ErrVersion
			// log.Printf("Server Get Done, ErrVersion\n")
		}
	} else if args.Version == 0 {
		// 如果Put的键不存在，但是Put的Version是0，说明要创建一个新键
		kv.KVMap[args.Key] = ValueVersionPair{
			Value:   args.Value,
			Version: 1,
		}
		reply.Err = rpc.OK
		// log.Printf("Server Get Done, OK\n")
	} else {
		// 如果Put的键不存在，并且Version也不是0，应该返回ErrNoKey
		reply.Err = rpc.ErrNoKey
		// log.Printf("Server Get Done, ErrNoKey\n")
	}
}
~~~

### /kvsrv1/client.go

~~~go
type Clerk struct {
	clnt   *tester.Clnt
	server string
}

func MakeClerk(clnt *tester.Clnt, server string) kvtest.IKVClerk {
	ck := &Clerk{clnt: clnt, server: server}
	// You may add code here.
	return ck
}

// Get fetches the current value and version for a key.  It returns
// ErrNoKey if the key does not exist. It keeps trying forever in the
// face of all other errors.
//
// You can send an RPC with code like this:
// ok := ck.clnt.Call(ck.server, "KVServer.Get", &args, &reply)
//
// The types of args and reply (including whether they are pointers)
// must match the declared types of the RPC handler function's
// arguments. Additionally, reply must be passed as a pointer.
func (ck *Clerk) Get(key string) (string, rpc.Tversion, rpc.Err) {
	// You will have to modify this function.
	return "", 0, rpc.ErrNoKey
	// 初始化调用RPC需要传入的两个参数
	getArgs := rpc.GetArgs{
		Key: key,
	}
	getReply := rpc.GetReply{}

	ck.clnt.Call(ck.server, "KVServer.Get", &getArgs, &getReply)
	return getReply.Value, getReply.Version, getReply.Err
}

// Put updates key with value only if the version in the
// request matches the version of the key at the server.  If the
// versions numbers don't match, the server should return
// ErrVersion.  If Put receives an ErrVersion on its first RPC, Put
// should return ErrVersion, since the Put was definitely not
// performed at the server. If the server returns ErrVersion on a
// resend RPC, then Put must return ErrMaybe to the application, since
// its earlier RPC might have been processed by the server successfully
// but the response was lost, and the Clerk doesn't know if
// the Put was performed or not.
//
// You can send an RPC with code like this:
// ok := ck.clnt.Call(ck.server, "KVServer.Put", &args, &reply)
//
// The types of args and reply (including whether they are pointers)
// must match the declared types of the RPC handler function's
// arguments. Additionally, reply must be passed as a pointer.
func (ck *Clerk) Put(key, value string, version rpc.Tversion) rpc.Err {
	// You will have to modify this function.
	return rpc.ErrNoKey
	// 初始化调用RPC需要传入的两个参数
	putArgs := rpc.PutArgs{
		Key:     key,
		Value:   value,
		Version: version,
	}
	putReply := rpc.PutReply{}

	ck.clnt.Call(ck.server, "KVServer.Put", &putArgs, &putReply)
	return putReply.Err
}
~~~

### 测试结果

1. 可靠性测试

   ~~~sh
   (base) root@master:~/Golang/code/go/src/MIT-Raft/src/kvsrv1# go test -v -run Reliable
   === RUN   TestReliablePut
   One client and reliable Put (reliable network)...
     ... Passed --  time  0.0s #peers 1 #RPCs     5 #Ops    0
   --- PASS: TestReliablePut (0.00s)
   === RUN   TestPutConcurrentReliable
   Test: many clients racing to put values to the same key (reliable network)...
   info: linearizability check timed out, assuming history is ok
     ... Passed --  time 11.1s #peers 1 #RPCs 99749 #Ops 99749
   --- PASS: TestPutConcurrentReliable (11.07s)
   === RUN   TestMemPutManyClientsReliable
   Test: memory use many put clients (reliable network)...
     ... Passed --  time  5.9s #peers 1 #RPCs 100000 #Ops    0
   --- PASS: TestMemPutManyClientsReliable (10.81s)
   PASS
   ok      6.5840/kvsrv1   21.925s
   ~~~

2. 竞争测试

   ~~~sh
   (base) root@master:~/Golang/code/go/src/MIT-Raft/src/kvsrv1# go test -race
   One client and reliable Put (reliable network)...
     ... Passed --  time  0.0s #peers 1 #RPCs     5 #Ops    0
   Test: many clients racing to put values to the same key (reliable network)...
     ... Passed --  time  8.5s #peers 1 #RPCs 18601 #Ops 18601
   Test: memory use many put clients (reliable network)...
     ... Passed --  time 20.1s #peers 1 #RPCs 100000 #Ops    0
   One client (unreliable network)...
   Fatal: 0: Get "k" err 
           /root/Golang/code/go/src/MIT-Raft/src/kvtest1/kvtest.go:178
           /root/Golang/code/go/src/MIT-Raft/src/kvsrv1/kvsrv_test.go:150
   --- FAIL: TestUnreliableNet (0.07s)
   FAIL
   exit status 1
   FAIL    6.5840/kvsrv1   48.086s
   ~~~

   这里测试失败是因为还没有完成后面的不可靠网络实现

## 使用键值对实现分层锁

> 其实针对这个实验，最重要的就是理解这个分层锁的运行机制。

* 官网说明是：

  在本练习中，你的任务是实现一个分层锁。 客户 调用 `Clerk.Put` 和 `Clerk.Get` 。该锁支持两种方法： `Acquire` 和 `Release` 。该锁的规范是，一次只能有一个客户端成功获取该锁；其他客户端必须等待第一个客户端使用 `Release` 释放该锁。

  我们在 `src/kvsrv1/lock/` 中提供了框架代码和测试。您需要修改 `src/kvsrv1/lock/lock.go` 。您的 `Acquire` 和 `Release` 代码可以通过调用 `lk.ck.Put()` 与您的键/值服务器通信。 和 `lk.ck.Get()` 。如果客户端在持有锁时崩溃， 锁永远不会打开。

  每个锁客户端都需要一个唯一的标识符；调用 `kvtest.RandValue(8)` 生成一个随机字符串。

  锁定服务应使用特定键来存储“锁定状态”（您需要具体定义锁定状态）。要使用的键通过参数 `MakeLock` 的 `l` 传递。 在 `src/kvsrv1/lock/lock.go` 中。

* 我的理解是：

  现在假定，有多个应用都在使用同一个KV Server服务器。其中client1和client2都是属于同一个应用的两个客户端。现在client1想对该应用的某个键值对做出两次修改，并且第二次修改是基于第一次修改的；client2也想对同一个键值对做两次修改，第二次修改也是基于第一次修改的；

  然而，当client1的第一次操作put11调用KV  Server的Put操作后，在执行第二次修改put12之前，该键值对的状态可能会被client2的第一次修改put21变更，这就使得client1的第二次修改非常可能会出错，这就需要client1将这两次修改作为一个逻辑原子操作执行。

  最好的方法就是当client1和client2在执行两次修改之前，先获取关于这个应用的分层互斥锁，只有获取到锁的client才可以对该应用相关的KV键值对进行修改，在client执行修改完毕后，再释放该应用的分层锁。这就是分层锁存在的意义场景。
  现在我们知道，分层锁和KV Server中的sync.mutex的区别

  1. 分层锁：**分层锁处理的是跨RPC的逻辑临界区操作，让两个RPC操作是“原子的”**
  2. sync.mutex：该锁，管理的是KV Server针对并发访问KV存储结构并修改的互斥访问，**是在每个RPC调用之中的，在分层锁的下层**

  那么该分层锁应该如何实现？

  既然分层锁是在KV Server逻辑之外的，并且能够被所有的client访问到，我们的第一反应是在KV Server的结构体中再添加一个分层锁的数组或切片，让所有的client都能访问到
  但是实际上，**并不需要添加新的存储结构，继续沿用原本的KV键值对存储结构即可，让Key作为分层锁的名字，对应不同的应用，Value对应锁的状态**

  好的，我们开始实现锁的Acquire操作时，遇到新的问题：
  
  1. 假定现在有两个客户端在Acquire同一个分层锁，通过判定该锁的状态来进行争夺，可能会出现两个客户端的Acquire都判定没有上锁后，都调用对分层锁的Put操作，进行上锁，这样会**导致锁的线性操作，这样可能会出现客户端1以为拿到了锁，但通过线性操作实际上锁被客户端2拿到的情况**，这应该如何解决呢？
  2. 到了需要Release锁的时候，**由于Acquire并不返回随机设定的client ID，那在release中如何判定该锁是不是该客户端持有的呢？**
  
  我们开始逐个思考问题的解决：
  
  1. 针对问题1：我们忽略了put操作的一个参数`version`，假定两个客户端都通过get操作发现锁是未上锁状态，一定都会执行`put(lockName, clientKey, version)`，可以发现，两个客户端传入的`version`都是一样的，根据上个实验中put操作的定义，**一定会有一个客户端的put操作返回ErrVersion，这就使得该客户端上锁失败，当该客户端发现上锁失败，自然应该返回重试上锁操作**
  
  2. 针对问题2：我们需要注意的是，分层锁的上锁状态lockState，为空时表示未上锁状态，不为空时，应当为持有锁的clientID，当客户端想要获取分层锁，**首先需要调用`MakeLock`，创造一个分层锁，该方法返回给客户端的并不是锁本身，而是锁的一个把柄，其中包含锁的名称，客户端对应该锁的钥匙也就是clientID，每个客户端在调用`MakeLock`时，对应的锁钥匙clientID都是不一样的**
  
     **当客户端获取到锁，并且上锁时，将锁本身的“锁芯”也就是lockState改为自己的clientID，也就是钥匙；当客户端想要释放锁的时候，首先检查自己的clientID与锁本身的lockState是否对应，如果不对应，是无法释放锁的；**只有持有锁的客户端才可以成功释放锁

### /kvsrv1/lock/lock.go

~~~go
type Lock struct {
	// IKVClerk is a go interface for k/v clerks: the interface hides
	// the specific Clerk type of ck but promises that ck supports
	// Put and Get.  The tester passes the clerk in when calling
	// MakeLock().
	ck kvtest.IKVClerk
	// You may add code here
	LockName  string // 对应该分层锁的名称
	ClientKey string // 对应该分层锁的状态，如果为空就是未上锁，如果不为空，应该是持有该锁的clientID
}

// The tester calls MakeLock() and passes in a k/v clerk; your code can
// perform a Put or Get by calling lk.ck.Put() or lk.ck.Get().
//
// Use l as the key to store the "lock state" (you would have to decide
// precisely what the lock state is).
func MakeLock(ck kvtest.IKVClerk, l string) *Lock {
	lk := &Lock{ck: ck}
	lk := &Lock{
		ck:        ck,
		LockName:  l,
		ClientKey: kvtest.RandValue(8),
	}
	// You may add code here

	// 将分层锁创建在KVMap中，利用Put操作，一开始创建锁的时候，不上锁，也就是value为空
	// 如果添加锁失败，那么说明KVMap中已经有了该锁
	// 所以这里的Err不接收并判定情况也没事，只需要保证KVMap中有该锁就可以
	ck.Put(lk.LockName, "", 0)
	return lk
}

func (lk *Lock) Acquire() {
	// Your code here
	// 获取对应的锁，如果对应的分层锁是上锁的状态，就一直Get，直到没有上错
	for {
		lockCode, lockVersion, lockErr := lk.ck.Get(lk.LockName)
		// 如果获取锁失败，或者锁状态不为未上锁，继续获取锁
		if lockErr != rpc.OK || lockCode != "" {
			continue
		}

		// 分层锁未上锁，并且获取到锁版本，可以上锁,将该客户端的钥匙（锁芯）放入KVMap，使得该锁目前只有该客户端可以解锁
		if lockErr = lk.ck.Put(lk.LockName, lk.ClientKey, lockVersion); lockErr == rpc.OK {
			// 如果上锁失败，继续获取锁，如果上锁成功，则跳出循环
			// 由于version的存在，当两个客户端同时执行上锁操作时，RPC的线性操作，导致后者执行上锁操作时，Version版本会不一样，导致上锁出错
			break
		}
		time.Sleep(1 * time.Second)
	}
}

func (lk *Lock) Release() {
	// Your code here
	// 获取该锁的状态，判定该客户端持有的钥匙，是否可以匹配上该锁的锁芯
	lockCode, lockVersion, lockErr := lk.ck.Get(lk.LockName)
	// 如果获取锁失败，或者客户端发现自己不是持有该锁的人，应该返回解锁失败吧
	if lockErr != rpc.OK || lockCode != lk.ClientKey {
		log.Fatalf("Release Lock %s Failed: %v\n", lk.LockName, lockErr)
	}

	// 发现自己是该锁的持有者，合法释放该锁
	if lockErr = lk.ck.Put(lk.LockName, "", lockVersion); lockErr != rpc.OK {
		log.Fatalf("Release Lock %s Failed: %v\n", lk.LockName, lockErr)
	}
}
~~~

### 测试结果

~~~sh
(base) root@master:~/Golang/code/go/src/MIT-Raft/src/kvsrv1/lock# go test -v -run Reliable
=== RUN   TestOneClientReliable
Test: 1 lock clients (reliable network)...
  ... Passed --  time  2.0s #peers 1 #RPCs  1332 #Ops    0
--- PASS: TestOneClientReliable (2.00s)
=== RUN   TestManyClientsReliable
Test: 10 lock clients (reliable network)...
  ... Passed --  time  4.0s #peers 1 #RPCs  7390 #Ops    0
--- PASS: TestManyClientsReliable (4.04s)
PASS
ok      6.5840/kvsrv1/lock      6.041s
~~~

## 键值服务器消息丢失情况

* 官网说明：

  如果网络丢弃了 RPC 请求消息，那么客户端重新发送请求即可解决问题：服务器将接收并执行重新发送的请求。

  然而，网络可能会丢弃 RPC 回复消息。客户端并不知道是哪条消息被丢弃了；客户端只会观察到没有收到回复。如果被丢弃的是回复消息，并且客户端重新发送了 RPC 请求，那么服务器将收到两份请求副本。对于 `Get` 来说是可以接受的，因为 `Get` 不会修改服务器状态。重新发送版本号相同的 `Put` RPC 也是安全的，因为服务器会执行 `Put` 。 根据版本号进行条件判断；如果服务器收到并且 执行 `Put` RPC，它将对重新传输的 RPC 副本做出响应，使用 `rpc.ErrVersion` 而不是再次执行 Put。

  棘手的情况是，如果服务器在对 Clerk 重试的 RPC 请求的响应中返回 `rpc.ErrVersion` 。在这种情况下，Clerk 无法确定服务器是否执行了 Clerk 的 `Put` ：服务器可能执行了第一个 RPC 请求，但网络可能丢弃了服务器返回的成功响应，因此服务器仅针对重传的 RPC 请求发送了 `rpc.ErrVersion` 。 或者，也可能是另一种情况。 Clerk在第一个 RPC 请求到达服务器之前更新了密钥。 因此服务器既没有执行Clerk的 RPC，也没有执行其他任何 RPC，并回复了该请求。 `rpc.ErrVersion` 对两者都适用。因此，如果Clerk收到 `rpc.ErrVersion` ， 重新传输 Put RPC， `Clerk.Put` 必须向应用程序返回 `rpc.ErrMaybe` 而不是 `rpc.ErrVersion` ，因为请求可能已被执行。此时，应用程序需要处理这种情况。如果服务器对初始（未重传）的 Put RPC 响应 `rpc.ErrVersion` ，则 Clerk 应向应用程序返回 `rpc.ErrVersion` ，因为服务器肯定没有执行该 RPC。

  现在你应该修改你的 `kvsrv1/client.go` 即使面临失败也要继续前进 RPC 请求和响应。 客户端的 `ck.clnt.Call()` 返回值为 `true` 这表明客户 收到来自 服务器；返回值 `false` 表示未收到回复（更准确地说， `Call()` 等待）。 对于超时时间间隔内的回复消息，返回 false 如果在规定时间内没有收到回复）。 你的 `Clerk` 应该持续重新发送 RPC 请求，直到收到回复为止。请参考上文 `rpc.ErrMaybe` 的讨论。您的解决方案无需对服务器进行任何更改。

* 我的理解：

  其实该实验就很好理解了，就是要解决客户端可能面临的在网络不稳定的情况下，如何让Get和Put操作仍然能够正常运行。

  官网说明也有提到，棘手的情况是，当客户端发送过一次Put请求后，如果Server的回复消息丢失，当客户端再次重试Put操作时，Server发回的错误可能是`ErrVersion`，这时候客户端就不知道操作是否真正被执行了

  具体解决方案直接见代码

### /kvsrv1/client.go

~~~go
type Clerk struct {
	clnt   *tester.Clnt
	server string
}

func MakeClerk(clnt *tester.Clnt, server string) kvtest.IKVClerk {
	ck := &Clerk{clnt: clnt, server: server}
	// You may add code here.
	return ck
}

// Get fetches the current value and version for a key.  It returns
// ErrNoKey if the key does not exist. It keeps trying forever in the
// face of all other errors.
//
// You can send an RPC with code like this:
// ok := ck.clnt.Call(ck.server, "KVServer.Get", &args, &reply)
//
// The types of args and reply (including whether they are pointers)
// must match the declared types of the RPC handler function's
// arguments. Additionally, reply must be passed as a pointer.
func (ck *Clerk) Get(key string) (string, rpc.Tversion, rpc.Err) {
	// You will have to modify this function.
	// 初始化调用RPC需要传入的两个参数
	getArgs := rpc.GetArgs{
		Key: key,
	}
	getReply := rpc.GetReply{}

	// 对于Get操作来说，不论多少次重试都没关系，因为Get操作不改变服务器状态
	for {
		ok := ck.clnt.Call(ck.server, "KVServer.Get", &getArgs, &getReply)
		if ok {
			return getReply.Value, getReply.Version, getReply.Err
		}
		time.Sleep(100 * time.Millisecond)
	}
}

// Put updates key with value only if the version in the
// request matches the version of the key at the server.  If the
// versions numbers don't match, the server should return
// ErrVersion.  If Put receives an ErrVersion on its first RPC, Put
// should return ErrVersion, since the Put was definitely not
// performed at the server. If the server returns ErrVersion on a
// resend RPC, then Put must return ErrMaybe to the application, since
// its earlier RPC might have been processed by the server successfully
// but the response was lost, and the Clerk doesn't know if
// the Put was performed or not.
//
// You can send an RPC with code like this:
// ok := ck.clnt.Call(ck.server, "KVServer.Put", &args, &reply)
//
// The types of args and reply (including whether they are pointers)
// must match the declared types of the RPC handler function's
// arguments. Additionally, reply must be passed as a pointer.
func (ck *Clerk) Put(key, value string, version rpc.Tversion) rpc.Err {
	// You will have to modify this function.
	// 初始化调用RPC需要传入的两个参数
	putArgs := rpc.PutArgs{
		Key:     key,
		Value:   value,
		Version: version,
	}
	putReply := rpc.PutReply{}

	// 一直重试，直到Client收到Server的Reply
	retryCount := 0
	for {
		ok := ck.clnt.Call(ck.server, "KVServer.Put", &putArgs, &putReply)
		retryCount++ // 增加重传计数
		// 如果是第一次发送，就收到了Server的回复，直接返回服务器的错误
		if ok && retryCount == 1 {
			return putReply.Err
		} else if ok && retryCount != 1 {
			// 如果收到了回复，但是不是第一次发送，即重试过，那么返回:可能错误
			// 收到的错误，也可能不是ErrVersion，如果是其他两种错误，就直接返回
			if putReply.Err != rpc.ErrVersion {
				return putReply.Err
			} else {
				return rpc.ErrMaybe
			}
		}
		time.Sleep(100 * time.Millisecond)
	}
}
~~~

### 测试结果

~~~sh
(locust-env) root@master:~/Golang/code/go/src/MIT-Raft/src/kvsrv1# go test -v
=== RUN   TestReliablePut
One client and reliable Put (reliable network)...
  ... Passed --  time  0.0s #peers 1 #RPCs     5 #Ops    0
--- PASS: TestReliablePut (0.00s)
=== RUN   TestPutConcurrentReliable
Test: many clients racing to put values to the same key (reliable network)...
  ... Passed --  time  7.2s #peers 1 #RPCs 99653 #Ops 99653
--- PASS: TestPutConcurrentReliable (7.19s)
=== RUN   TestMemPutManyClientsReliable
Test: memory use many put clients (reliable network)...
  ... Passed --  time  5.9s #peers 1 #RPCs 100000 #Ops    0
--- PASS: TestMemPutManyClientsReliable (10.54s)
=== RUN   TestUnreliableNet
One client (unreliable network)...
  ... Passed --  time  3.5s #peers 1 #RPCs   259 #Ops  212
--- PASS: TestUnreliableNet (3.48s)
PASS
ok      6.5840/kvsrv1   21.233s
~~~

## 在不可靠网络情况下实现分层锁

* 同样的，在网络不稳定情况下，分层锁的Acquire和Release操作可能也会出现获取失败，释放失败的情况，这也是需要解决的问题

  解决方法直接见代码

### /kvsrv1/lock/lock.go

~~~go
type Lock struct {
	// IKVClerk is a go interface for k/v clerks: the interface hides
	// the specific Clerk type of ck but promises that ck supports
	// Put and Get.  The tester passes the clerk in when calling
	// MakeLock().
	ck kvtest.IKVClerk
	// You may add code here
	LockName  string // 对应该分层锁的名称
	ClientKey string // 对应该分层锁的状态，如果为空就是未上锁，如果不为空，应该是持有该锁的clientID
}

// The tester calls MakeLock() and passes in a k/v clerk; your code can
// perform a Put or Get by calling lk.ck.Put() or lk.ck.Get().
//
// Use l as the key to store the "lock state" (you would have to decide
// precisely what the lock state is).
func MakeLock(ck kvtest.IKVClerk, l string) *Lock {
	lk := &Lock{
		ck:        ck,
		LockName:  l,
		ClientKey: kvtest.RandValue(8),
	}
	// You may add code here

	// 将分层锁创建在KVMap中，利用Put操作，一开始创建锁的时候，不上锁，也就是value为空
	// 如果添加锁失败，那么说明KVMap中已经有了该锁
	// 所以这里的Err不接收并判定情况也没事，只需要保证KVMap中有该锁就可以
	ck.Put(lk.LockName, "", 0)
	return lk
}

func (lk *Lock) Acquire() {
	// Your code here
	// 获取对应的锁，如果对应的分层锁是上锁的状态，就一直Get，直到没有上错
	for {
		lockCode, lockVersion, lockErr := lk.ck.Get(lk.LockName)
		// 如果获取锁失败，或者锁状态不为未上锁，继续获取锁
		if lockErr != rpc.OK || lockCode != "" {
			continue
		}

		// 分层锁未上锁，并且获取到锁版本，可以上锁,将该客户端的钥匙（锁芯）放入KVMap，使得该锁目前只有该客户端可以解锁
		if lockErr = lk.ck.Put(lk.LockName, lk.ClientKey, lockVersion); lockErr == rpc.OK {
			// 如果上锁失败，继续获取锁，如果上锁成功，则跳出循环
			// 由于version的存在，当两个客户端同时执行上锁操作时，RPC的线性操作，导致后者执行上锁操作时，Version版本会不一样，导致上锁出错
			break
		} else if lockErr == rpc.ErrMaybe {
			// 如果在上锁的途中，发生了丢包，返回了ErrMaybe错误，检查目前的锁芯是否是该客户端对应的
			// 如果是，那说明上锁成功了，如果不是，那就是被别人锁了，或者还没有上锁，重新获取锁
			lockCode, _, _ := lk.ck.Get(lk.LockName)
			if lockCode == lk.ClientKey {
				break
			}
		}
		time.Sleep(1 * time.Second)
	}
}

func (lk *Lock) Release() {
	// Your code here
	// 由于网络不可靠，所以可能会出现锁释放失败的情况,因此，针对可能失败的情况需要单独处理
	retryCount := 0
	for {
		// 获取该锁的状态，判定该客户端持有的钥匙，是否可以匹配上该锁的锁芯
		retryCount++
		lockCode, lockVersion, lockErr := lk.ck.Get(lk.LockName)
		// 如果获取锁失败，应该返回解锁失败吧
		if lockErr != rpc.OK {
			log.Fatalf("Release Lock %s Failed, Get Lock Failed:%v\n", lk.LockName, lockErr)
		}

		// 获取锁成功，并且发现自己不是锁的持有者
		if lockCode != lk.ClientKey {
			// 如果是第一次尝试解锁，那就逻辑错误，出现问题
			if retryCount == 1 {
				log.Fatalf("Release Lock %s Failed, retryCount=1, LockCode Error:%v\n", lk.LockName, lockErr)
			} else {
				// 如果不是第一次尝试解锁，那么说明锁可能已经被之前的尝试解锁操作解锁成功了
				break
			}
		}

		// 发现自己是该锁的持有者，合法释放该锁，如果释放成功，就退出循环
		lockErr = lk.ck.Put(lk.LockName, "", lockVersion)
		if lockErr == rpc.OK {
			break
		} else if lockErr != rpc.ErrMaybe {
			// 否则，如果返回的错误不是ErrMaybe，报错
			log.Fatalf("Release Lock %s Failed, release lock Error:%v\n", lk.LockName, lockErr)
		}
		// 如果返回的错误是ErrMaybe，重试释放锁
	}
}
~~~

### 测试结果

1. 所有测试

   ~~~sh
   (locust-env) root@master:~/Golang/code/go/src/MIT-Raft/src/kvsrv1/lock# go test -v
   === RUN   TestOneClientReliable
   Test: 1 lock clients (reliable network)...
     ... Passed --  time  2.0s #peers 1 #RPCs  1332 #Ops    0
   --- PASS: TestOneClientReliable (2.00s)
   === RUN   TestManyClientsReliable
   Test: 10 lock clients (reliable network)...
     ... Passed --  time  4.1s #peers 1 #RPCs  7080 #Ops    0
   --- PASS: TestManyClientsReliable (4.06s)
   === RUN   TestOneClientUnreliable
   Test: 1 lock clients (unreliable network)...
     ... Passed --  time  2.5s #peers 1 #RPCs    73 #Ops    0
   --- PASS: TestOneClientUnreliable (2.54s)
   === RUN   TestManyClientsUnreliable
   Test: 10 lock clients (unreliable network)...
     ... Passed --  time  4.5s #peers 1 #RPCs   555 #Ops    0
   --- PASS: TestManyClientsUnreliable (4.50s)
   PASS
   ok      6.5840/kvsrv1/lock      13.104s
   ~~~

2. 竞争测试

   ~~~sh
   (locust-env) root@master:~/Golang/code/go/src/MIT-Raft/src/kvsrv1# go test -race
   One client and reliable Put (reliable network)...
     ... Passed --  time  0.0s #peers 1 #RPCs     5 #Ops    0
   Test: many clients racing to put values to the same key (reliable network)...
     ... Passed --  time  9.6s #peers 1 #RPCs 19145 #Ops 19145
   Test: memory use many put clients (reliable network)...
     ... Passed --  time 19.9s #peers 1 #RPCs 100000 #Ops    0
   One client (unreliable network)...
     ... Passed --  time  8.6s #peers 1 #RPCs   263 #Ops  212
   PASS
   ok      6.5840/kvsrv1   58.300s
   ~~~





# Lab3：Raft

## Part 3A： Leader Election

### 测试项详述

* InitialElection

* ReElection

  1. 初始状态

     启动 3 个服务器，确认存在一个 Leader（`leader1`）。

  2. 断开原 Leader

     将 `leader1` 完全断网（`DisconnectAll(leader1)`），使其无法通信。

     剩余 2 个节点构成多数派（quorum），应能**选出新 Leader（`leader2`）**。

  3. 原 Leader 重新加入

     恢复 `leader1` 的网络连接。

     验证：

     - 集群仍只有一个 Leader（即 `leader2`）；
     - `leader1` 应自动转为 Follower（不会干扰现有 Leader）。

  4. 制造无多数派分区

     断开 `leader2` 和另一个节点（共断开 2 个），只剩 1 个节点在线。

     此时**无 quorum（<2）**，应**无法选举新 Leader**。

     等待超过 `2 * ElectionTimeout`，确认**没有 Leader**（`checkNoLeader`）。

  5. 恢复多数派

     重新连接其中一个断开的节点，使在线节点数回到 2（形成 quorum）。

     验证：**成功选举出新 Leader**。

  6. 最后节点回归

     将最后一个断开的节点也重新连接。

     验证：集群**仍保持一个 Leader**，系统稳定。

* ManyElections

  1. 初始状态
     - 启动 7 个服务器，确认存在一个r Leader。
  2. 重复 9 次（`iters = 10`）以下操作
     - **随机断开 3 个节点**（可能包括当前 Leader，也可能不包括）；
     - 此时剩余 **4 个节点在线**（≥ 多数派，因为 7/2+1 = 4）；
     - 验证：**集群仍能选出且仅有一个 Leader**（要么原 Leader 未被断开，要么剩余 4 节点成功重选）；
     - **重新连接之前断开的 3 个节点**，恢复全连通；
     - （隐含：系统应快速收敛，无脑裂）
  3. 最终验证
     - 所有节点连通后，再次确认存在唯一 Leader。

## Part 3B：Log

### 测试项详述

* BasicAgree

  
