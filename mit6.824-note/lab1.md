# 思路
worker负责发送RPC请求到coordinator，coordinator负责分配任务到worker，worker执行完任务之后需要发送自己任务的状态，等待所有的worker的map任务执行完毕后，coordinator会将状态变成reduce阶段，然后将reduce任务下发到worker，所有的worker完成任务后，将自己的阶段变为完成，在每次worker发送请求的时候会将自己的上一次工作的状态，和自己的worker的索引发送到coodinator上

通过协程不断轮训所有的任务判断任务是否超时，通过协程回收任务
# 实现
## worker
worker的实现比较简单，就是不断发起RPC请求去调用coordinator申请任务，然后在本地执行任务
```go
func Worker(mapf func(string, string) []KeyValue, reducef func(string, []string) string) {
	workerId := strconv.Itoa(os.Getpid())
	log.Printf("Worker %s started \n", workerId)

	var LastTaskType string
	var LastTaskIndex int
	for {
		request := Request{WorkerId: workerId, LastTaskType: LastTaskType, LastTaskIndex: LastTaskIndex}
		response := Response{}
		log.Printf("Worker %s apply task \n", workerId)
		call("Coordinator.ApplyTask", &request, &response)
		if response.TaskType == "" {
			log.Printf("Received job finish singal from coordinator\n")
			break
		} else if response.TaskType == MAP {
			doMap(response, workerId, mapf)
		} else if response.TaskType == REDUCE {
			doReduce(response, workerId, reducef)
		} else {
			log.Printf("unsuppot task type %s", response.TaskType)
			break
		}
		LastTaskType = response.TaskType
		LastTaskIndex = response.TaskIndex
	}
}
```
- 需要保证map任务和reduce任务的原子性，避免异常情况导致文件受损，使得之后的任务无法执行下去的bug。具体就是先生成一个临时文件再利用系统调用`os.Rename`来完成原子替换操作，保证写文件的原子性
- mersort or hashSort：对于Reduce来说，输入是一堆文件，输出是一个文件，执行过程中在保证不OOM的情况下，不断把`<key, list(intermediate_value)>`喂给用户的reduce函数执行得到最终的`<kev,value>`对，然后写出到输出文件中

## Coordinator
Coordinator主要用来管理任务，在设计Coordinator要管理当前运行的阶段
```go
type Coordinator struct {
	lock sync.Mutex
	stage string
	nMap int
	nReduce int
	tasks map[string]Task
	availableTasks chan Task
}
type Task struct {
	// worker类型
	Type string
	// 索引
	Index int
	// Map输入文件地址
	MapInputFile string
	// workerId
	WorkerId string
	// 任务的结束时间
	Deadline time.Time
}
```