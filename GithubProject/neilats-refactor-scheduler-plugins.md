---
tags:
  - neilats_refactor
  - scheudling-framework
---

# Neilats代码与开发日志

> * GitHub项目链接
>
>   https://github.com/WyYong7240/neilats-refactor-scheduler-plugins

## 项目代码

> 1. neilats_refactor.go
>
>    neilats_refatctor自定义调度插件的主体实现部分，实现了Prefilter、Filter、Score三个scheduling-framework扩展点
>
> 2. get_prometheus_metrics.go
>
>    neilats_refactor自定义调度插件的工具文件，用于通过Prometheus提供的api，查询多个对应资源指标
>
> 3. latency_run_get.go
>
>    neilats_refactor自定义调度插件的工具文件，用于在开启SLA约束情况下，初始化主机之间的延迟测量、获取延迟数据文件等
>
> 4. build-rtt-matrix.go
>
>    neilats_refactor自定义调度插件的工具文件，用于在开启SLA约束情况下，利用获取的延迟数据文件，构建各个节点之间的延迟矩阵rtt-matrix
>
> 5. common.go
>
>    共有文件，包含了一些多个文件中使用到的公共变量
>
> 6. neilats_unit_test.go
>
>    neilats_refactor自定义调度插件的单元测试文件，包含两类测试，即SLA开启和关闭情况下的单元测试

### neilats_refactor.go

~~~GO
package neilats_refactor_go

import (
	"context"
	"fmt"
	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/klog/v2"
	"k8s.io/kubernetes/pkg/scheduler/framework"
	"math"
	"strconv"
)

const Name = "NeilatsRefactorScheduler"

// 声明自定义调度器结构体
type NeilatsRefactorScheduler struct {
	config NeilatsRefactorSchedulerArgs
	handle framework.Handle
	logger klog.Logger
}

// 声明要实现的扩展点， 同时检测自定义调度器是否实现了这两个扩展点
var _ framework.PreFilterPlugin = &NeilatsRefactorScheduler{}
var _ framework.FilterPlugin = &NeilatsRefactorScheduler{}
var _ framework.ScorePlugin = &NeilatsRefactorScheduler{}

// 声明自定义调度器的自定义参数结构体
// 调度器添加自定义参数的方法，参考了其他调度插件中的pkg/nodeResources的实现方法
type UserAddressSecretMap struct {
	NodeAddress string
	NodeSecret  string
}
type NeilatsRefactorSchedulerArgs struct {
	// 如果自定义调度器有参数，需要参数结构体添加该内联类型，是Kubernetes库的要求
	metav1.TypeMeta
	PrometheusAddress        string                          `json:"prometheusAddress"`
	NetworkDevice            map[string]string               `json:"networkDevice"`
	StorageDevice            map[string]string               `json:"storageDevice"`
	EnableSLA                bool                            `json:"enableSLA"`
	KubeNodeAddressAndSecret map[string]UserAddressSecretMap `json:"kubeNodeAddressAndSecret"`
}

// 如果自定义调度器有参数，需要参数结构体实现runtime.Object类型的接口
var _ runtime.Object = &NeilatsRefactorSchedulerArgs{}

// 需要实现DeepCopyObject、DeepCopyInto这两个接口， 是Kubernetes架构设计与代码生成工具的要求
// 该结构体要被设计为可以被序列化、反序列化、在组件之间传递，并且可以安全的拷贝
// 这两个方法也可以通过code-generator来自动生成
func (in *NeilatsRefactorSchedulerArgs) DeepCopyInto(out *NeilatsRefactorSchedulerArgs) {
	*out = *in
	out.TypeMeta = in.TypeMeta

	// 拷贝基本字段
	out.PrometheusAddress = in.PrometheusAddress
	out.EnableSLA = in.EnableSLA

	// 拷贝map[string]string
	if in.NetworkDevice != nil {
		out.NetworkDevice = make(map[string]string, len(in.NetworkDevice))
		for k, v := range in.NetworkDevice {
			out.NetworkDevice[k] = v
		}
	}
	if in.StorageDevice != nil {
		out.StorageDevice = make(map[string]string, len(in.StorageDevice))
		for k, v := range in.StorageDevice {
			out.StorageDevice[k] = v
		}
	}
	if in.KubeNodeAddressAndSecret != nil {
		out.KubeNodeAddressAndSecret = make(map[string]UserAddressSecretMap, len(in.KubeNodeAddressAndSecret))
		for k, v := range in.KubeNodeAddressAndSecret {
			out.KubeNodeAddressAndSecret[k] = UserAddressSecretMap{NodeAddress: v.NodeAddress, NodeSecret: v.NodeSecret}
		}
	}
	return
}

func (in *NeilatsRefactorSchedulerArgs) DeepCopy() *NeilatsRefactorSchedulerArgs {
	if in == nil {
		return nil
	}
	out := new(NeilatsRefactorSchedulerArgs)
	in.DeepCopyInto(out)
	return out
}

// 需要参数结构体实现DeepCopyObject接口
func (in *NeilatsRefactorSchedulerArgs) DeepCopyObject() runtime.Object {
	if c := in.DeepCopy(); c != nil {
		return c
	}
	return nil
}

// 返回插件名称
func (neilats *NeilatsRefactorScheduler) Name() string {
	return Name
}

// 新建并初始化一个新的插件并返回，将其注册到调度器中
func New(neilatsArgs runtime.Object, h framework.Handle) (framework.Plugin, error) {
	defaultConfig := NeilatsRefactorSchedulerArgs{
		PrometheusAddress: "http://192.168.3.226:31739", // 默认Prometheus地址
		NetworkDevice: map[string]string{ // 默认网络设备配置
			"master": "ens18",
			"node1":  "ens18",
			"node2":  "ens18",
		},
		StorageDevice: map[string]string{ // 默认存储设备配置
			"master": "/dev/mapper/ubuntu--vg-ubuntu--lv",
			"node1":  "/dev/mapper/ubuntu--vg-ubuntu--lv",
			"node2":  "/dev/mapper/ubuntu--vg-ubuntu--lv",
		},
		EnableSLA: false, // 默认不开启SLA约束要求
		// 键为Kubernetes中的节点名称
		KubeNodeAddressAndSecret: map[string]UserAddressSecretMap{
			"master": {"192.168.3.226", "mobisys912"},
			"node1":  {"192.168.3.229", "mobisys912"},
			"node2":  {"192.168.3.224", "mobisys912"},
		}, // 由于默认不开启SLA约束，所以不要求kubeNodeAddressAndSecret参数
	}

	// 如果参数neilatsArgs不为空，则尝试解析其中的配置参数
	if neilatsArgs != nil {
		// 将runtime.Object类型的参数，转换为调度器所需的参数类型
		args, ok := neilatsArgs.(*NeilatsRefactorSchedulerArgs)
		if !ok {
			return nil, fmt.Errorf("want args to be of type NeilatsRefactorSchedulerArgs, got %T", neilatsArgs)
		}
		// 如果各个参数存在，将defaultConfig 中的各项参数改为传入的参数，否则使用默认参数
		if args.PrometheusAddress != "" {
			defaultConfig.PrometheusAddress = args.PrometheusAddress
		}
		if len(args.NetworkDevice) > 0 {
			defaultConfig.NetworkDevice = args.NetworkDevice
		}
		if len(args.StorageDevice) > 0 {
			defaultConfig.StorageDevice = args.StorageDevice
		}
		defaultConfig.EnableSLA = args.EnableSLA
		if args.EnableSLA {
			if len(args.KubeNodeAddressAndSecret) == 0 {
				return nil, fmt.Errorf("when EnableSLA is True, KubeNodeAddressAndSecret must have content")
			}
			defaultConfig.KubeNodeAddressAndSecret = args.KubeNodeAddressAndSecret

			// 启动调度器的延迟测试初始化
			//if err := InitNodeLatencyTestTool(defaultConfig.KubeNodeAddressAndSecret); err != nil {
			//	return nil, fmt.Errorf("enable SLA init Node Latency Test Tool Failed:%v", err)
			//}
			// 启动各个节点之间的延迟测试
			//if err := StartLatencyTest(defaultConfig.KubeNodeAddressAndSecret); err != nil {
			//	return nil, fmt.Errorf("enable SLA Start Node Latency Test Tool Failed:%v", err)
			//}

			// 此处存在一个问题，子程序与主程序关系紧密（指子程序的启动与主程序相关）又疏远（子程序仅仅帮助主程序生成与修改文件）那么这种情况应该使用goroutine线程实现还是多进程实现？
			// 此处产生一个设想，能否在仅仅调度具有SLA需求的Pod时，才下载一次远程主机上的延迟测试文件，才进行一轮rttMatrix矩阵文件的构建
			// 但是让远程主机上的延迟测试一直运行，这样不仅能够省去rttMatrix文件，还可以省去多线程、文件锁等复杂操作
			// 开始不断间隔下载延迟测试结果文件
			//go GetLatencyTestResultInterval(defaultConfig.KubeNodeAddressAndSecret)
			// 开始构建rttMatrix文件
			//go BuildRtt(defaultConfig.KubeNodeAddressAndSecret)
		}
	}

	// 这里返回我们创建并初始化完成的自定义调度器
	return &NeilatsRefactorScheduler{
		handle: h,
		config: defaultConfig,
		logger: klog.FromContext(context.Background()),
	}, nil
}

// 用于在开启了SLA的情况下，过滤掉没有nei_node和SLA约束要求的Pod
func (neilats *NeilatsRefactorScheduler) PreFilter(ctx context.Context, state *framework.CycleState, p *v1.Pod) (*framework.PreFilterResult, *framework.Status) {
	// 仅当开启了SLA约束要求配置开始检查Pod是否存在有效标签
	if neilats.config.EnableSLA {
		neiNodeLabel := "nei_node"
		SLAConstraintLabel := "sla"

		// 检查Pod是否存在邻近节点标签
		neiNodeValue, neiNodeExist := p.GetLabels()[neiNodeLabel]
		// 检查Pod是否存在SLA约束要求标签
		slaConstraintValue, slaExist := p.GetLabels()[SLAConstraintLabel]

		// 如果两个标签同时都不存在，那么就是个普通Pod，不用被过滤
		if !neiNodeExist && !slaExist {
			return nil, framework.NewStatus(framework.Success, fmt.Sprintf("Pod %s haven't NeiNodeLable And SLAConstraintLable, pass Prefilter", p.Name))
			// 如果两个标签都存在，那么就是一个具有SLA要求的Pod，开始检查连个标签值有效性
		} else if neiNodeExist && slaExist {
			// 检查Pod指定的邻近节点是否在指定的需要具有SLA要求的集群节点中
			if _, exist := neilats.config.KubeNodeAddressAndSecret[neiNodeValue]; !exist {
				return nil, framework.NewStatus(framework.Unschedulable, fmt.Sprintf("Pod %s NeiNodeValue %s not exist in Kubernetes Cluster", p.Name, neiNodeValue))
			}
			// 检查Pod指定的SLAConstraint标签值能否转换为Float64类型
			if _, err := strconv.ParseFloat(slaConstraintValue, 64); err != nil {
				return nil, framework.NewStatus(framework.Unschedulable, fmt.Sprintf("Pod %s SLAConstraintLable Value %s can't convert to Float64", p.Name, slaConstraintValue))
			}
			return nil, framework.NewStatus(framework.Success, fmt.Sprintf("Pod %s NeiNodeLable %s And SLAConstraintLabel %q, pass Prefilter", p.Name, neiNodeValue, slaConstraintValue))
		} else {
			// 如果两个标签只存在一个，那么这是无效的情况，需要被过滤掉
			return nil, framework.NewStatus(framework.Unschedulable, fmt.Sprintf("Pod %s haven't NeiNodeLable: nei_node Or SLAConstraintLable: sla", p.Name))
		}
	}
	return nil, framework.NewStatus(framework.Success, fmt.Sprintf("Enable SLA: %t, all pod pass Prefilter", neilats.config.EnableSLA))
}

func (neilats *NeilatsRefactorScheduler) PreFilterExtensions() framework.PreFilterExtensions {
	return neilats
}

// AddPod和RemovePod属于PreFilterExtension的实现部分，用于模拟Pod被正式调度之前，Pod被添加到节点和从节点上删除的操作；
// 主要用于统计Pod的资源使用量，用于增量更新资源使用量
// 由于此处实现PreFilter接口只是用于检查Pod是否具有某个标签，不涉及Pod的资源使用量问题，可以简单实现AddPod和RemovePod
func (neilats *NeilatsRefactorScheduler) AddPod(ctx context.Context, cycleState *framework.CycleState, podToSchedule *v1.Pod, podToAdd *framework.PodInfo, nodeInfo *framework.NodeInfo) *framework.Status {
	return framework.NewStatus(framework.Success, "")
}

func (neilats *NeilatsRefactorScheduler) RemovePod(ctx context.Context, cycleState *framework.CycleState, podToSchedule *v1.Pod, podToRemove *framework.PodInfo, nodeInfo *framework.NodeInfo) *framework.Status {
	return framework.NewStatus(framework.Success, "")
}

func (neilats *NeilatsRefactorScheduler) Filter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	// 检查Pod是否存在邻近节点标签
	neiNodeValue, neiNodeExist := pod.GetLabels()["nei_node"]
	// 检查Pod是否存在SLA约束要求标签
	slaConstraintValue, slaExist := pod.GetLabels()["sla"]

	if neilats.config.EnableSLA && neiNodeExist && slaExist {
		// 获取Pod指定的邻近节点与SLA要求
		SLAConstraintValue, err := strconv.ParseFloat(slaConstraintValue, 64)
		if err != nil {
			return framework.NewStatus(framework.UnschedulableAndUnresolvable, fmt.Sprintf("Failed Convert SLAConstraint Str to Float64:%v", err))
		}

		// 进行一次延迟测试文件的下载
		if err := DownloadLatencyTestResult(neilats.config.KubeNodeAddressAndSecret); err != nil {
			return framework.NewStatus(framework.UnschedulableAndUnresolvable, fmt.Sprintf("Failed Download Latency Test Result:%v", err))
		}

		// 构建一次RttMatrix
		rttMatrix, err := BuildRttWithOutFile(neilats.config.KubeNodeAddressAndSecret)
		if err != nil {
			return framework.NewStatus(framework.UnschedulableAndUnresolvable, fmt.Sprintf("Failed Build RttMatrix:%v", err))
		}

		// 根据RttMatrix，筛选满足Pod的SLA要求的Pod
		if rttMatrix[nodeInfo.Node().Name][neiNodeValue] <= SLAConstraintValue {
			return framework.NewStatus(framework.Success, "Node "+nodeInfo.Node().Name+" rtt Satisfy Pod "+pod.Name+" SLA Constraint")
		} else {
			return framework.NewStatus(framework.Unschedulable, "Node "+nodeInfo.Node().Name+" rtt UnSatisfy Pod "+pod.Name+" SLA Constraint")
		}
	}
	// 如果没有开启SLA，则所有节点通过过滤
	return framework.NewStatus(framework.Success, "Node:"+nodeInfo.Node().Name)
}

func (neilats *NeilatsRefactorScheduler) Score(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) (int64, *framework.Status) {
	LBScore, err := neilats.LBScore(p, nodeName)
	if err != nil {
		return 0, framework.NewStatus(framework.Error, err.Error())
	}
	return LBScore, nil
}

// 计算节点的负载均衡得分
func (neilats *NeilatsRefactorScheduler) LBScore(p *v1.Pod, nodeName string) (int64, error) {
	// 1. 计算Pod部署在节点上后，CPU的实际利用率
	podCpuRequest := float64(0)
	if p.Spec.Containers[0].Resources.Requests != nil {
		// 一个Pod的CPU资源请求，包含其中所有container的CPU资源请求之和
		for _, container := range p.Spec.Containers {
			if container.Resources.Requests.Cpu() != nil {
				// .MilliValue() 方法返回以毫秒为单位的CPU值，及CPU的毫核
				podCpuRequest += float64(container.Resources.Requests.Cpu().MilliValue())
			}
		}
	}
	nodeCpuCapacity, err := GetNodeTotalCpu(neilats.config.PrometheusAddress, nodeName)
	if err != nil {
		return 0, fmt.Errorf("get node %s cpu capacity failed: %v", nodeName, err)
	}
	NodeCpuIdleRate, err := GetNodeCpuIdleRate(neilats.config.PrometheusAddress, nodeName)
	if err != nil {
		return 0, fmt.Errorf("get node %s cpu idle rate failed: %v", nodeName, err)
	}
	nodeCpuUsed := (1 - NodeCpuIdleRate) * nodeCpuCapacity
	realNodeCpuUseRate := (nodeCpuUsed + podCpuRequest) / nodeCpuCapacity

	// 2. 计算Pod部署在节点上后，内存的实际利用率
	podMemRequest := float64(0)
	if p.Spec.Containers[0].Resources.Requests != nil {
		for _, container := range p.Spec.Containers {
			if container.Resources.Requests.Memory() != nil {
				// .Value() 方法返回以字节为单位的内存值
				podMemRequest += float64(container.Resources.Requests.Memory().Value()) / 1024 / 1024
			}
		}
	}
	nodeMemCapacity, err := GetNodeTotalMemory(neilats.config.PrometheusAddress, nodeName)
	if err != nil {
		return 0, fmt.Errorf("get node %s mem capacity failed: %v", nodeName, err)
	}
	NodeMemAvailableRate, err := GetNodeMemoryAvailableRate(neilats.config.PrometheusAddress, nodeName)
	if err != nil {
		return 0, fmt.Errorf("get node %s mem available rate failed: %v", nodeName, err)
	}
	nodeMemUsed := (1 - NodeMemAvailableRate) * nodeMemCapacity
	realNodeMemUseRate := (nodeMemUsed + podMemRequest) / nodeMemCapacity

	// 3. 计算Pod部署在节点上后，网络的实际利用率
	podNetworkRequest := float64(0)
	if value, exist := p.GetLabels()["network-request"]; exist { // 如果Pod的标签中有"network-request"字段，则取出其值作为网络请求值，其单位为字节/s
		podNetworkRequest, _ = strconv.ParseFloat(value, 64)
	}
	nodeNetworkCapacity := GetNodeTotalNetwork()
	NodeNetworkAvailableRate, err := GetNodeNetworkAvailableRate(neilats.config.PrometheusAddress, nodeName, neilats.config.NetworkDevice[nodeName])
	if err != nil {
		return 0, fmt.Errorf("get node %s network available rate failed: %v", nodeName, err)
	}
	nodeNetworkUsed := (1 - NodeNetworkAvailableRate) * nodeNetworkCapacity
	realNodeNetworkUseRate := (nodeNetworkUsed + podNetworkRequest) / nodeNetworkCapacity

	// 4. 计算Pod部署在节点上后，磁盘的实际利用率
	podDiskRequest := float64(0)
	// 这里其实本意想如果Pod没有要求disk资源的话，那么就将所有container的镜像大小作为disk请求值，如果有那就加上所有container的镜像大小之和
	if value, exist := p.GetLabels()["disk-request"]; exist { // 如果Pod的标签中有"disk-request"字段，则取出其值作为磁盘请求值，其单位为MB
		podDiskRequest, _ = strconv.ParseFloat(value, 64)
	}
	nodeDiskCapacity, err := GetNodeTotalDisk(neilats.config.PrometheusAddress, nodeName, neilats.config.StorageDevice[nodeName])
	if err != nil {
		return 0, fmt.Errorf("get node %s disk capacity failed: %v", nodeName, err)
	}
	NodeDiskAvailableRate, err := GetNodeDiskAvailableRate(neilats.config.PrometheusAddress, nodeName, neilats.config.StorageDevice[nodeName])
	if err != nil {
		return 0, fmt.Errorf("get node %s disk available rate failed: %v", nodeName, err)
	}
	nodeDiskUsed := (1 - NodeDiskAvailableRate) * nodeDiskCapacity
	realNodeDiskUseRate := (nodeDiskUsed + podDiskRequest) / nodeDiskCapacity

	// 5. 计算四资源的均值
	avgScore := (realNodeCpuUseRate + realNodeMemUseRate + realNodeNetworkUseRate + realNodeDiskUseRate) / 4
	// 6. 计算四资源的方差
	variance := math.Pow(realNodeCpuUseRate-avgScore, 2) + math.Pow(realNodeMemUseRate-avgScore, 2) + math.Pow(realNodeNetworkUseRate-avgScore, 2) + math.Pow(realNodeDiskUseRate-avgScore, 2)
	// 7. 计算最终负载均衡得分
	LBScore := int64(100 - 100*variance)
	fmt.Println("Node Name: ", nodeName, "LB Score: ", LBScore)

	return LBScore, nil
}

func (neilats *NeilatsRefactorScheduler) ScoreExtensions() framework.ScoreExtensions {
	return neilats
}

func (neilats *NeilatsRefactorScheduler) NormalizeScore(ctx context.Context, state *framework.CycleState, p *v1.Pod, scores framework.NodeScoreList) *framework.Status {
	// 默认的排序方法
	//return helper.DefaultNormalizeScore(framework.MaxNodeScore, false, scores)

	// 将原始得分线性映射到[MinNodeScore, MaxNodeScore]，此处即0-100的范围内
	// 找出原始的分数最小值和最大值
	var maxScore int64 = -math.MaxInt64
	var minScore int64 = math.MaxInt64
	for _, nodeScore := range scores {
		if nodeScore.Score > maxScore {
			maxScore = nodeScore.Score
		}
		if nodeScore.Score < minScore {
			minScore = nodeScore.Score
		}
	}

	// 将原始得分映射到新的区间
	oldRange := maxScore - minScore
	newRange := framework.MaxNodeScore - framework.MinNodeScore
	for i, nodeScore := range scores {
		if oldRange == 0 {
			scores[i].Score = framework.MinNodeScore
		} else {
			scores[i].Score = ((nodeScore.Score - minScore) / oldRange * newRange) + framework.MinNodeScore
		}
	}
	return nil
}
~~~



### get_prometheus_metrics.go

~~~GO
package neilats_refactor_go

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
	"strconv"
)

// 对于Prometheus的/api/v1/query端点，响应结构体统一的，这个结构体适用于所有PRomeQL查询的响应
type PrometheusQueryResponse struct {
	Status string `json:"status"`
	Data   struct {
		ResultType string `json:"resultType"`
		Result     []struct {
			Metric map[string]string `json:"metric"`
			Value  []interface{}     `json:"value"`
		}
	}
}

// 获取Prometheus的查询结果
func queryPrometheus(promURL string, promQL string) (*PrometheusQueryResponse, error) {
	// 构造请求URL
	u, _ := url.Parse(promURL)
	u.Path = "/api/v1/query"
	params := url.Values{"query": []string{promQL}}
	u.RawQuery = params.Encode()

	// 发起请求
	resp, err := http.Get(u.String())
	if err != nil {
		fmt.Println("Error querying Prometheus:", err)
		return nil, err
	}
	defer resp.Body.Close()

	// 解析响应
	var result PrometheusQueryResponse
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		fmt.Println("Error decoding Prometheus response:", err)
		return nil, err
	}

	if result.Status != "success" {
		return nil, fmt.Errorf("Prometheus query failed: %s", result.Status)
	}
	return &result, nil
}

// GetNodeCpuIdleRate 获取指定节点的CPU空闲率
func GetNodeCpuIdleRate(prometheusAddress, nodeName string) (float64, error) {
	// 构造查询语句
	promQL := fmt.Sprintf("sum(increase(node_cpu_seconds_total{mode='idle',instance='%s'}[1m]))/sum(increase(node_cpu_seconds_total{instance='%s'}[1m]))", nodeName, nodeName)
	result, err := queryPrometheus(
		prometheusAddress,
		promQL,
	)
	if err != nil {
		fmt.Println("Error querying Prometheus Cpu idle rate:", err)
		return 0, fmt.Errorf("error querying Prometheus Cpu idle rate: %s", err)
	}
	CpuIdleRate, _ := strconv.ParseFloat(result.Data.Result[0].Value[1].(string), 64)
	fmt.Println("CPU idle rate of node", nodeName, "is", CpuIdleRate)
	return CpuIdleRate, nil
}

func GetNodeMemoryAvailableRate(prometheusAddress, nodeName string) (float64, error) {
	// 构造查询语句
	promQL := fmt.Sprintf("node_memory_MemAvailable_bytes{instance='%s'}/node_memory_MemTotal_bytes{instance='%s'}", nodeName, nodeName)
	result, err := queryPrometheus(
		prometheusAddress,
		promQL,
	)
	if err != nil {
		fmt.Println("Error querying Prometheus Memory available rate:", err)
		return 0, fmt.Errorf("error querying Prometheus Memory available rate: %s", err)
	}
	MemoryAvailableRate, _ := strconv.ParseFloat(result.Data.Result[0].Value[1].(string), 64)
	fmt.Println("Memory available of node", nodeName, "is", MemoryAvailableRate)
	return MemoryAvailableRate, nil
}

func GetNodeNetworkAvailableRate(prometheusAddress, nodeName, networkDevice string) (float64, error) {
	const totalNetworkTraffic = 1000000 // 网络接口的最大传输速度经过测压得出，最大为100 0000字节/s
	receiveRatePromql := fmt.Sprintf("rate(node_network_receive_bytes_total{instance='%s',device='%s'}[1m])", nodeName, networkDevice)
	transmitRatePromql := fmt.Sprintf("rate(node_network_transmit_bytes_total{instance='%s',device='%s'}[1m])", nodeName, networkDevice)
	receiveRateResult, err := queryPrometheus(
		prometheusAddress,
		receiveRatePromql,
	)
	if err != nil {
		fmt.Println("Error querying Prometheus network receive rate:", err)
		return 0, fmt.Errorf("error querying Prometheus network receive rate: %s", err)
	}

	transmitRateResult, err := queryPrometheus(
		prometheusAddress,
		transmitRatePromql,
	)
	if err != nil {
		fmt.Println("Error querying Prometheus network transmit rate:", err)
		return 0, fmt.Errorf("error querying Prometheus network transmit rate: %s", err)
	}
	// 分别获取每秒接受和发送的字节数
	receiveBytes, _ := strconv.ParseFloat(receiveRateResult.Data.Result[0].Value[1].(string), 64)
	transmitBytes, _ := strconv.ParseFloat(transmitRateResult.Data.Result[0].Value[1].(string), 64)
	// 以该节点的接收速率和发送速率均值作为该节点的网络利用率
	networkAvailableRate := (totalNetworkTraffic*2 - receiveBytes - transmitBytes) / totalNetworkTraffic * 2
	fmt.Println("Network available of node", nodeName, "is", networkAvailableRate)
	return networkAvailableRate, nil
}

func GetNodeDiskAvailableRate(prometheusAddress, nodeName, storageDevice string) (float64, error) {
	promQL := fmt.Sprintf("((node_filesystem_free_bytes{fstype!='',device='%s',instance='%s'} / node_filesystem_size_bytes{fstype!='',device='%s',instance='%s'})) * 100", storageDevice, nodeName, storageDevice, nodeName)
	result, err := queryPrometheus(
		prometheusAddress,
		promQL,
	)
	if err != nil {
		fmt.Println("Error querying Prometheus disk available rate:", err)
		return 0, fmt.Errorf("error querying Prometheus disk available rate: %s", err)
	}
	DiskAvailableRate, _ := strconv.ParseFloat(result.Data.Result[0].Value[1].(string), 64)
	fmt.Println("Disk available of node", nodeName, "is", DiskAvailableRate)
	return DiskAvailableRate, nil
}

// GetNodeTotalCpu 获取指定节点的总CPU核数,例如宿主机使用的是8核心，那么每秒提供的CPU总量是 8 * 1000 = 8000毫核
func GetNodeTotalCpu(prometheusAddress, nodeName string) (float64, error) {
	promQL := fmt.Sprintf("count(node_cpu_seconds_total{mode='user',instance='%s'})", nodeName)
	result, err := queryPrometheus(
		prometheusAddress,
		promQL,
	)
	if err != nil {
		fmt.Println("Error querying Prometheus total CPU:", err)
		return 0, fmt.Errorf("error querying Prometheus total CPU: %s", err)
	}
	totalCpu, _ := strconv.ParseFloat(result.Data.Result[0].Value[1].(string), 64)
	totalCpu = totalCpu * 1000 // 单位转换为毫核
	fmt.Println("Total CPU of node", nodeName, "is", totalCpu)
	return totalCpu, nil
}

func GetNodeTotalMemory(prometheusAddress, nodeName string) (float64, error) {
	promQL := fmt.Sprintf("node_memory_MemTotal_bytes{instance='%s'}", nodeName)
	result, err := queryPrometheus(
		prometheusAddress,
		promQL,
	)
	if err != nil {
		fmt.Println("Error querying Prometheus total memory:", err)
		return 0, fmt.Errorf("error querying Prometheus total memory: %s", err)
	}
	totalMemory, _ := strconv.ParseFloat(result.Data.Result[0].Value[1].(string), 64)
	totalMemory = totalMemory / (1024 * 1024) // 单位转换为兆字节
	fmt.Println("Total memory of node", nodeName, "is", totalMemory, "MB")
	return totalMemory, nil
}

func GetNodeTotalNetwork() float64 {
	return 1000000 // 网络接口的最大传输速度经过测压得出，最大为100 0000字节/s
}

func GetNodeTotalDisk(prometheusAddress, nodeName, storageDevice string) (float64, error) {
	promQL := fmt.Sprintf("node_filesystem_size_bytes{fstype!='',device='%s',instance='%s'}", storageDevice, nodeName)
	result, err := queryPrometheus(
		prometheusAddress,
		promQL,
	)
	if err != nil {
		fmt.Println("Error querying Prometheus total disk:", err)
		return 0, fmt.Errorf("error querying Prometheus total disk: %s", err)
	}
	totalDisk, _ := strconv.ParseFloat(result.Data.Result[0].Value[1].(string), 64)
	totalDisk = totalDisk / (1024 * 1024) // 单位转换为MB
	fmt.Println("Total disk of node", nodeName, "is", totalDisk, "MB")
	return totalDisk, nil
}
~~~



### latency_run_get.go

~~~GO
package neilats_refactor_go

import (
	"fmt"
	"github.com/pkg/sftp"
	"golang.org/x/crypto/ssh"
	"io"
	"log"
	"os"
	"time"
)

func getSshClient(nodeName, nodeAddress, nodeSecret string) (*ssh.Client, error) {
	// 配置SSH客户端配置
	config := &ssh.ClientConfig{
		// 默认以root登录各个节点
		User: "root",
		Auth: []ssh.AuthMethod{
			// 设置SSH密码
			ssh.Password(nodeSecret),
		},
		HostKeyCallback: ssh.InsecureIgnoreHostKey(), // 这里为了简便忽略了host key检查，在生产环境中不推荐这样做
	}

	// 连接到远程服务器
	client, err := ssh.Dial("tcp", nodeAddress+":22", config)
	if err != nil {
		return nil, fmt.Errorf("failed to login Node %q: %v", nodeName, err)
	}
	return client, nil
}

func getFileOnRemote(nodeName, nodeAddress, nodeSecret, remotePath, localPath string) error {
	// 连接到远程服务器
	client, err := getSshClient(nodeName, nodeAddress, nodeSecret)
	if err != nil {
		return fmt.Errorf("failed to login Node %q: %v", nodeName, err)
	}
	defer client.Close()

	// 创建SFTP客户端
	sftpClient, err := sftp.NewClient(client)
	if err != nil {
		return fmt.Errorf("after Login Node %q, Failed Create Sftp Client: %v", nodeName, err)
	}
	defer sftpClient.Close()

	// 打开远程主机上的文件
	remoteFile, err := sftpClient.Open(remotePath)
	if err != nil {
		return fmt.Errorf("after Create Sftp Client on Node %q, Failed Open remoteFile:%v", nodeName, err)
	}
	defer remoteFile.Close()

	// 创建本地文件
	localFile, err := os.Create(localPath)
	if err != nil {
		return fmt.Errorf("failed Create Local file:%v", err)
	}
	defer localFile.Close()

	// 将远程主机上的文件内容复制到本地文件中
	if _, err := io.Copy(localFile, remoteFile); err != nil {
		return fmt.Errorf("failed Copy Remote file to Local:%v", err)
	}
	log.Printf("Down Load Node %q File Success!", nodeName)
	return nil
}

func writeFileOnRemote(nodeName, nodeAddress, nodeSecret, remotePath, content string) error {
	// 连接到远程服务器
	client, err := getSshClient(nodeName, nodeAddress, nodeSecret)
	if err != nil {
		return fmt.Errorf("failed to login Node %q: %v", nodeName, err)
	}
	defer client.Close()

	// 创建SFTP客户端
	sftpClient, err := sftp.NewClient(client)
	if err != nil {
		return fmt.Errorf("after Login Node %q, Failed Create Sftp Client: %v", nodeName, err)
	}
	defer sftpClient.Close()

	// 在远程主机创建文件
	file, err := sftpClient.Create(remotePath)
	if err != nil {
		return fmt.Errorf("after Create SftpClient on Node %q, Failed Create File: %v", nodeName, err)
	}

	// 向创建的文件中写入内容
	if _, err := file.Write([]byte(content)); err != nil {
		return fmt.Errorf("after Create File on Node %q, Failed Write File: %v", nodeName, err)
	}

	// 将创建的文件设置为可执行类型
	if err := sftpClient.Chmod(remotePath, 0755); err != nil {
		return fmt.Errorf("after Write File on Node %q, Failed Chmod File: %v", nodeName, err)
	}
	return nil
}

func executeCommandOnRemote(nodeName, nodeAddress, nodeSecret, command string) error {
	// 连接到远程服务器
	conn, err := getSshClient(nodeName, nodeAddress, nodeSecret)
	if err != nil {
		return fmt.Errorf("Failed to login Node %q: %v", nodeName, err)
	}
	defer conn.Close()

	// 创建一个新的会话
	session, err := conn.NewSession()
	if err != nil {
		return fmt.Errorf("After Login Node %q, Failed to Create Session: %v", nodeName, err)
	}
	defer session.Close()

	// 设置输出流
	//session.Stdout = bufio.NewWriter(log.Writer())
	//session.Stderr = bufio.NewWriter(log.Writer())

	// 在远程服务器上执行命令
	if err := session.Run(command); err != nil {
		return fmt.Errorf("After Create Session, Failed to Execute command")
	}

	fmt.Println("Command executed successfully")
	return nil
}

func InitNodeLatencyTestTool(KubeNodeAddressAndSecret map[string]UserAddressSecretMap) error {
	for k, v := range KubeNodeAddressAndSecret {
		fmt.Printf("Init Node %q Latency Test Tool\n", k)
		// 先下载测试工具
		command := "apt-get install fping -y"
		if err := executeCommandOnRemote(k, v.NodeAddress, v.NodeSecret, command); err != nil {
			return fmt.Errorf("node %q install Latency Test Tool Failed", k)
		}
		fmt.Printf("Install Latency Test Tool Success")

		// 创建测试目录并写入延迟测试脚本
		var remoteDirectoryPath string = "/root/ws/network-latency-test"
		createDirectoryCommand := "mkdir -p " + remoteDirectoryPath
		if err := executeCommandOnRemote(k, v.NodeAddress, v.NodeSecret, createDirectoryCommand); err != nil {
			return fmt.Errorf("node %q install Latency Test Tool Failed", k)
		}
		fmt.Printf("Node %q Create Latency Test Directory Success", k)

		// 定义当前节点要发送的延迟测试目标IP地址，去除当前主机的IP
		var ips = "ips=("
		for node, value := range KubeNodeAddressAndSecret {
			if node != k {
				ips = ips + "\"" + value.NodeAddress + "\" "
			}
		}
		ips = ips + ")\n"
		var scriptContent = `
# 定义存储数据的文件路径

while true; do
    # 获取当前时间戳，格式为YYYY-MM-DD HH:MM:SS
    timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    for ip in "${ips[@]}"; do
        output_file="latency_results_${ip}.txt"
        # 调用fping命令，-c 1表示只发送一个数据包
        result=$(fping -e $ip 2>/dev/null)
        # 将时间戳和fping结果追加到文件中
        echo "$timestamp, $result" >> $output_file
    done
    # 每隔一段时间（这里是1秒）执行一次，可按需调整
    sleep 10
done
`
		// 生成延迟测量脚本文件内容
		content := "#!/bin/bash\n" + ips + scriptContent
		// 将脚本写入远程主机中
		if err := writeFileOnRemote(k, v.NodeAddress, v.NodeSecret, remoteDirectoryPath+"/latency_test.sh", content); err != nil {
			return fmt.Errorf("write script file to remote node %q failed: %v", k, err)
		}
	}
	return nil
}

func StartLatencyTest(KubeNodeAddressAndSecret map[string]UserAddressSecretMap) error {
	for k, v := range KubeNodeAddressAndSecret {
		command := "cd /root/ws/network-latency-test && screen -dmS latency_test ./latency_test.sh"
		if err := executeCommandOnRemote(k, v.NodeAddress, v.NodeSecret, command); err != nil {
			return fmt.Errorf("start Node %q Latency Test Failed: %v", k, err)
		}
		log.Printf("Success Start Node %q Latency Test!", k)
	}
	return nil
}

func StopLatencyTest(KubeNodeAddressAndSecret map[string]UserAddressSecretMap) error {
	for k, v := range KubeNodeAddressAndSecret {
		command := "pkill -f latency_test.sh"
		if err := executeCommandOnRemote(k, v.NodeAddress, v.NodeSecret, command); err != nil {
			return fmt.Errorf("start Node %q Latency Test Failed: %v", k, err)
		}
		log.Printf("Success Stop Node %q Latency Test!", k)
	}
	return nil
}

func DownloadLatencyTestResult(KubeNodeAddressAndSecret map[string]UserAddressSecretMap) error {
	localPath := "latency/"
	remoteDirectoryPath := "/root/ws/network-latency-test/"
	for k, v := range KubeNodeAddressAndSecret {
		for nodeName, value := range KubeNodeAddressAndSecret {
			if nodeName != k {
				localFileName := fmt.Sprintf("%s2%s", k, nodeName)
				remoteFileName := fmt.Sprintf("latency_results_%s.txt", value.NodeAddress)
				if err := getFileOnRemote(k, v.NodeAddress, v.NodeSecret, remoteDirectoryPath+remoteFileName, localPath+localFileName); err != nil {
					return fmt.Errorf("failed get Latency Test On Node %q", k)
				}
			}
		}
	}
	return nil
}

func GetLatencyTestResultInterval(KubeNodeAddressAndSecret map[string]UserAddressSecretMap) error {
	for true {
		time.Sleep(10)
		LatencyFileLock.Lock()
		if err := DownloadLatencyTestResult(KubeNodeAddressAndSecret); err != nil {
			return fmt.Errorf("failed Get Latency Test Result Interval:%v", err)
		}
	}
	return nil
}
~~~



### build-rtt-matrix.go

~~~GO
package neilats_refactor_go

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
	"time"
)

const MEASUREMENT_TIME = 10
const LAST_N_LINES = 10

func BuildRttWithOutFile(KubeNodeAddressAndSecret map[string]UserAddressSecretMap) (map[string]map[string]float64, error) {
	nodeNum := len(KubeNodeAddressAndSecret)

	// 初始化rtt矩阵
	rttMatrix := make(map[string]map[string]float64, nodeNum)
	for k, _ := range KubeNodeAddressAndSecret {
		rttMatrix[k] = make(map[string]float64, nodeNum)
	}

	for fromNode, _ := range KubeNodeAddressAndSecret {
		for toNode, _ := range KubeNodeAddressAndSecret {
			if fromNode == toNode {
				rttMatrix[fromNode][toNode] = 0.0
			} else {
				value, err := getMeanOfLatencyFromTxt(fromNode, toNode)
				if err != nil {
					return nil, fmt.Errorf("get MeanOfLatencyFromTxt Failed:%v", err)
				} else if value >= 99 {
					rttMatrix[fromNode][toNode] = rttMatrix[toNode][fromNode]
				} else {
					rttMatrix[fromNode][toNode] = value
				}
			}
		}
	}

	// rttMatrix构建完成，返回给调度插件
	return rttMatrix, nil
}

func BuildRtt(KubeNodeAddressAndSecret map[string]UserAddressSecretMap) error {
	resultFile := "./latency/rtt_matrix.txt"
	timeOut := MEASUREMENT_TIME
	nodeNum := len(KubeNodeAddressAndSecret)

	// 获取需要构建RTT矩阵的节点名称
	nodes := make([]string, nodeNum)
	for k, _ := range KubeNodeAddressAndSecret {
		nodes = append(nodes, k)
	}

	// 初始化rtt矩阵
	rttMatrix := make([][]float64, nodeNum)
	for i := range rttMatrix {
		rttMatrix[i] = make([]float64, nodeNum)
	}

	for true {
		if timeOut > 0 {
			timeOut -= 1
			time.Sleep(1)
		} else {
			// 开始刷新RTT矩阵
			for i := range rttMatrix {
				for j := range rttMatrix {
					if i == j {
						rttMatrix[i][j] = 0.0
					} else {
						// 获取延迟结果文件的文件锁
						LatencyFileLock.Lock()
						value, err := getMeanOfLatencyFromTxt(nodes[i], nodes[j])
						// 延迟结果文件锁解锁
						LatencyFileLock.Unlock()
						if err != nil {
							return fmt.Errorf("get MeanOfLatencyFromTxt Failed:%v", err)
						} else if value >= 99 {
							rttMatrix[i][j] = rttMatrix[j][i]
						} else {
							rttMatrix[i][j] = value
						}
					}
				}
			}
			// 要写回RTT矩阵了,在获取rttMatrix文件锁之前，先构造好rttMatrix矩阵的新内容
			// 遍历二维数组并构建rttMatrix文件内容
			var lines []string
			for _, row := range rttMatrix {
				var line []string
				for _, value := range row {
					// 将所有值字符串化
					line = append(line, fmt.Sprintf("%f", value))
				}
				// 将一行中所有值变为一个字符串，中间用","间隔
				lines = append(lines, strings.Join(line, ","))
			}
			// 将所有行变为一个字符串，中间用换行符间隔
			content := strings.Join(lines, "\n")
			// 获取RTT矩阵文件的文件锁
			RttMatrixFileLock.Lock()
			// 由于后面的操作都涉及到rttMatrix文件的写入，所以推迟文件锁的解锁操作到函数结束
			defer RttMatrixFileLock.Unlock()
			rttFile, err := os.OpenFile(resultFile, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0644)
			if err != nil {
				return fmt.Errorf("open RTT File Failed!:%v", err)
			}
			defer rttFile.Close()

			// 写入rttMatrix
			_, err = rttFile.WriteString(content)
			if err != nil {
				return fmt.Errorf("failed to write to rtt-matrix file:%v", err)
			}
		}
	}
	return nil
}

func getMeanOfLatencyFromTxt(nodeFrom, nodeTo string) (float64, error) {
	resultFilePath := fmt.Sprintf("./latency/%s2%s", nodeFrom, nodeTo)
	// 用于对所有延迟数据求和后求平均
	var latencySum float64 = 0.0
	resultFile, err := os.Open(resultFilePath)
	if err != nil {
		return 0.0, fmt.Errorf("Failed Open File %s2%s.txt", nodeFrom, nodeTo)
	}
	defer resultFile.Close()
	// 保存每行数据字符串
	var lines []string
	scanner := bufio.NewScanner(resultFile)
	for scanner.Scan() {
		lines = append(lines, scanner.Text())
		if len(lines) > LAST_N_LINES {
			// 仅保留最后10行数据，代码含义为切片从下标为1的元素开始到最后一个元素作为一个新切片
			lines = lines[1:]
		}
	}
	if err := scanner.Err(); err != nil {
		return 0, fmt.Errorf("get Last N Lines in ResultFile Failed:%v", err)
	}

	for _, line := range lines {
		splitedLine := strings.Split(line, " ")
		dataStr := splitedLine[len(splitedLine)-2]
		dataStr, _ = strings.CutPrefix(dataStr, "(")
		latencyData, err := strconv.ParseFloat(dataStr, 64)
		if err != nil {
			return 0, fmt.Errorf("convert Data String to Float Failed:%v", err)
		}
		latencySum += latencyData
	}
	// 返回延迟平均值
	return latencySum / float64(len(lines)), nil
}
~~~



### common.go

~~~GO
package neilats_refactor_go

import "sync"

var RttMatrixFileLock sync.Mutex
var LatencyFileLock sync.Mutex
~~~



### neilats_unit_test.go

~~~GO
package neilats_refactor_go

import (
	"fmt"
	"k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/informers"
	clientsetfake "k8s.io/client-go/kubernetes/fake"
	"k8s.io/klog/v2/ktesting"
	"k8s.io/kubernetes/pkg/scheduler/framework"
	fakeframework "k8s.io/kubernetes/pkg/scheduler/framework/fake"
	"k8s.io/kubernetes/pkg/scheduler/framework/plugins/defaultbinder"
	"k8s.io/kubernetes/pkg/scheduler/framework/plugins/queuesort"
	frameworkruntime "k8s.io/kubernetes/pkg/scheduler/framework/runtime"
	st "k8s.io/kubernetes/pkg/scheduler/testing"
	testutil "sigs.k8s.io/scheduler-plugins/test/util"
	"testing"
)

func TestNeilatsUnit(t *testing.T) {
	tests := []struct {
		name       string                       // 测试名称
		nodeInfos  []*framework.NodeInfo        // 模拟节点信息
		pods       []*v1.Pod                    // 待测试的 Pod
		configData NeilatsRefactorSchedulerArgs //插件的参数配置
	}{
		{
			name: "neilats unit test SLA False",
			nodeInfos: []*framework.NodeInfo{
				makeNodeInfo("node1"),
				makeNodeInfo("node2"),
				makeNodeInfo("master"),
			},
			pods: []*v1.Pod{
				makeNeilatsTestPod("test-pod-1", "", "", 100, 500*1024*1024, 1024, 1024*1024),
				makeNeilatsTestPod("test-pod-2", "", "", 100, 200*1024*1024, 2048, 2*1024*1024),
			},
			configData: NeilatsRefactorSchedulerArgs{
				PrometheusAddress: "http://39.98.76.224:32599",
				NetworkDevice: map[string]string{
					"master": "eth0",
					"node1":  "eth0",
					"node2":  "eth0",
				},
				StorageDevice: map[string]string{
					"master": "/dev/vda2",
					"node1":  "/dev/vda2",
					"node2":  "/dev/vda2",
				},
				EnableSLA:                false,
				KubeNodeAddressAndSecret: map[string]UserAddressSecretMap{},
			},
		},
		{
			name: "neilats unit test SLA True",
			nodeInfos: []*framework.NodeInfo{
				makeNodeInfo("node1"),
				makeNodeInfo("node2"),
				makeNodeInfo("master"),
			},
			pods: []*v1.Pod{
				makeNeilatsTestPod("test-pod-0", "", "", 100, 500*1024*1024, 1024, 1024*1024),
				makeNeilatsTestPod("test-pod-1", "node3", "0.5", 100, 500*1024*1024, 1024, 1024*1024),
				makeNeilatsTestPod("test-pod-2", "node1", "as", 100, 200*1024*1024, 2048, 2*1024*1024),
				makeNeilatsTestPod("test-pod-3", "master", "0.6", 100, 200*1024*1024, 2048, 2*1024*1024),
				makeNeilatsTestPod("test-pod-4", "node2", "", 100, 200*1024*1024, 2048, 2*1024*1024),
				makeNeilatsTestPod("test-pod-5", "", "0.1", 100, 200*1024*1024, 2048, 2*1024*1024),
			},
			configData: NeilatsRefactorSchedulerArgs{
				PrometheusAddress: "http://39.98.76.224:32599",
				NetworkDevice: map[string]string{
					"master": "eth0",
					"node1":  "eth0",
					"node2":  "eth0",
				},
				StorageDevice: map[string]string{
					"master": "/dev/vda2",
					"node1":  "/dev/vda2",
					"node2":  "/dev/vda2",
				},
				EnableSLA: true,
				KubeNodeAddressAndSecret: map[string]UserAddressSecretMap{
					"master": {"39.98.76.224", "Mobisys912!"},
					"node2":  {"47.92.233.4", "Mobisys912!"},
					"node1":  {"47.92.228.102", "Mobisys912!"},
				},
			},
		},
	}

	for _, test := range tests {
		// 执行子测试
		t.Run(test.name, func(t *testing.T) {
			_, ctx := ktesting.NewTestContext(t)                         // 创建日志记录器，上下文
			cs := clientsetfake.NewSimpleClientset()                     // 创建假的客户端集，模拟Kubernetes API
			informerFactory := informers.NewSharedInformerFactory(cs, 0) // 创建共享的 informer 工厂

			// 注册插件，包括绑定插件，队列排序插件，以及自定义得分插件，除了自定义得分插件，其他插件均为必须的调度插件
			registeredPlugins := []st.RegisterPluginFunc{
				st.RegisterBindPlugin(defaultbinder.Name, defaultbinder.New),
				st.RegisterQueueSortPlugin(queuesort.Name, queuesort.New),

				st.RegisterPluginAsExtensions(Name, New, "PreFilter"),
				st.RegisterPluginAsExtensions(Name, New, "Filter"),
				st.RegisterPluginAsExtensions(Name, New, "Score"),
			}

			// 使用fakeSharedLister模拟节点信息，注入测试用例中的节点信息
			fakeSharedLister := &fakeSharedLister{nodes: test.nodeInfos}
			// 初始化框架和插件，注入所有依赖项，客户端，Informer，节点快照
			fh, err := st.NewFramework(
				ctx,
				registeredPlugins,
				"default-scheduler",
				frameworkruntime.WithClientSet(cs),
				frameworkruntime.WithInformerFactory(informerFactory),
				frameworkruntime.WithSnapshotSharedLister(fakeSharedLister),
				frameworkruntime.WithPodNominator(testutil.NewPodNominator(nil)),
			)
			if err != nil {
				t.Fatalf("fail to create framework: %s", err)
			}

			// 创建插件配置对象
			//var configObj runtime.Object
			//if test.config != nil {
			//	configObj = makePluginConfig(test.config)
			//}

			// 创建并初始化插件
			pe, _ := New(&test.configData, fh)
			// 由于实现了多个扩展点，我们将其断言为自定义调度插件类型，而不是单一的ScorePlugin类型
			neilatsPlugin := pe.(*NeilatsRefactorScheduler)
			//scorePlugin := pe.(framework.ScorePlugin)

			// 用于PreFilter阶段的单元测试
			prefilter_passed_pods := make([]*v1.Pod, 0)
			for _, pod := range test.pods {
				_, status := neilatsPlugin.PreFilter(ctx, nil, pod)
				if status.Code() == framework.Success {
					prefilter_passed_pods = append(prefilter_passed_pods, pod)
					t.Logf(fmt.Sprintf("Pod %s Pass PreFilter!", pod.Name))
				} else {
					t.Logf(fmt.Sprintf("Pod %s UnPass PreFilter!", pod.Name))
				}
			}

			// 用于Filter阶段的单元测试
			for _, pod := range prefilter_passed_pods {
				for _, node := range test.nodeInfos {
					status := neilatsPlugin.Filter(ctx, nil, pod, node)
					if status.Code() == framework.Success {
						t.Logf(fmt.Sprintf("Node %s Pass Pod %s SLA Constraint!", node.Node().Name, pod.Name))
					} else if status.Code() == framework.Unschedulable {
						t.Logf(fmt.Sprintf("Node %s UnPass Pod %s SLA Constraint!", node.Node().Name, pod.Name))
					} else {
						t.Logf(fmt.Sprintf("Node %s Get rttMatrix Failed!", node.Node().Name))
					}
				}
			}

			// 收集所有节点得分用户归一化,并打印原始节点评分
			var scoreList framework.NodeScoreList
			for _, nodeInfo := range test.nodeInfos {
				nodeName := nodeInfo.Node().Name
				score, err := neilatsPlugin.Score(ctx, nil, test.pods[0], nodeName)
				if err != nil {
					t.Logf("Scoring node %s failed: %v", nodeName, err)
					continue
				}
				t.Logf("Node %s score: %d", nodeName, score)
				scoreList = append(scoreList, framework.NodeScore{Name: nodeName, Score: score})
			}

			// 归一化并打印结果
			if status := neilatsPlugin.ScoreExtensions().NormalizeScore(ctx, nil, test.pods[0], scoreList); !status.IsSuccess() {
				t.Logf("Normalize score failed: %v", status)
			}
			t.Log("\n=== Normalized scores ===\n")
			for _, score := range scoreList {
				t.Logf("Node %s score: %d", score.Name, score.Score)
			}
		})
	}
}

func makeNodeInfo(node string) *framework.NodeInfo {
	nodeInfo := framework.NewNodeInfo()
	// 设置节点名称和容量
	nodeInfo.SetNode(&v1.Node{
		ObjectMeta: metav1.ObjectMeta{Name: node},
	})
	return nodeInfo
}

// 几个传入参数，CPU的单位是毫核， Memory的单位是字节， Network的单位是字节/秒， Disk的单位是字节
// 创建neilats调度器的单元测试Pod
func makeNeilatsTestPod(name, neiNode, sla string, cpu, memory, network, disk int64) *v1.Pod {
	pod := &v1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:   name,
			Labels: map[string]string{},
		},
		Spec: v1.PodSpec{
			Containers: []v1.Container{
				{
					Name:  "test-container-1",
					Image: "nginx:latest",
					// 一次性初始化所有资源请求和限制的方法
					Resources: v1.ResourceRequirements{
						Requests: v1.ResourceList{}, // 需要先初始化然后再执行添加操作
						Limits:   v1.ResourceList{}, // 需要先初始化然后再执行添加操作
					},
				},
			},
		},
	}
	if cpu > 0 {
		pod.Spec.Containers[0].Resources.Requests[v1.ResourceCPU] = *resource.NewMilliQuantity(cpu, resource.DecimalSI)
	}
	if memory > 0 {
		pod.Spec.Containers[0].Resources.Requests[v1.ResourceMemory] = *resource.NewQuantity(memory, resource.BinarySI)
	}
	if network > 0 {
		pod.Labels["network-request"] = fmt.Sprintf("%d", network)
	}
	if disk > 0 {
		pod.Labels["disk-request"] = fmt.Sprintf("%d", disk)
	}
	if neiNode != "" {
		pod.Labels["nei_node"] = neiNode
	}
	if sla != "" {
		pod.Labels["sla"] = sla
	}
	return pod
}

// 插件参数配置辅助函数
//func makePluginConfig(config map[string]interface{}) runtime.Object {
//	return &unstructured.Unstructured{
//		Object: map[string]interface{}{
//			"apiVersion": "v2",
//			"kind":       "NeilatsPluginConfig",
//			"args":       config,
//		},
//	}
//}

type fakeSharedLister struct {
	nodes []*framework.NodeInfo
}

func (f *fakeSharedLister) StorageInfos() framework.StorageInfoLister {
	return nil
}
func (f *fakeSharedLister) NodeInfos() framework.NodeInfoLister {
	return fakeframework.NodeInfoLister(f.nodes)
}
~~~

## 各代码问题记录

### neilats_refactor.go

#### 自定义调度器实现自定义参数配置

* 如下是自定义调度器的结构体和调度器配置参数结构体

  ~~~Go
  type NeilatsRefactorScheduler struct {
  	config NeilatsRefactorSchedulerArgs
  	handle framework.Handle
  	logger klog.Logger
  }
  
  // 声明自定义调度器的自定义参数结构体
  // 调度器添加自定义参数的方法，参考了其他调度插件中的pkg/nodeResources的实现方法
  type UserAddressSecretMap struct {
  	NodeAddress string
  	NodeSecret  string
  }
  type NeilatsRefactorSchedulerArgs struct {
  	// 如果自定义调度器有参数，需要参数结构体添加该内联类型，是Kubernetes库的要求
  	metav1.TypeMeta
  	PrometheusAddress        string                          `json:"prometheusAddress"`
  	NetworkDevice            map[string]string               `json:"networkDevice"`
  	StorageDevice            map[string]string               `json:"storageDevice"`
  	EnableSLA                bool                            `json:"enableSLA"`
  	KubeNodeAddressAndSecret map[string]UserAddressSecretMap `json:"kubeNodeAddressAndSecret"`
  }
  ~~~

* 自定义调度插件的参数以结构体方式村子啊，并且名称有具体要求，在`/scheduler/manifast/install/charts/as-a-second-scheduler/values.yaml`中有体现

  ~~~yaml
  # Customize the enabled plugins' config.
  # Refer to the "pluginConfig" section of manifests/<plugin>/scheduler-config.yaml.
  # For example, for Coscheduling plugin, you want to customize the permit waiting timeout to 10 seconds:
  pluginConfig:
  - name: Coscheduling
    args:
      permitWaitingTimeSeconds: 10 # default is 60
  # Or, customize the other plugins
  # - name: NodeResourceTopologyMatch
  #   args:
  #     scoringStrategy:
  #       type: MostAllocated # default is LeastAllocated
  #- name: SySched
  #  args:
  #    defaultProfileNamespace: "default"
  #    defaultProfileName: "full-seccomp"
  ~~~

  在https://github.com/kubernetes-sigs/scheduler-plugins/blob/release-1.28/doc/develop.md 也有说明：

  **if you are adding a new plugin args struct, to have it properly decoded, its name needs to follow the convention `<PluginName>Args`.**

* 由于如果要将在yaml中配置的参数传入自定义调度器的话，需要在调度器初始化阶段传入，即通过`func New(neilatsArgs runtime.Object, h framework.Handle) (framework.Plugin, error)`函数，所以需要使用传入的参数类型`runtime.Object`来初始化调度器配置，方式就是将`runtime.Object`类型转换为`NeilatsRefactorSchedulerArgs`类型

  这就有一个要求，即参数结构体需要实现`runtime.Object`类型的接口，并且还需要实现`DeepCopy`、`DeepCopyInto`（这两个方法也可以通过code-generator来生成）这两个接口，这是Kubernetes架构设计与代码生成工具的要求
  使用Kubernetes代码生成器的方法，请见本文开发测试日志部分[[neilats-refactor-scheduler-plugins#修改4]]、[[neilats-refactor-scheduler-plugins#修改6]]

  除此以外，参数结构体中还要加上`metav1.TypeMeta`，这是Kubernetes库的要求

  这样设计是因为需要参数结构体被设计为可以序列化、反序列化、在组件之间传递并且可以安全拷贝的；

* 如下是`DeepCopy`、`DeepCopyInto`两个方法的实现

  ~~~Go
  // 如果自定义调度器有参数，需要参数结构体实现runtime.Object类型的接口
  var _ runtime.Object = &NeilatsRefactorSchedulerArgs{}
  
  // 需要实现DeepCopyObject、DeepCopyInto这两个接口， 是Kubernetes架构设计与代码生成工具的要求
  // 该结构体要被设计为可以被序列化、反序列化、在组件之间传递，并且可以安全的拷贝
  // 这两个方法也可以通过code-generator来自动生成
  func (in *NeilatsRefactorSchedulerArgs) DeepCopyInto(out *NeilatsRefactorSchedulerArgs) {
  	*out = *in
  	out.TypeMeta = in.TypeMeta
  
  	// 拷贝基本字段
  	out.PrometheusAddress = in.PrometheusAddress
  	out.EnableSLA = in.EnableSLA
  
  	// 拷贝map[string]string
  	if in.NetworkDevice != nil {
  		out.NetworkDevice = make(map[string]string, len(in.NetworkDevice))
  		for k, v := range in.NetworkDevice {
  			out.NetworkDevice[k] = v
  		}
  	}
  	if in.StorageDevice != nil {
  		out.StorageDevice = make(map[string]string, len(in.StorageDevice))
  		for k, v := range in.StorageDevice {
  			out.StorageDevice[k] = v
  		}
  	}
  	if in.KubeNodeAddressAndSecret != nil {
  		out.KubeNodeAddressAndSecret = make(map[string]UserAddressSecretMap, len(in.KubeNodeAddressAndSecret))
  		for k, v := range in.KubeNodeAddressAndSecret {
  			out.KubeNodeAddressAndSecret[k] = UserAddressSecretMap{NodeAddress: v.NodeAddress, NodeSecret: v.NodeSecret}
  		}
  	}
  	return
  }
  
  func (in *NeilatsRefactorSchedulerArgs) DeepCopy() *NeilatsRefactorSchedulerArgs {
  	if in == nil {
  		return nil
  	}
  	out := new(NeilatsRefactorSchedulerArgs)
  	in.DeepCopyInto(out)
  	return out
  }
  
  // 需要参数结构体实现DeepCopyObject接口
  func (in *NeilatsRefactorSchedulerArgs) DeepCopyObject() runtime.Object {
  	if c := in.DeepCopy(); c != nil {
  		return c
  	}
  	return nil
  }
  ~~~

* 将`runtime.Object`类型转换为参数结构体类型并配置自定义调度器的代码如下

  ~~~GO
  // 新建并初始化一个新的插件并返回，将其注册到调度器中
  func New(neilatsArgs runtime.Object, h framework.Handle) (framework.Plugin, error) {
  	defaultConfig := NeilatsRefactorSchedulerArgs{
  		//..............
  	}
  
  	// 如果参数neilatsArgs不为空，则尝试解析其中的配置参数
  	if neilatsArgs != nil {
  		// 将runtime.Object类型的参数，转换为调度器所需的参数类型
  		args, ok := neilatsArgs.(*NeilatsRefactorSchedulerArgs)
  		if !ok {
  			return nil, fmt.Errorf("want args to be of type NeilatsRefactorSchedulerArgs, got %T", neilatsArgs)
  		}
  		// 如果各个参数存在，将defaultConfig 中的各项参数改为传入的参数，否则使用默认参数
  		if args.PrometheusAddress != "" {
  			defaultConfig.PrometheusAddress = args.PrometheusAddress
  		}
  		if len(args.NetworkDevice) > 0 {
  			defaultConfig.NetworkDevice = args.NetworkDevice
  		}
  		if len(args.StorageDevice) > 0 {
  			defaultConfig.StorageDevice = args.StorageDevice
  		}
  		defaultConfig.EnableSLA = args.EnableSLA
  		if args.EnableSLA {
  			if len(args.KubeNodeAddressAndSecret) == 0 {
  				return nil, fmt.Errorf("when EnableSLA is True, KubeNodeAddressAndSecret must have content")
  			}
  			defaultConfig.KubeNodeAddressAndSecret = args.KubeNodeAddressAndSecret
  			
              //..........
  		}
  	}
  
  	// 这里返回我们创建并初始化完成的自定义调度器
  	return &NeilatsRefactorScheduler{
  		handle: h,
  		config: defaultConfig,
  		logger: klog.FromContext(context.Background()),
  	}, nil
  }
  
  ~~~

#### PreFilter扩展点的实现

* 首先是声明扩展点的实现

  ~~~GO
  // 声明要实现的扩展点， 同时检测自定义调度器是否实现了这两个扩展点
  var _ framework.PreFilterPlugin = &NeilatsRefactorScheduler{}
  ~~~

  通过查看定义，发现`PreFilter`需要实现两个接口，即`PreFilter`、`PreFilterExtension`

  通过查看`PreFilterExtension`的定义，发现其包含了`AddPod`、`RemovePod`两个接口，因此一共需要实现四个接口

  `PreFilter`接口的作用就是过滤不符合调度要求的Pod,具体实现不用多说

* `AddPod`、`RemovePod`属于`PreFilterExtension`的实现部分,用于模拟Pod被正式调度之前，Pod被添加到节点上和从节点上删除的操作

  主要用于统计Pod的资源使用量、用于增量更新资源使用量

  由于此处实现的`PreFilter`接口仅用于检查Pod是否具有某个标签，不涉及Pod的资源使用量问题，可以简单实现`AddPod`、`RemovePod`

  ~~~GO
  func (neilats *NeilatsRefactorScheduler) PreFilterExtensions() framework.PreFilterExtensions {
  	return neilats
  }
  
  func (neilats *NeilatsRefactorScheduler) AddPod(ctx context.Context, cycleState *framework.CycleState, podToSchedule *v1.Pod, podToAdd *framework.PodInfo, nodeInfo *framework.NodeInfo) *framework.Status {
  	return framework.NewStatus(framework.Success, "")
  }
  
  func (neilats *NeilatsRefactorScheduler) RemovePod(ctx context.Context, cycleState *framework.CycleState, podToSchedule *v1.Pod, podToRemove *framework.PodInfo, nodeInfo *framework.NodeInfo) *framework.Status {
  	return framework.NewStatus(framework.Success, "")
  }
  
  ~~~

#### Score扩展点的实现

* 首先是声明扩展点的实现

  ~~~GO
  // 声明要实现的扩展点， 同时检测自定义调度器是否实现了这两个扩展点
  var _ framework.ScorePlugin = &NeilatsRefactorScheduler{}
  ~~~

  通过查看定义，发现`Score`扩展点需要实现两个接口，`Score`、`ScoreExtension`，进而查看`ScoreExtension`定义，发现需要实现`NormalizeScore`接口

* `Score`接口的实现和用于不再赘述

  由于不同的Score插件打分范围不同，Kubernetes调度器要求所有的插件分数最终归一化到0-100范围，以便同意加权计算总分，所以选哟引入`NormalizeScore`，在所有节点的Score执行完后，统一调整分数范围

  实现如下:

  ~~~GO
  func (neilats *NeilatsRefactorScheduler) ScoreExtensions() framework.ScoreExtensions {
  	return neilats
  }
  
  func (neilats *NeilatsRefactorScheduler) NormalizeScore(ctx context.Context, state *framework.CycleState, p *v1.Pod, scores framework.NodeScoreList) *framework.Status {
  	// 默认的排序方法
  	//return helper.DefaultNormalizeScore(framework.MaxNodeScore, false, scores)
  
  	// 将原始得分线性映射到[MinNodeScore, MaxNodeScore]，此处即0-100的范围内
  	// 找出原始的分数最小值和最大值
  	var maxScore int64 = -math.MaxInt64
  	var minScore int64 = math.MaxInt64
  	for _, nodeScore := range scores {
  		if nodeScore.Score > maxScore {
  			maxScore = nodeScore.Score
  		}
  		if nodeScore.Score < minScore {
  			minScore = nodeScore.Score
  		}
  	}
  
  	// 将原始得分映射到新的区间
  	oldRange := maxScore - minScore
  	newRange := framework.MaxNodeScore - framework.MinNodeScore
  	for i, nodeScore := range scores {
  		if oldRange == 0 {
  			scores[i].Score = framework.MinNodeScore
  		} else {
  			scores[i].Score = ((nodeScore.Score - minScore) / oldRange * newRange) + framework.MinNodeScore
  		}
  	}
  	return nil
  }
  ~~~

### get_prometheus_metrics.go

#### 使用Go语言接收Prometheus的Query结果

* 相对与使用Python来接收Prometheus的Query结果，使用Go会更加繁琐一些，需要定义一个用于接收返回结果的结构体

  ~~~GO
  type PrometheusQueryResponse struct {
  	Status string `json:"status"`
  	Data   struct {
  		ResultType string `json:"resultType"`
  		Result     []struct {
  			Metric map[string]string `json:"metric"`
  			Value  []interface{}     `json:"value"`
  		}
  	}
  }
  ~~~

  好在对于Prometheus的`/apiv1/query`端点来说，响应结构体是统一的，这个结构体适用于所有的PRome QL查询的响应，

* 对于Prometheus的查询，需要自己构建请求URL，并自己解析返回的响应

  ~~~GO
  func queryPrometheus(promURL string, promQL string) (*PrometheusQueryResponse, error) {
  	// 构造请求URL
  	u, _ := url.Parse(promURL)
  	u.Path = "/api/v1/query"
  	params := url.Values{"query": []string{promQL}}
  	u.RawQuery = params.Encode()
  
  	// 发起请求
  	resp, err := http.Get(u.String())
  	if err != nil {
  		fmt.Println("Error querying Prometheus:", err)
  		return nil, err
  	}
  	defer resp.Body.Close()
  
  	// 解析响应
  	var result PrometheusQueryResponse
  	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
  		fmt.Println("Error decoding Prometheus response:", err)
  		return nil, err
  	}
  
  	if result.Status != "success" {
  		return nil, fmt.Errorf("Prometheus query failed: %s", result.Status)
  	}
  	return &result, nil
  }
  ~~~

### latency_run_get.go

#### 使用ssh在远程主机上执行程序发生卡的处理

* 起因是在使用session.Run()来执行命令`cd /root/ws/network-latency-test && nohup ./latency_test.sh > /dev/null 2>&1 &`时，在第一个主机上执行这个命令可迅速返回，但执行第二个主机上执行时，会在`session.Run()`处卡顿，无响应，但是远程主机上已经实际运行了响应的命令；

  了解到问题本质在于`session.Run()`会等待远程命令结束，但使用`nohup &`后台运行，SSH session没有干净退出

  1. session.Run会等待远程进程完全释放所有文件描述符
  2. session.Run是为同步命令设计的，其要求执行一个命令，等待它退出，然后返回
  3. Go的ssh.Session默认会分配PTY伪终端

  可以使用如下解决方法：

  1. 使用`session.Start()`+`session.Close()`

     ~~~GO
     func executeCommandOnRemoteAsync(session *ssh.Session, command string) error {
         // Start 不等待命令结束
         if err := session.Start(command); err != nil {
             return err
         }
     
         // 立即关闭 stdin，通知远程命令可以脱离
         session.SendRequest("exit-status", false, nil)
     
         // 可以 sleep 一点时间确保命令已启动（可选）
         // time.Sleep(100 * time.Millisecond)
     
         // 立即关闭 session（不要调用 Wait）
         return session.Close()
     }
     ~~~

  2. 使用ssh的-f和disown更彻底的脱离

     ~~~GO
     command := `cd /root/ws/network-latency-test && nohup ./latency_test.sh > latency.log 2>&1 < /dev/null & disown`
     ~~~

     `< /dev/null`用于防止stdin阻塞，`disown`使得从shell job control中移除，脱离session

  3. 使用`setsid`让进程脱离session

     ~~~GO
     command := `cd /root/ws/network-latency-test && nohup setsid ./latency_test.sh > latency.log 2>&1 < /dev/null &`
     ~~~

     `setsid`让进程成为新session的leader，完全脱离原SSH Session

  4. 使用`screen`或`tmux`

     ~~~GO
     command := `cd /root/ws/network-latency-test && screen -dmS latency_test ./latency_test.sh`
     ~~~

     `screen -dmS`是的后台启动一个screen会话，完全独立于SSH，即使网络断开也不影响

### build-rtt-matrix.go

#### 读取文件的倒数N行的实现

* 其实想法很简单，就是让存储行数据的变量长度始终控制在N，文件中超过N的部分循环回去覆盖原来的数据

  ~~~GO
  	resultFile, err := os.Open(resultFilePath)
  	if err != nil {
  		return 0.0, fmt.Errorf("Failed Open File %s2%s.txt", nodeFrom, nodeTo)
  	}
  	defer resultFile.Close()
  	// 保存每行数据字符串
  	var lines []string
  	scanner := bufio.NewScanner(resultFile)
  	for scanner.Scan() {
  		lines = append(lines, scanner.Text())
  		if len(lines) > LAST_N_LINES {
  			// 仅保留最后10行数据，代码含义为切片从下标为1的元素开始到最后一个元素作为一个新切片
  			lines = lines[1:]
  		}
  	}
  	if err := scanner.Err(); err != nil {
  		return 0, fmt.Errorf("get Last N Lines in ResultFile Failed:%v", err)
  	}
  ~~~

### common.go

#### 变量互斥锁

* 主要用于控制几个文件的互斥读取的，比如rtt-matrix、下载下来的延迟数据文件，但是由于后面并不打算采用多线程运行延迟数据下载、延迟矩阵构建，所以放弃了使用文件互斥锁

  但是在这里还是讲述一下使用方法

  ~~~GO
  import "sync"
  
  var RttMatrixFileLock sync.Mutex
  var LatencyFileLock sync.Mutex
  ~~~

  这是定义

  至于使用，只需要在临界区之前使用`.Lock()`,临界区之后使用`.Unlock()`即可

  如果直到函数返回都是临界区，可以使用defer

  ~~~GO
  			// 获取RTT矩阵文件的文件锁
  			RttMatrixFileLock.Lock()
  			// 由于后面的操作都涉及到rttMatrix文件的写入，所以推迟文件锁的解锁操作到函数结束
  			defer RttMatrixFileLock.Unlock()
  ~~~

### neilats_unit_test.go

> 该部分其实就是自定义调度插件的单元测试了，其实算是挺重要的部分，还是逐个部分讲解吧

#### 测试案例结构体

~~~GO
	tests := []struct {
		name       string                       // 测试名称
		nodeInfos  []*framework.NodeInfo        // 模拟节点信息
		pods       []*v1.Pod                    // 待测试的 Pod
		configData NeilatsRefactorSchedulerArgs //插件的参数配置
	}
~~~

显而易见，就是用于调度插件测试的每个测试案例的结构体，后面跟的是逐个测试案例

~~~GO
	tests := []struct {
		name       string                       // 测试名称
		nodeInfos  []*framework.NodeInfo        // 模拟节点信息
		pods       []*v1.Pod                    // 待测试的 Pod
		configData NeilatsRefactorSchedulerArgs //插件的参数配置
	}{
		{
			name: "neilats unit test SLA False",
			nodeInfos: []*framework.NodeInfo{
				makeNodeInfo("node1"),
				makeNodeInfo("node2"),
				makeNodeInfo("master"),
			},
			pods: []*v1.Pod{
				makeNeilatsTestPod("test-pod-1", "", "", 100, 500*1024*1024, 1024, 1024*1024),
				makeNeilatsTestPod("test-pod-2", "", "", 100, 200*1024*1024, 2048, 2*1024*1024),
			},
			configData: NeilatsRefactorSchedulerArgs{
				PrometheusAddress: "http://39.98.76.224:32599",
				NetworkDevice: map[string]string{
					"master": "eth0",
					"node1":  "eth0",
					"node2":  "eth0",
				},
				StorageDevice: map[string]string{
					"master": "/dev/vda2",
					"node1":  "/dev/vda2",
					"node2":  "/dev/vda2",
				},
				EnableSLA:                false,
				KubeNodeAddressAndSecret: map[string]UserAddressSecretMap{},
			},
		},
		{
			name: "neilats unit test SLA True",
			nodeInfos: []*framework.NodeInfo{
				makeNodeInfo("node1"),
				makeNodeInfo("node2"),
				makeNodeInfo("master"),
			},
			pods: []*v1.Pod{
				makeNeilatsTestPod("test-pod-0", "", "", 100, 500*1024*1024, 1024, 1024*1024),
				makeNeilatsTestPod("test-pod-1", "node3", "0.5", 100, 500*1024*1024, 1024, 1024*1024),
				makeNeilatsTestPod("test-pod-2", "node1", "as", 100, 200*1024*1024, 2048, 2*1024*1024),
				makeNeilatsTestPod("test-pod-3", "master", "0.6", 100, 200*1024*1024, 2048, 2*1024*1024),
				makeNeilatsTestPod("test-pod-4", "node2", "", 100, 200*1024*1024, 2048, 2*1024*1024),
				makeNeilatsTestPod("test-pod-5", "", "0.1", 100, 200*1024*1024, 2048, 2*1024*1024),
			},
			configData: NeilatsRefactorSchedulerArgs{
				PrometheusAddress: "http://39.98.76.224:32599",
				NetworkDevice: map[string]string{
					"master": "eth0",
					"node1":  "eth0",
					"node2":  "eth0",
				},
				StorageDevice: map[string]string{
					"master": "/dev/vda2",
					"node1":  "/dev/vda2",
					"node2":  "/dev/vda2",
				},
				EnableSLA: true,
				KubeNodeAddressAndSecret: map[string]UserAddressSecretMap{
					"master": {"39.98.76.224", "Mobisys912!"},
					"node2":  {"47.92.233.4", "Mobisys912!"},
					"node1":  {"47.92.228.102", "Mobisys912!"},
				},
			},
		},
	}
~~~

#### 进行逐测试案例的测试运行

~~~GO
	for _, test := range tests {
		// 执行子测试
		t.Run(test.name, func(t *testing.T) {
            //........
~~~

这就是逐个测试案例的运行

#### 为调度插件测试创建背景环境

* 日志记录器、上下文、假客户端集、Informer工厂的创建

  ~~~GO
  			_, ctx := ktesting.NewTestContext(t)                         // 创建日志记录器，上下文
  			cs := clientsetfake.NewSimpleClientset()                     // 创建假的客户端集，模拟Kubernetes API
  			informerFactory := informers.NewSharedInformerFactory(cs, 0) // 创建共享的 informer 工厂
  ~~~

* 注册插件

  ~~~GO
  			registeredPlugins := []st.RegisterPluginFunc{
  				st.RegisterBindPlugin(defaultbinder.Name, defaultbinder.New),
  				st.RegisterQueueSortPlugin(queuesort.Name, queuesort.New),
  
  				st.RegisterPluginAsExtensions(Name, New, "PreFilter"),
  				st.RegisterPluginAsExtensions(Name, New, "Filter"),
  				st.RegisterPluginAsExtensions(Name, New, "Score"),
  			}
  ~~~

  自定义调度器实现了哪几个扩展点，就在后面加上哪几个扩展点的注册信息

* 使用fakeSharedLister模拟节点信息，注入测试用例中的节点信息

  ~~~GO
  			fakeSharedLister := &fakeSharedLister{nodes: test.nodeInfos}
  ........
  
  type fakeSharedLister struct {
  	nodes []*framework.NodeInfo
  }
  
  func (f *fakeSharedLister) StorageInfos() framework.StorageInfoLister {
  	return nil
  }
  func (f *fakeSharedLister) NodeInfos() framework.NodeInfoLister {
  	return fakeframework.NodeInfoLister(f.nodes)
  }
  ~~~

  这是Kubernetes调度器测试中标准来提供模拟的节点信息

  `fakeSharedLister`是实现了`framework.SharedLister`接口的测试专用结构体

* 初始化框架和插件，注入所有依赖项、客户端、Informer、节点快照等

  ~~~GO
  			fh, err := st.NewFramework(
  				ctx,
  				registeredPlugins,
  				"default-scheduler",
  				frameworkruntime.WithClientSet(cs),
  				frameworkruntime.WithInformerFactory(informerFactory),
  				frameworkruntime.WithSnapshotSharedLister(fakeSharedLister),
  				frameworkruntime.WithPodNominator(testutil.NewPodNominator(nil)),
  			)
  			if err != nil {
  				t.Fatalf("fail to create framework: %s", err)
  			}
  ~~~

  本来以为创建的Framework fh在后来的单元测试中，并没有被使用到，但是后来才发现在创建自定义调度器的时候当作参数传入了，作用如下：

  frameworkHandle是必须的，几乎所有的非trivial插件都会用到；fh提供获取待调度Pod列表、获取节点信息快照、与APIServer通信、记录调度事件、监控Pods和Nodes等资源变化的功能

  如果插件实现了Filter、Score、PreBind等扩展点，需要访问集群状态或发送事件，就必须使用fh

  > 非trivial插件：

* 创建并初始化自定义调度插件

  ~~~GO
  			pe, _ := New(&test.configData, fh)
  			// 由于实现了多个扩展点，我们将其断言为自定义调度插件类型，而不是单一的ScorePlugin类型
  			neilatsPlugin := pe.(*NeilatsRefactorScheduler)
  			//scorePlugin := pe.(framework.ScorePlugin)
  ~~~

  **这种使用New函数创建插件的流程，是标准的插件创建流程，完全模拟了生产环境中调度器加载插件的过程**

  其调用New函数、对配置进行解码、符合插件注册规范、可用于生产、常用于集成测试与e2e测试

  其实后来在其他调度插件的单元测试文件中发现了另一种创建自定义调度器的方式，如下：

  ~~~Go
  			temp := &NeilatsRefactorScheduler{
  				handle: fh,
  				config: test.configData,
  			}
  ~~~

  **该方法不调用New函数，使用手动赋值，不对配置进行解码；不符合插件注册规范；不可用于生产；常用于单元测试、快速逻辑验证测试**

  同时，由于直接传入了强类型的`test.configData`，所以不再需要通过`runtime.Object`反序列化来配置调度器参数

* Prefilter测试

  ~~~GO
  			prefilter_passed_pods := make([]*v1.Pod, 0)
  			for _, pod := range test.pods {
  				_, status := neilatsPlugin.PreFilter(ctx, nil, pod)
  				if status.Code() == framework.Success {
  					prefilter_passed_pods = append(prefilter_passed_pods, pod)
  					t.Logf(fmt.Sprintf("Pod %s Pass PreFilter!", pod.Name))
  				} else {
  					t.Logf(fmt.Sprintf("Pod %s UnPass PreFilter!", pod.Name))
  				}
  			}
  ~~~

  针对`Prefilter`扩展点的逻辑测试

* Filter测试

  ~~~GO
  			for _, pod := range prefilter_passed_pods {
  				for _, node := range test.nodeInfos {
  					status := neilatsPlugin.Filter(ctx, nil, pod, node)
  					if status.Code() == framework.Success {
  						t.Logf(fmt.Sprintf("Node %s Pass Pod %s SLA Constraint!", node.Node().Name, pod.Name))
  					} else if status.Code() == framework.Unschedulable {
  						t.Logf(fmt.Sprintf("Node %s UnPass Pod %s SLA Constraint!", node.Node().Name, pod.Name))
  					} else {
  						t.Logf(fmt.Sprintf("Node %s Get rttMatrix Failed!", node.Node().Name))
  					}
  				}
  			}
  ~~~

  针对`Filter`扩展点的逻辑测试，其使用的Pod都是经过`PreFilter`过滤之后的Pod，这里使用的手动模仿两个扩展点之间Pod的约束

* Score测试

  ~~~GO
  			var scoreList framework.NodeScoreList
  			for _, nodeInfo := range test.nodeInfos {
  				nodeName := nodeInfo.Node().Name
  				score, err := neilatsPlugin.Score(ctx, nil, test.pods[0], nodeName)
  				if err != nil {
  					t.Logf("Scoring node %s failed: %v", nodeName, err)
  					continue
  				}
  				t.Logf("Node %s score: %d", nodeName, score)
  				scoreList = append(scoreList, framework.NodeScore{Name: nodeName, Score: score})
  			}
  
  			// 归一化并打印结果
  			if status := neilatsPlugin.ScoreExtensions().NormalizeScore(ctx, nil, test.pods[0], scoreList); !status.IsSuccess() {
  				t.Logf("Normalize score failed: %v", status)
  			}
  			t.Log("\n=== Normalized scores ===\n")
  			for _, score := range scoreList {
  				t.Logf("Node %s score: %d", score.Name, score.Score)
  			}
  ~~~

  这里针对`Score`扩展点的测试，只使用了一个Pod

* 虚拟Node的创建

  ~~~GO
  func makeNodeInfo(node string) *framework.NodeInfo {
  	nodeInfo := framework.NewNodeInfo()
  	// 设置节点名称和容量
  	nodeInfo.SetNode(&v1.Node{
  		ObjectMeta: metav1.ObjectMeta{Name: node},
  	})
  	return nodeInfo
  }
  ~~~

  用于在单元测试中，模拟虚拟的节点信息，上述创建函数，仅包含节点名称的创建

* 虚拟Pod的创建

  ~~~GO
  func makeNeilatsTestPod(name, neiNode, sla string, cpu, memory, network, disk int64) *v1.Pod {
  	pod := &v1.Pod{
  		ObjectMeta: metav1.ObjectMeta{
  			Name:   name,
  			Labels: map[string]string{},
  		},
  		Spec: v1.PodSpec{
  			Containers: []v1.Container{
  				{
  					Name:  "test-container-1",
  					Image: "nginx:latest",
  					// 一次性初始化所有资源请求和限制的方法
  					Resources: v1.ResourceRequirements{
  						Requests: v1.ResourceList{}, // 需要先初始化然后再执行添加操作
  						Limits:   v1.ResourceList{}, // 需要先初始化然后再执行添加操作
  					},
  				},
  			},
  		},
  	}
  	if cpu > 0 {
  		pod.Spec.Containers[0].Resources.Requests[v1.ResourceCPU] = *resource.NewMilliQuantity(cpu, resource.DecimalSI)
  	}
  	if memory > 0 {
  		pod.Spec.Containers[0].Resources.Requests[v1.ResourceMemory] = *resource.NewQuantity(memory, resource.BinarySI)
  	}
  	if network > 0 {
  		pod.Labels["network-request"] = fmt.Sprintf("%d", network)
  	}
  	if disk > 0 {
  		pod.Labels["disk-request"] = fmt.Sprintf("%d", disk)
  	}
  	if neiNode != "" {
  		pod.Labels["nei_node"] = neiNode
  	}
  	if sla != "" {
  		pod.Labels["sla"] = sla
  	}
  	return pod
  }
  ~~~

  其实主要就是用于在单元测试中，创建模拟出来的Pod资源，上述创建函数包含了创建Pod的Request资源、Label、Name、容器列表、容器镜像等

  需要注意的是，Resources和Labels是需要预先创建出空的变量后，才可以添加

#### 使用框架集成测试

* 框架集成测试的代码(并未实际采用)

  ~~~Go
  	// 执行子测试
  	for _, test := range tests {
  		// 针对测试案例中的每个Pod执行调度测试
  		for _, pod := range test.pods {
  			t.Run(test.name, func(t *testing.T) {
  				_, ctx := ktesting.NewTestContext(t)                         // 创建日志记录器，上下文
  				cs := clientsetfake.NewSimpleClientset()                     // 创建假的客户端集，模拟Kubernetes API
  				informerFactory := informers.NewSharedInformerFactory(cs, 0) // 创建共享的 informer 工厂
  
  				// 注册插件，包括绑定插件，队列排序插件，以及自定义得分插件，除了自定义得分插件，其他插件均为必须的调度插件
  				registeredPlugins := []st.RegisterPluginFunc{
  					st.RegisterBindPlugin(defaultbinder.Name, defaultbinder.New),
  					st.RegisterQueueSortPlugin(queuesort.Name, queuesort.New),
  
  					st.RegisterPluginAsExtensions(Name, New, "PreFilter"),
  					st.RegisterPluginAsExtensions(Name, New, "Filter"),
  					st.RegisterPluginAsExtensions(Name, New, "Score"),
  				}
  
  				// 使用fakeSharedLister模拟节点信息，注入测试用例中的节点信息
  				fakeSharedLister := &fakeSharedLister{nodes: test.nodeInfos}
  				// 初始化框架和插件，注入所有依赖项，客户端，Informer，节点快照
  				fh, err := st.NewFramework(
  					ctx,
  					registeredPlugins,
  					"default-scheduler",
  					frameworkruntime.WithClientSet(cs),
  					frameworkruntime.WithInformerFactory(informerFactory),
  					frameworkruntime.WithSnapshotSharedLister(fakeSharedLister),
  					frameworkruntime.WithPodNominator(testutil.NewPodNominator(nil)),
  				)
  				if err != nil {
  					t.Fatalf("fail to create framework: %s", err)
  				}
  
  				// 用于PreFilter阶段的单元测试：使用框架集成方式
  				state := framework.NewCycleState()
  				_, status := fh.RunPreFilterPlugins(ctx, state, pod)
  				if status.IsSuccess() {
  					t.Logf(fmt.Sprintf("Pod %s Pass PreFilter!", pod.Name))
  				} else {
  					t.Logf(fmt.Sprintf("Pod %s UnPass PreFilter!", pod.Name))
  				}
  
  				// 验证PreFilter是否写入了state
  				//preFilterResult, err := state.Read()
  
  				// 用于Filter阶段的单元测试:框架集成方式
  				for _, node := range test.nodeInfos {
  					// 从集成框架中获取NodeInfo
  					nodeInfo, err := fh.SnapshotSharedLister().NodeInfos().Get(node.Node().Name)
  					if err != nil {
  						t.Fatalf("failed to get node from framework for %s: %v", node.Node().Name, err)
  					}
  
  					status := fh.RunFilterPlugins(ctx, state, pod, nodeInfo)
  					if status.IsSuccess() {
  						t.Logf(fmt.Sprintf("Node %s Pass Pod %s SLA Constraint!", node.Node().Name, pod.Name))
  					} else if status.IsUnschedulable() {
  						t.Logf(fmt.Sprintf("Node %s UnPass Pod %s SLA Constraint!", node.Node().Name, pod.Name))
  					} else {
  						t.Logf(fmt.Sprintf("Node %s Get rttMatrix Failed!", node.Node().Name))
  					}
  				}
  
  				// ScorePlugins测试：框架集成方式
  				var nodeList []*v1.Node
  				nodeInfos, err := fh.SnapshotSharedLister().NodeInfos().List()
  				for _, nodeInfo := range nodeInfos {
  					nodeList = append(nodeList, nodeInfo.Node())
  				}
  				scores, status := fh.RunScorePlugins(ctx, state, test.pods[0], nodeList)
  				if !status.IsSuccess() {
  					t.Errorf("RunScorePlugins Failed: %v", status)
  				}
  				t.Log("=== Normalized Scores (Via Framework) ===")
  				for _, score := range scores {
  					t.Logf("Node %s: %d", score.Name, score.Scores)
  				}
  			})
  		}
  	}
  ~~~

  

* 框架集成测试中调度器参数传入

  这是一个**极其关键**的问题，触及了调度器插件测试中的 **配置传递机制**。你问到了点子上！

  在集成测试中，使用 `fh`（`framework.Framework`）调用插件时，**插件的配置参数（如阈值、策略、URL等）必须通过 `KubeSchedulerConfiguration` 传入**，而不是直接构造插件实例。

  ---

  ### ✅ 核心答案：

  > **通过 `frameworkruntime.WithComponentConfig()` 注入 `KubeSchedulerConfiguration`，其中包含你的插件配置。**

  ---

  ### ✅ 详细步骤：如何向插件传递配置参数

  #### 1. 定义你的插件配置结构体

  ```go
  // pkg/scheduler/apis/myplugin/v1/types.go
  package v1
  
  import (
      metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
      schedulerconfig "k8s.io/kube-scheduler/config/v1"
  )
  
  // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
  
  // MyPluginArgs 是你的插件配置参数
  type MyPluginArgs struct {
      metav1.TypeMeta `json:",inline"`
  
      // 调度器配置基类
      schedulerconfig.PluginConfig `json:",inline"`
  
      // 你的自定义参数
      HighThreshold float64 `json:"highThreshold,omitempty"`
      LowThreshold  float64 `json:"lowThreshold,omitempty"`
      MaxRetries    int32   `json:"maxRetries,omitempty"`
      EnableFeature bool    `json:"enableFeature,omitempty"`
  }
  ```

  #### 2. 在插件中读取配置

  ```go
  // pkg/scheduler/myplugin/myplugin.go
  type MyPlugin struct {
      fh    framework.Framework
      args  *v1.MyPluginArgs
      score float64
  }
  
  func New(fh framework.Framework, args runtime.Object) (framework.Plugin, error) {
      // 断言参数类型
      pluginArgs, ok := args.(*v1.MyPluginArgs)
      if !ok {
          return nil, fmt.Errorf("want args to be of type MyPluginArgs, got %T", args)
      }
  
      // 验证参数（可选）
      if err := validateArgs(pluginArgs); err != nil {
          return nil, err
      }
  
      return &MyPlugin{
          fh:   fh,
          args: pluginArgs,
      }, nil
  }
  
  func (pl *MyPlugin) Filter(...) {
      // 使用配置参数
      if pl.args.HighThreshold > 0.8 {
          // 应用激进策略
      }
  }
  ```

  #### 3. 在测试中创建 `KubeSchedulerConfiguration` 并注入 `fh`

  ```go
  // test/myplugin_test.go
  func TestMyPlugin_WithConfig(t *testing.T) {
      // 1. 构造你的插件配置
      pluginArgs := &v1.MyPluginArgs{
          HighThreshold: 0.9,
          LowThreshold:  0.3,
          MaxRetries:    3,
          EnableFeature: true,
      }
  
      // 2. 创建 PluginConfig
      pluginConfig := schedulerconfig.PluginConfig{
          Name: "MyPlugin",
          Args: pluginArgs,
      }
  
      // 3. 创建 KubeSchedulerConfiguration
      componentConfig := &schedulerconfig.KubeSchedulerConfiguration{
          Profiles: []schedulerconfig.KubeSchedulerProfile{
              {
                  Plugins: &schedulerconfig.Plugins{
                      Filter: schedulerconfig.PluginSet{
                          Enabled: []schedulerconfig.Plugin{
                              {Name: "MyPlugin"},
                          },
                      },
                      Score: schedulerconfig.PluginSet{
                          Enabled: []schedulerconfig.Plugin{
                              {Name: "MyPlugin", Weight: 10},
                          },
                      },
                  },
                  PluginConfig: []schedulerconfig.PluginConfig{
                      pluginConfig, // 👈 注入你的配置
                  },
              },
          },
      }
  
      // 4. 创建 fakeSharedLister
      fakeLister := &fakeSharedLister{nodes: testNodes}
  
      // 5. 创建 Framework，注入配置
      fh, err := st.NewFramework(
          ctx,
          []st.RegisterPluginFunc{
              st.Register("MyPlugin", New), // 注册你的插件工厂函数
          },
          "default-scheduler",
          frameworkruntime.WithSnapshotSharedLister(fakeLister),
          frameworkruntime.WithComponentConfig(*componentConfig), // 👈 注入配置
          frameworkruntime.WithPodNominator(nominator),
      )
      if err != nil {
          t.Fatal(err)
      }
  
      // 6. 现在 fh 中的插件已经持有你传入的配置！
      // 可以安全调用 RunFilterPlugins 等
      state := framework.NewCycleState()
      status := fh.RunFilterPlugins(ctx, state, pod, nodeInfo)
  }
  ```

  ---

  ### ✅ 为什么必须这样做？

  因为：

  1. **模拟真实调度器行为**
     - 生产调度器从 `--config` 文件加载 `KubeSchedulerConfiguration`。
     - 插件通过 `args runtime.Object` 接收配置。

  2. **框架统一管理插件生命周期**
     - `framework` 负责调用 `New()` 工厂函数，并传入 `args`。
     - 你不能手动 `&MyPlugin{args: ...}`，否则 `fh` 不知道这个实例。

  3. **支持多 Profile**
     - 一个调度器可以有多个 `Profile`，每个 Profile 可以为同一插件传入不同参数。

  ---

  ### ✅ 验证配置是否传入成功（测试中）

  ```go
  // 在测试中验证插件是否收到正确配置
  t.Run("validate plugin args", func(t *testing.T) {
      // 获取插件实例（需要暴露 getter 或使用反射）
      plugin := fh.PreFilterPlugins()["MyPlugin"]
      myPlugin, ok := plugin.(*MyPlugin)
      if !ok {
          t.Fatal("plugin type assertion failed")
      }
  
      if myPlugin.args.HighThreshold != 0.9 {
          t.Errorf("want HighThreshold=0.9, got %f", myPlugin.args.HighThreshold)
      }
  })
  ```

  > ⚠️ 注意：`fh` 不直接暴露插件实例。你可能需要：
  > - 在测试包中提供 `GetMyPlugin(fh framework.Framework) *MyPlugin`
  > - 或使用 `reflect`（不推荐）

  ---

  ### ✅ 最佳实践建议

  | 做法                                | 说明                           |
  | ----------------------------------- | ------------------------------ |
  | ✅ 定义 `v1.MyPluginArgs`            | 使用 API 组 + 版本控制配置结构 |
  | ✅ 实现 `defaulting` 和 `validation` | 为字段提供默认值和校验         |
  | ✅ 在 `New()` 中验证 `args`          | 确保配置合法                   |
  | ✅ 使用 `WithComponentConfig()`      | 在测试中注入完整配置           |
  | ✅ 避免全局变量传参                  | 不要 `var TestConfig = ...`    |

  ---

  ### ✅ 总结

  | 问题                             | 解决方案                                                     |
  | -------------------------------- | ------------------------------------------------------------ |
  | **如何向集成测试中的插件传参？** | 通过 `KubeSchedulerConfiguration.PluginConfig`               |
  | **具体怎么操作？**               | 1. 定义 `MyPluginArgs`<br>2. 在 `New()` 中接收 `args`<br>3. 用 `WithComponentConfig()` 注入配置 |
  | **为什么不能直接传？**           | 因为 `fh` 管理插件生命周期，必须通过标准配置机制注入         |

  > 🎯 **记住**：  
  > 在集成测试中，**一切都要像真实调度器一样工作**。  
  > 配置 → `KubeSchedulerConfiguration` → `fh` → 插件 `New()` 函数。

* 每个Pod的测试独立循环

* 

## 开发测试日志

### 25.8.27

> 本次是neilats-refactor自定义调度器代码设计实现后，仅仅经过三个扩展点的单元逻辑测试，没有经过集成测试的第一次上实机测试
>
> 启动正常，运行正常
>
> 在接收Pod并对齐进行调度时，发生错误

* 错误日志如下：

  ~~~sh
  (base) root@master:~/work/SchedulingFramework# kubectl logs neilats-refactor-scheduler-8dc48d54-t4th7 -n k8s-learn
  I0827 08:17:43.548350       1 serving.go:348] Generated self-signed cert in-memory
  W0827 08:17:43.548855       1 client_config.go:618] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
  I0827 08:17:43.911734       1 capacity_scheduling.go:190] "CapacityScheduling start"
  I0827 08:17:43.912366       1 server.go:154] "Starting Kubernetes Scheduler" version="v0.0.20250826"
  I0827 08:17:43.912380       1 server.go:156] "Golang settings" GOGC="" GOMAXPROCS="" GOTRACEBACK=""
  I0827 08:17:43.915805       1 requestheader_controller.go:169] Starting RequestHeaderAuthRequestController
  I0827 08:17:43.915832       1 shared_informer.go:311] Waiting for caches to sync for RequestHeaderAuthRequestController
  I0827 08:17:43.915841       1 configmap_cafile_content.go:202] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::client-ca-file"
  I0827 08:17:43.915848       1 configmap_cafile_content.go:202] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
  I0827 08:17:43.915852       1 shared_informer.go:311] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
  I0827 08:17:43.915858       1 shared_informer.go:311] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
  I0827 08:17:43.916101       1 secure_serving.go:213] Serving securely on [::]:10259
  I0827 08:17:43.916153       1 tlsconfig.go:240] "Starting DynamicServingCertificateController"
  I0827 08:17:44.015964       1 shared_informer.go:318] Caches are synced for RequestHeaderAuthRequestController
  I0827 08:17:44.015977       1 shared_informer.go:318] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
  I0827 08:17:44.015977       1 shared_informer.go:318] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
  Total CPU of node node1 is 8000
  Total CPU of node node2 is 8000
  Total CPU of node node3 is 8000
  CPU idle rate of node node1 is 0.9740831021825964
  CPU idle rate of node node2 is 0.971455035365414
  CPU idle rate of node node3 is 0.8611212282530519
  Total memory of node node1 is 15991.8828125 MB
  Total memory of node node2 is 15991.87890625 MB
  Total memory of node node3 is 15991.91796875 MB
  Memory available of node node1 is 0.36791077113828746
  Memory available of node node3 is 0.2245858564787731
  Memory available of node node2 is 0.2880517235188466
  Network available of node node1 is 3.4436522666666667
  Network available of node node2 is 3.4915427111111113
  Total disk of node node1 is 39001.28125 MB
  Total disk of node node2 is 39001.28125 MB
  E0827 08:17:44.025309       1 runtime.go:79] Observed a panic: runtime.boundsError{x:0, y:0, signed:true, code:0x0} (runtime error: index out of range [0] with length 0)
  goroutine 596 [running]:
  k8s.io/apimachinery/pkg/util/runtime.logPanic({0x24d0340?, 0xc000f03350})
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/apimachinery/pkg/util/runtime/runtime.go:75 +0x85
  k8s.io/apimachinery/pkg/util/runtime.HandleCrash({0x0, 0x0, 0x5?})
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/apimachinery/pkg/util/runtime/runtime.go:49 +0x6b
  panic({0x24d0340?, 0xc000f03350?})
          /usr/local/go/src/runtime/panic.go:914 +0x21f
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.GetNodeNetworkAvailableRate({0x2691d01, 0x1a}, {0xc00092322a, 0x5}, {0x0, 0x0})
          /go/src/sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go/get_prometheus_metrics.go:107 +0x57c
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.(*NeilatsRefactorScheduler).LBScore(0xc00063ae00, 0xc000f1fb00, {0xc00092322a, 0x5})
          /go/src/sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go/neilats_refactor.go:322 +0x55f
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.(*NeilatsRefactorScheduler).Score(0xc000d86f90?, {0x2a3bce8?, 0xc000a83360?}, 0xc000cab7f8?, 0xc000f1fb00?, {0xc00092322a?, 0x5?})
          /go/src/sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go/neilats_refactor.go:264 +0x25
  k8s.io/kubernetes/pkg/scheduler/framework/runtime.(*instrumentedScorePlugin).Score(0xc000125f60, {0x2a3bce8, 0xc000a83360}, 0xc0017d4da0?, 0x20?, {0xc00092322a, 0x5})
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/kubernetes/pkg/scheduler/framework/runtime/instrumented_plugins.go:82 +0x85
  k8s.io/kubernetes/pkg/scheduler/framework/runtime.(*frameworkImpl).runScorePlugin(0x2a40ea8?, {0x2a3bce8?, 0xc000a83360?}, {0x2a2ddc0?, 0xc000125f60?}, 0x4109a5?, 0xc0015a4db0?, {0xc00092322a?, 0x30?})
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/kubernetes/pkg/scheduler/framework/runtime/framework.go:1126 +0x2d8
  k8s.io/kubernetes/pkg/scheduler/framework/runtime.(*frameworkImpl).RunScorePlugins.func2(0x2)
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/kubernetes/pkg/scheduler/framework/runtime/framework.go:1055 +0x2f0
  k8s.io/kubernetes/pkg/scheduler/framework/parallelize.Parallelizer.Until.func1(0x0?)
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/kubernetes/pkg/scheduler/framework/parallelize/parallelism.go:60 +0x46
  k8s.io/client-go/util/workqueue.ParallelizeUntil.func1()
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/client-go/util/workqueue/parallelizer.go:90 +0xfa
  created by k8s.io/client-go/util/workqueue.ParallelizeUntil in goroutine 444
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/client-go/util/workqueue/parallelizer.go:76 +0x1d6
  panic: runtime error: index out of range [0] with length 0 [recovered]
          panic: runtime error: index out of range [0] with length 0
  
  goroutine 596 [running]:
  k8s.io/apimachinery/pkg/util/runtime.HandleCrash({0x0, 0x0, 0x5?})
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/apimachinery/pkg/util/runtime/runtime.go:56 +0xcd
  panic({0x24d0340?, 0xc000f03350?})
          /usr/local/go/src/runtime/panic.go:914 +0x21f
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.GetNodeNetworkAvailableRate({0x2691d01, 0x1a}, {0xc00092322a, 0x5}, {0x0, 0x0})
          /go/src/sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go/get_prometheus_metrics.go:107 +0x57c
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.(*NeilatsRefactorScheduler).LBScore(0xc00063ae00, 0xc000f1fb00, {0xc00092322a, 0x5})
          /go/src/sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go/neilats_refactor.go:322 +0x55f
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.(*NeilatsRefactorScheduler).Score(0xc000d86f90?, {0x2a3bce8?, 0xc000a83360?}, 0xc000cab7f8?, 0xc000f1fb00?, {0xc00092322a?, 0x5?})
          /go/src/sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go/neilats_refactor.go:264 +0x25
  k8s.io/kubernetes/pkg/scheduler/framework/runtime.(*instrumentedScorePlugin).Score(0xc000125f60, {0x2a3bce8, 0xc000a83360}, 0xc0017d4da0?, 0x20?, {0xc00092322a, 0x5})
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/kubernetes/pkg/scheduler/framework/runtime/instrumented_plugins.go:82 +0x85
  k8s.io/kubernetes/pkg/scheduler/framework/runtime.(*frameworkImpl).runScorePlugin(0x2a40ea8?, {0x2a3bce8?, 0xc000a83360?}, {0x2a2ddc0?, 0xc000125f60?}, 0x4109a5?, 0xc0015a4db0?, {0xc00092322a?, 0x30?})
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/kubernetes/pkg/scheduler/framework/runtime/framework.go:1126 +0x2d8
  k8s.io/kubernetes/pkg/scheduler/framework/runtime.(*frameworkImpl).RunScorePlugins.func2(0x2)
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/kubernetes/pkg/scheduler/framework/runtime/framework.go:1055 +0x2f0
  k8s.io/kubernetes/pkg/scheduler/framework/parallelize.Parallelizer.Until.func1(0x0?)
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/kubernetes/pkg/scheduler/framework/parallelize/parallelism.go:60 +0x46
  k8s.io/client-go/util/workqueue.ParallelizeUntil.func1()
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/client-go/util/workqueue/parallelizer.go:90 +0xfa
  created by k8s.io/client-go/util/workqueue.ParallelizeUntil in goroutine 444
          /go/src/sigs.k8s.io/scheduler-plugins/vendor/k8s.io/client-go/util/workqueue/parallelizer.go:76 +0x1d6
  ~~~

  经过分析，通过如下日志内容

  ~~~sh
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.GetNodeNetworkAvailableRate({0x2691d01, 0x1a}, {0xc00092322a, 0x5}, {0x0, 0x0})
          /go/src/sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go/get_prometheus_metrics.go:107 +0x57c
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.(*NeilatsRefactorScheduler).LBScore(0xc00063ae00, 0xc000f1fb00, {0xc00092322a, 0x5})
          /go/src/sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go/neilats_refactor.go:322 +0x55f
  sigs.k8s.io/scheduler-plugins/pkg/neilats_refactor_go.(*NeilatsRefactorScheduler).Score(0xc000d86f90?, {0x2a3bce8?, 0xc000a83360?}, 0xc000cab7f8?, 0xc000f1fb00?, {0xc00092322a?, 0x5?})
  ~~~

  发现错误发生在函数`GetNodeNetworkAvailableRate`，即`get_prometheus_metrics.go:107`

  具体为：

  ~~~GO
  	// 分别获取每秒接受和发送的字节数
  	receiveBytes, _ := strconv.ParseFloat(receiveRateResult.Data.Result[0].Value[1].(string), 64)
  	transmitBytes, _ := strconv.ParseFloat(transmitRateResult.Data.Result[0].Value[1].(string), 64)
  ~~~

  原因是数组越界访问，但是经过实际Prometheus对于确定的PromQL的查询，是有结果的，出于健壮性，对其进行空数据返回检查

  ~~~GO
  	if len(transmitRateResult.Data.Result) == 0 {
  		log.Printf("get node network transmit rate response data length is 0!\n")
  		return 0, fmt.Errorf("get node network transmit rate response data length is 0!")
  	}
  	if len(receiveRateResult.Data.Result) == 0 {
  		log.Printf("get node network receive rate response data length is 0!\n")
  		return 0, fmt.Errorf("get network receive available rate response data length is 0!")
  	}
  ~~~

  同理对其他Prometheus查询指标也进行了空返回检查

* 修改2：

  除了添加了空返回检查，还将所有的错误返回改为有错误的直接返回错误，没错误的通过fmt.Errof返回自定义错误

  并且在发生错误的返回之前， 使用log.Printf向标准输出输出错误内容，上面的修改完的代码也能看得出来修改

### 25.8.28

> 这次是经过8.27修改后的一次上机测试

#### 修改1

* 错误描述

  ~~~sh
  I0828 02:10:53.515058       1 log.go:245] Total CPU of node: node3 is 8000.000000
  I0828 02:10:53.515192       1 log.go:245] Total CPU of node: node2 is 8000.000000
  I0828 02:10:53.515234       1 log.go:245] Total CPU of node: master is 8000.000000
  I0828 02:10:53.515293       1 log.go:245] Total CPU of node: node1 is 8000.000000
  CPU idle rate of node node3 is 0.8589966367716494
  CPU idle rate of node node2 is 0.9722409340967585
  CPU idle rate of node node1 is 0.9092410626799803
  CPU idle rate of node master is 0.9835813381068899
  I0828 02:10:53.517266       1 log.go:245] Total memory of node: node3 is 15991.917969 MB
  I0828 02:10:53.517449       1 log.go:245] Total memory of node: node2 is 15991.878906 MB
  I0828 02:10:53.517643       1 log.go:245] Total memory of node: node1 is 15991.882812 MB
  I0828 02:10:53.517996       1 log.go:245] Total memory of node: master is 15991.882812 MB
  I0828 02:10:53.518173       1 log.go:245] Memory available of node: node2 is 0.282104
  I0828 02:10:53.518374       1 log.go:245] Memory available of node: node3 is 0.218635
  I0828 02:10:53.518452       1 log.go:245] Memory available of node: node1 is 0.346946
  I0828 02:10:53.518952       1 log.go:245] Memory available of node: master is 0.833898
  I0828 02:10:53.519168       1 log.go:245] get node network receive rate response data length is 0!
  I0828 02:10:53.519187       1 log.go:245] get node node3 network available rate failed: get network receive available rate response data length is 0!
  I0828 02:10:53.519551       1 log.go:245] Network available of node: node2 is 3.459523
  I0828 02:10:53.519905       1 log.go:245] Network available of node: node1 is 3.495220
  I0828 02:10:53.520453       1 log.go:245] Network available of node: master is 3.607534
  I0828 02:10:53.520506       1 log.go:245] Total disk of node: node2 is 39001.281250 MB
  I0828 02:10:53.520678       1 log.go:245] Total disk of node: node1 is 39001.281250 MB
  I0828 02:10:53.521420       1 log.go:245] Total disk of node: master is 39001.281250 MB
  I0828 02:10:53.521511       1 log.go:245] Disk available of node: node2 is 21.728573
  I0828 02:10:53.521531       1 log.go:245] Node Name: node2 LB Score: -30932
  I0828 02:10:53.521625       1 log.go:245] Disk available of node: node1 is 24.030831
  I0828 02:10:53.521645       1 log.go:245] Node Name: node1 LB Score: -38254
  I0828 02:10:53.522250       1 log.go:245] Disk available of node: master is 23.928901
  I0828 02:10:53.522260       1 log.go:245] Node Name: master LB Score: -37085
  E0828 02:10:53.522806       1 schedule_one.go:154] "Error selecting node for pod" err="running Score plugins: plugin \"NeilatsRefactorScheduler\" failed with: get node node3 network available rate failed: get network receive available rate response data length is 0!" pod="k8s-learn/nginx-deployment-test-neilats-f7b7fdcfb-dldtp"
  E0828 02:10:53.523040       1 schedule_one.go:1004] "Error scheduling pod; retrying" err="running Score plugins: plugin \"NeilatsRefactorScheduler\" failed with: get node node3 network available rate failed: get network receive available rate response data length is 0!" pod="k8s-learn/nginx-deployment-test-neilats-f7b7fdcfb-dldtp"
  ~~~

  即找不到Node3的网络可用信息，后来发现，当其使用默认配置信息时，我并没有补全Node3的相关节点信息，导致LBScore函数访问Node3的network、disk余量信息时，找不到相应设备

* 原因分析与解决

    ~~~go
        NodeNetworkAvailableRate, err := GetNodeNetworkAvailableRate(neilats.config.PrometheusAddress, nodeName, neilats.config.NetworkDevice[nodeName])
    ~~~

    可以看到，这里需要传入Prometheus地址、节点名称还有对应节点名称的网络设备名称，但是默认配置中并没有添加Node3的配置，修改后的默认配置如下：

    ~~~go
        defaultConfig := NeilatsRefactorSchedulerArgs{
            PrometheusAddress: "http://192.168.3.226:31739", // 默认Prometheus地址
            NetworkDevice: map[string]string{ // 默认网络设备配置
                "master": "ens18",
                "node1":  "ens18",
                "node2":  "ens18",
                "node3":  "ens18",
            },
            StorageDevice: map[string]string{ // 默认存储设备配置
                "master": "/dev/mapper/ubuntu--vg-ubuntu--lv",
                "node1":  "/dev/mapper/ubuntu--vg-ubuntu--lv",
                "node2":  "/dev/mapper/ubuntu--vg-ubuntu--lv",
                "node3":  "/dev/mapper/ubuntu--vg-ubuntu--lv",
            },
            EnableSLA: false, // 默认不开启SLA约束要求
            // 键为Kubernetes中的节点名称
            KubeNodeAddressAndSecret: map[string]UserAddressSecretMap{
                "master": {"192.168.3.226", "mobisys912"},
                "node1":  {"192.168.3.229", "mobisys912"},
                "node2":  {"192.168.3.224", "mobisys912"},
                "node3":  {"192.168.3.228", "mobisys912"},
            }, // 由于默认不开启SLA约束，所以不要求kubeNodeAddressAndSecret参数
        }
    ~~~

#### 修改2

* 问题描述

  思来想去，虽然每个步骤如果出错，都有log.Printf一行debug信息，但是每个步骤执行成功也应该添加一行日志信息，因此许多位置都添加了执行成功输出日志信息的代码

#### 修改3

* 问题描述

  经过上述的代码修改，`EnableSLA`为false的情况已经正常通过了，其日志文件如下：

  ~~~sh
  I0828 03:26:50.567295       1 log.go:245] Total CPU of node: node1 is 8000.000000
  I0828 03:26:50.567313       1 log.go:245] Total CPU of node: node2 is 8000.000000
  I0828 03:26:50.567400       1 log.go:245] Total CPU of node: node3 is 8000.000000
  I0828 03:26:50.567465       1 log.go:245] Total CPU of node: master is 8000.000000
  CPU idle rate of node node2 is 0.9696893154836662
  CPU idle rate of node node1 is 0.9713260644112278
  CPU idle rate of node master is 0.9812877601253297
  I0828 03:26:50.569919       1 log.go:245] Total memory of node: node2 is 15991.878906 MB
  CPU idle rate of node node3 is 0.8605224110417699
  I0828 03:26:50.570269       1 log.go:245] Total memory of node: node1 is 15991.882812 MB
  I0828 03:26:50.571211       1 log.go:245] Total memory of node: master is 15991.882812 MB
  I0828 03:26:50.571235       1 log.go:245] Total memory of node: node3 is 15991.917969 MB
  I0828 03:26:50.571317       1 log.go:245] Memory available of node: node2 is 0.282392
  I0828 03:26:50.571696       1 log.go:245] Memory available of node: node1 is 0.366947
  I0828 03:26:50.572204       1 log.go:245] Memory available of node: master is 0.832659
  I0828 03:26:50.572582       1 log.go:245] Memory available of node: node3 is 0.240131
  I0828 03:26:50.573996       1 log.go:245] Network available of node: node1 is 3.523777
  I0828 03:26:50.574318       1 log.go:245] Network available of node: node2 is 3.460808
  I0828 03:26:50.575039       1 log.go:245] Total disk of node: node1 is 39001.281250 MB
  I0828 03:26:50.575119       1 log.go:245] Network available of node: master is 3.535055
  I0828 03:26:50.575154       1 log.go:245] Total disk of node: node2 is 39001.281250 MB
  I0828 03:26:50.576015       1 log.go:245] Network available of node: node3 is 3.210784
  I0828 03:26:50.576127       1 log.go:245] Total disk of node: master is 39001.281250 MB
  I0828 03:26:50.576338       1 log.go:245] Disk available of node: node1 is 23.367542
  I0828 03:26:50.576360       1 log.go:245] Node Name: node1 LB Score: -35930
  I0828 03:26:50.576774       1 log.go:245] Disk available of node: node2 is 21.176858
  I0828 03:26:50.576804       1 log.go:245] Node Name: node2 LB Score: -29288
  I0828 03:26:50.576961       1 log.go:245] Total disk of node: node3 is 39001.281250 MB
  I0828 03:26:50.577219       1 log.go:245] Disk available of node: master is 23.966090
  I0828 03:26:50.577233       1 log.go:245] Node Name: master LB Score: -37268
  I0828 03:26:50.578155       1 log.go:245] Disk available of node: node3 is 24.178733
  I0828 03:26:50.578169       1 log.go:245] Node Name: node3 LB Score: -39179
  I0828 03:26:50.580248       1 log.go:245] Total CPU of node: node2 is 8000.000000
  I0828 03:26:50.580343       1 log.go:245] Total CPU of node: node3 is 8000.000000
  I0828 03:26:50.580353       1 log.go:245] Total CPU of node: node1 is 8000.000000
  I0828 03:26:50.580389       1 log.go:245] Total CPU of node: master is 8000.000000
  CPU idle rate of node node3 is 0.8605224110417699
  CPU idle rate of node node2 is 0.9696893154836662
  CPU idle rate of node master is 0.9812877601253297
  CPU idle rate of node node1 is 0.9713260644112278
  I0828 03:26:50.583268       1 log.go:245] Total memory of node: node3 is 15991.917969 MB
  I0828 03:26:50.583298       1 log.go:245] Total memory of node: node2 is 15991.878906 MB
  I0828 03:26:50.583322       1 log.go:245] Total memory of node: node1 is 15991.882812 MB
  I0828 03:26:50.583371       1 log.go:245] Total memory of node: master is 15991.882812 MB
  I0828 03:26:50.584289       1 log.go:245] Memory available of node: node2 is 0.282392
  I0828 03:26:50.584324       1 log.go:245] Memory available of node: node3 is 0.240131
  I0828 03:26:50.584829       1 log.go:245] Memory available of node: node1 is 0.366947
  I0828 03:26:50.584947       1 log.go:245] Memory available of node: master is 0.832659
  I0828 03:26:50.588269       1 log.go:245] Network available of node: node3 is 3.210784
  I0828 03:26:50.588307       1 log.go:245] Network available of node: node2 is 3.460808
  I0828 03:26:50.588677       1 log.go:245] Network available of node: master is 3.535055
  I0828 03:26:50.589184       1 log.go:245] Network available of node: node1 is 3.523777
  I0828 03:26:50.589256       1 log.go:245] Total disk of node: node2 is 39001.281250 MB
  I0828 03:26:50.589313       1 log.go:245] Total disk of node: node3 is 39001.281250 MB
  I0828 03:26:50.589821       1 log.go:245] Total disk of node: master is 39001.281250 MB
  I0828 03:26:50.590261       1 log.go:245] Total disk of node: node1 is 39001.281250 MB
  I0828 03:26:50.590353       1 log.go:245] Disk available of node: node2 is 21.176858
  I0828 03:26:50.590366       1 log.go:245] Node Name: node2 LB Score: -29288
  I0828 03:26:50.590694       1 log.go:245] Disk available of node: node3 is 24.178733
  I0828 03:26:50.590748       1 log.go:245] Node Name: node3 LB Score: -39179
  I0828 03:26:50.591108       1 log.go:245] Disk available of node: master is 23.966090
  I0828 03:26:50.591142       1 log.go:245] Node Name: master LB Score: -37268
  I0828 03:26:50.591171       1 log.go:245] Disk available of node: node1 is 23.367542
  I0828 03:26:50.591183       1 log.go:245] Node Name: node1 LB Score: -35930
  ~~~

  调度结果如下：

  ~~~sh
  (base) root@master:~/work/SchedulingFramework# kubectl get pod -n k8s-learn -o wide
  NAME                                               READY   STATUS    RESTARTS      AGE   IP             NODE     NOMINATED NODE   READINESS GATES
  my-custom-metrics-app-deployment-5976cd49c-5wngs   1/1     Running   0             45d   10.244.3.184   node3    <none>           <none>
  my-custom-metrics-app-deployment-5976cd49c-dr77c   1/1     Running   0             45d   10.244.3.182   node3    <none>           <none>
  neilats-refactor-controller-65d7f8c6d7-kw97q       1/1     Running   0             89s   10.244.2.183   node2    <none>           <none>
  neilats-refactor-scheduler-779c969d89-4rgrh        1/1     Running   0             89s   10.244.2.184   node2    <none>           <none>
  nginx-deployment-7c79c4bf97-2qwvz                  1/1     Running   4 (62d ago)   83d   10.244.3.81    node3    <none>           <none>
  nginx-deployment-7c79c4bf97-5rrkc                  1/1     Running   5 (46h ago)   83d   10.244.1.60    node1    <none>           <none>
  nginx-deployment-test-neilats-f7b7fdcfb-krqsf      1/1     Running   0             45s   10.244.0.26    master   <none>           <none>
  nginx-deployment-test-neilats-f7b7fdcfb-vrkvd      1/1     Running   0             45s   10.244.2.185   node2    <none>           <none>
  ~~~

  但是不知道为什么，好像传入的自定义参数并没有被识别到，调度器并没有接收到传入的自定义参数，具有自定义参数的values.yaml文件如下

  ~~~yaml
  # Default values for scheduler-plugins-as-a-second-scheduler.
  # This is a YAML-formatted file.
  # Declare variables to be passed into your templates.
  
  scheduler:
    name: neilats-refactor-scheduler
    image: wuyong7240/scheduler-plugins/neilats-refactor-scheduler:v2.2
    replicaCount: 1
    leaderElect: false
    nodeSelector: {}
    affinity: {}
    tolerations: []
  
  controller:
    name: neilats-refactor-controller
    image: wuyong7240/scheduler-plugins/neilats-refactor-controller:v2.2
    replicaCount: 1
    nodeSelector: {}
    affinity: {}
    tolerations: []
  
  # LoadVariationRiskBalancing and TargetLoadPacking are not enabled by default
  # as they need extra RBAC privileges on metrics.k8s.io.
  
  plugins:
    enabled: ["Coscheduling","CapacityScheduling","NodeResourceTopologyMatch","NodeResourcesAllocatable","NeilatsRefactorScheduler"]
    disabled: ["PrioritySort"] # only in-tree plugins need to be defined here
  
  # Customize the enabled plugins' config.
  # Refer to the "pluginConfig" section of manifests/<plugin>/scheduler-config.yaml.
  # For example, for Coscheduling plugin, you want to customize the permit waiting timeout to 10 seconds:
  pluginConfig:
  - name: Coscheduling
    args:
      permitWaitingTimeSeconds: 10 # default is 60
  - name: NeilatsRefactorScheudler
    args:
      prometheusAddress: "http://http://192.168.3.226:31739"
      networkDevice:
        master: "ens18"
        node1: "ens18"
        node2: "ens18"
        node3: "ens18"
      storageDevice:
        master: "/dev/mapper/ubuntu--vg-ubuntu--lv"
        node1: "/dev/mapper/ubuntu--vg-ubuntu--lv"
        node2: "/dev/mapper/ubuntu--vg-ubuntu--lv"
        node3: "/dev/mapper/ubuntu--vg-ubuntu--lv"
      enableSLA: true
      kubeNodeAddressAndSecret:
        master:
          nodeAddress: "192.168.3.226"
          nodeSecret: "mobisys912"
        node1:
          nodeAddress: "192.168.3.229"
          nodeSecret: "mobisys912"
        node2:
          nodeAddress: "192.168.3.224"
          nodeSecret: "mobisys912"
        node3:
          nodeAddress: "192.168.3.228"
          nodeSecret: "mobisys912"
  # Or, customize the other plugins
  # - name: NodeResourceTopologyMatch
  #   args:
  #     scoringStrategy:
  #       type: MostAllocated # default is LeastAllocated
  #- name: SySched
  #  args:
  #    defaultProfileNamespace: "default"
  #    defaultProfileName: "full-seccomp"
  ~~~

* 原因分析与解决

  发现是`values.yaml`文件中的自定义调度插件配置名称写错了`NeilatsRefactorScheudler`，应该是`NeilatsRefactorScheduler`

  

#### 修改4

* 问题描述

  更改后，重新部署自定义调度插件，发现其出现`CrashLoopOff`错误

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins-my/manifests/install/charts# kubectl get pod -n k8s-learn -o wide
  NAME                                               READY   STATUS             RESTARTS       AGE    IP             NODE     NOMINATED NODE   READINESS GATES
  my-custom-metrics-app-deployment-5976cd49c-5wngs   1/1     Running            0              45d    10.244.3.184   node3    <none>           <none>
  my-custom-metrics-app-deployment-5976cd49c-dr77c   1/1     Running            0              45d    10.244.3.182   node3    <none>           <none>
  neilats-refactor-controller-7f877f5cbf-852jj       1/1     Running            0              13s    10.244.2.197   node2    <none>           <none>
  neilats-refactor-scheduler-6667f9486c-c68vz        0/1     CrashLoopBackOff   1 (11s ago)    13s    10.244.2.198   node2    <none>           <none>
  ~~~

  查看日志，发现其发生如下错误

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins-my/manifests/install/charts# kubectl logs neilats-refactor-scheduler-6667f9486c-c68vz -n k8s-learn
  I0828 09:10:41.858497       1 serving.go:348] Generated self-signed cert in-memory
  W0828 09:10:41.859182       1 client_config.go:618] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
  I0828 09:10:42.069019       1 log.go:245] Custom Neilats Args Detected!
  I0828 09:10:42.069032       1 log.go:245] want args to be of type NeilatsRefactorSchedulerArgs, got *runtime.Unknown
  E0828 09:10:42.069046       1 run.go:74] "command failed" err="initializing profiles: creating profile for scheduler name neilats-refactor-scheduler: initializing plugin \"NeilatsRefactorScheduler\": want args to be of type NeilatsRefactorSchedulerArgs, got *runtime.Unknown"
  ~~~

* 原因分析

  1. 参数结构体应该包含`metav1.TypeMeta`，并应该跟有tag`json:",inline"`

     ~~~go
     // NeilatsRefactorSchedulerArgs holds args needed by InitPlugin.
     type NeilatsRefactorSchedulerArgs struct {
         metav1.TypeMeta `json:",inline"`
     
         PrometheusAddress        string                          `json:"prometheusAddress"`
         NetworkDevice            map[string]string               `json:"networkDevice"`
         StorageDevice            map[string]string               `json:"storageDevice"`
         EnableSLA                bool                            `json:"enableSLA"`
         KubeNodeAddressAndSecret map[string]UserAddressSecretMap `json:"kubeNodeAddressAndSecret"`
     }
     ~~~

     

  2. DeepCopy实现有问题，缺少`+k8s:deepcopy-gen`注释

     ~~~go
     // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
     
     // NeilatsRefactorSchedulerArgs ...
     type NeilatsRefactorSchedulerArgs struct {
         ...
     }
     ~~~

     这个注释必须有，因为`k8s.io/apimachinery`的反射机制会检查接口实现

  3. 结构体没有被注册到Kubernetes的Scheme中，即参数结构体应该放在`scheduler-plugins/apis/config/types.go`中，deepcopy函数应该最好使用`./hack/update-codegen.sh`来自动生成

     自己实现要放在`scheduler-plugins/apis/config/zz_generated.deepcopy.go`中

     因为Kubernetes会去Scheme中查找对应的Args类型，然后使用scheme反序列化yaml到GO结构体中

* 修改

  涉及到的文件有：`scheduler-plugins/apis/config/types.go`、`scheduler-plugins/apis/config/zz_generated.deepcopy.go`、`scheduler-plugins/pkg/neilats_refactor_go/neilats_refactor.go`、`scheduler-plugins/pkg/neilats_refactor_go/neilats_unit_test.go``scheduler-plugins/pkg/neilats_refactor_go/build-rtt-matrix.go``scheduler-plugins/pkg/neilats_refactor_go/latency_run_get.go`

### 25.9.1

#### 修改5

通过使用clash-for-linux的方式，在Ubuntu中成功clone了原项目，并成功利用`hack/update-codegen.sh`针对自定义参数类型自动生成了Deepcopy等接口函数，具体方法如下

1. clone项目到本地

   ~~~sh
   git clone https://github.com/kubernetes-sigs/scheduler-plugins.git
   ~~~

2. 根据使用的集群版本，切换到相应分支

   ~~~sh
   git branch -a
   (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins# git branch -a
   * master
     remotes/origin/HEAD -> origin/master
     remotes/origin/master
     remotes/origin/release-1.18
     remotes/origin/release-1.19
     remotes/origin/release-1.20
     remotes/origin/release-1.21
     remotes/origin/release-1.22
     remotes/origin/release-1.23
     remotes/origin/release-1.24
     remotes/origin/release-1.25
     remotes/origin/release-1.26
     remotes/origin/release-1.27
     remotes/origin/release-1.28
     remotes/origin/release-1.29
     remotes/origin/release-1.30
     remotes/origin/release-1.31
     remotes/origin/release-1.32
   (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins# git checkout release-1.28
   branch 'release-1.28' set up to track 'origin/release-1.28'.
   Switched to a new branch 'release-1.28'
   ~~~

3. 替换相应文件，`scheduler-plugins/pkg/neilats_refactor_go/*.go`、`scheduler-plugins/apis/config/types.go`、`scheduler-plugins/cmd/scheduler/main.go`

   然后执行`hack/update-codegen.sh`

4. 发生错误

   ~~~sh
   (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins# ./hack/update-codegen.sh
   WARNING: generate-internal-groups.sh is deprecated.
   WARNING: Please use k8s.io/code-generator/kube_codegen.sh instead.
   
   Generating deepcopy funcs
   Generating defaulters
   Generating conversions
   WARNING: generate-groups.sh is deprecated.
   WARNING: Please use k8s.io/code-generator/kube_codegen.sh instead.
   
   WARNING: Specifying "all" as a generator is deprecated.
   WARNING: Please list the specific generators needed.
   WARNING: "all" is now an alias for "applyconfiguration,client,deepcopy,informer,lister"; new code generators WILL NOT be added to this set
   
   WARNING: generate-internal-groups.sh is deprecated.
   WARNING: Please use k8s.io/code-generator/kube_codegen.sh instead.
   
   Generating deepcopy funcs
   Generating apply configuration for scheduling:v1alpha1 at sigs.k8s.io/scheduler-plugins/pkg/generated/applyconfiguration
   Generating clientset for scheduling:v1alpha1 at sigs.k8s.io/scheduler-plugins/pkg/generated/clientset
   Generating listers for scheduling:v1alpha1 at sigs.k8s.io/scheduler-plugins/pkg/generated/listers
   Generating informers for scheduling:v1alpha1 at sigs.k8s.io/scheduler-plugins/pkg/generated/informers
   panic: runtime error: invalid memory address or nil pointer dereference [recovered]
           panic: runtime error: invalid memory address or nil pointer dereference
   [signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0xa0969e]
   
   goroutine 121 [running]:
   go/types.(*Checker).handleBailout(0xc001d2a000, 0xc001435d78)
           /usr/local/go/src/go/types/check.go:434 +0x88
   panic({0xc33480?, 0x13b6220?})
           /usr/local/go/src/runtime/panic.go:792 +0x132
   go/types.(*StdSizes).Sizeof(0x0, {0xe417e0, 0x13bf120})
           /usr/local/go/src/go/types/sizes.go:229 +0x31e
   go/types.(*Config).sizeof(...)
           /usr/local/go/src/go/types/sizes.go:334
   go/types.representableConst.func1(...)
           /usr/local/go/src/go/types/const.go:77
   go/types.representableConst({0xe48170, 0xe5eac0}, 0xc001d2a000, 0x13bf120, 0x0)
           /usr/local/go/src/go/types/const.go:93 +0x1e9
   go/types.(*Checker).arrayLength(0xc001d2a000, {0xe466b0, 0xc0017df060?})
           /usr/local/go/src/go/types/typexpr.go:536 +0x1ce
   go/types.(*Checker).typInternal(0xc001d2a000, {0xe44d00, 0xc0017f7560}, 0x0)
           /usr/local/go/src/go/types/typexpr.go:316 +0x3f5
   go/types.(*Checker).definedType(0xc001d2a000, {0xe44d00, 0xc0017f7560}, 0xc001435490?)
           /usr/local/go/src/go/types/typexpr.go:193 +0x2f
   go/types.(*Checker).varType(0xc001d2a000, {0xe44d00, 0xc0017f7560})
           /usr/local/go/src/go/types/typexpr.go:158 +0x25
   go/types.(*Checker).structType(0xc001d2a000, 0xc001d247b0, 0xc001d247b0?)
           /usr/local/go/src/go/types/struct.go:114 +0x187
   go/types.(*Checker).typInternal(0xc001d2a000, {0xe44c70, 0xc0017cfba8}, 0xc001d22140)
           /usr/local/go/src/go/types/typexpr.go:333 +0x245
   go/types.(*Checker).definedType(0xc001d2a000, {0xe44c70, 0xc0017cfba8}, 0xd17478?)
           /usr/local/go/src/go/types/typexpr.go:193 +0x2f
   go/types.(*Checker).typeDecl(0xc001d2a000, 0xc001d22140, 0xc0017dda40, 0x0)
           /usr/local/go/src/go/types/decl.go:635 +0x526
   go/types.(*Checker).objDecl(0xc001d2a000, {0xe4e1c0, 0xc001d22140}, 0x0)
           /usr/local/go/src/go/types/decl.go:190 +0x9c5
   go/types.(*Checker).packageObjects(0xc001d2a000)
           /usr/local/go/src/go/types/resolver.go:678 +0x3be
   go/types.(*Checker).checkFiles(0xc001d2a000, {0xc0019d21a0?, 0x9ca17b?, 0xc001435da8?})
           /usr/local/go/src/go/types/check.go:489 +0x2a6
   go/types.(*Checker).Files(0xc000401090?, {0xc0019d21a0?, 0xc00131b1a0?, 0x6?})
           /usr/local/go/src/go/types/check.go:452 +0x75
   sigs.k8s.io/controller-tools/pkg/loader.(*loader).typeCheck(0xc00034c780, 0xc0005ad060)
           /root/Golang/code/go/pkg/mod/sigs.k8s.io/controller-tools@v0.11.1/pkg/loader/loader.go:286 +0x34a
   sigs.k8s.io/controller-tools/pkg/loader.(*Package).NeedTypesInfo(0xc0005ad060)
           /root/Golang/code/go/pkg/mod/sigs.k8s.io/controller-tools@v0.11.1/pkg/loader/loader.go:99 +0x39
   sigs.k8s.io/controller-tools/pkg/loader.(*TypeChecker).check(0xc00016bc20, 0xc0005ad060)
           /root/Golang/code/go/pkg/mod/sigs.k8s.io/controller-tools@v0.11.1/pkg/loader/refs.go:268 +0x297
   sigs.k8s.io/controller-tools/pkg/loader.(*TypeChecker).check.func1(0x66?)
           /root/Golang/code/go/pkg/mod/sigs.k8s.io/controller-tools@v0.11.1/pkg/loader/refs.go:262 +0x4d
   created by sigs.k8s.io/controller-tools/pkg/loader.(*TypeChecker).check in goroutine 48
           /root/Golang/code/go/pkg/mod/sigs.k8s.io/controller-tools@v0.11.1/pkg/loader/refs.go:260 +0x1aa
   (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins#
   (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins#
   ~~~

   经过确认，是`controller-tools`某些版本中已知的bug，尤其是在使用了不兼容的Go版本、`controller-gen`旧版本，这说明`controller-tools v0.11.1`这个版本，已知在Go 1.20+上存在兼容性问题

5. 解决方法

   此处采用的方法是将`controller-tools`升级到兼容版本

   ~~~sh
   # 升级到 v0.12.1 或更高（推荐 v0.14.0）
   go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.14.0
   ~~~

   注意，此处虽然安装了v0.14.0版本，但是脚本中并未采用，需要修改`hack/update-codegen.sh`中关于`controller-tools`的版本

   ~~~sh
   #!/usr/bin/env bash
   
   # Copyright 2017 The Kubernetes Authors.
   #
   # Licensed under the Apache License, Version 2.0 (the "License");
   # you may not use this file except in compliance with the License.
   # You may obtain a copy of the License at
   #
   #     http://www.apache.org/licenses/LICENSE-2.0
   #
   # Unless required by applicable law or agreed to in writing, software
   # distributed under the License is distributed on an "AS IS" BASIS,
   # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   # See the License for the specific language governing permissions and
   # limitations under the License.
   
   set -o errexit
   set -o nounset
   set -o pipefail
   
   SCRIPT_ROOT=$(dirname "${BASH_SOURCE[@]}")/..
   
   TOOLS_DIR=$(realpath ./hack/tools)
   TOOLS_BIN_DIR="${TOOLS_DIR}/bin"
   GO_INSTALL=$(realpath ./hack/go-install.sh)
   CONTROLLER_GEN_VER=v0.14.0	# 关于使用的代码生成工具版本
   CONTROLLER_GEN_BIN=controller-gen
   CONTROLLER_GEN=${TOOLS_BIN_DIR}/${CONTROLLER_GEN_BIN}-${CONTROLLER_GEN_VER}
   # Need v1 to support defaults in CRDs, unfortunately limiting us to k8s 1.16+
   CRD_OPTIONS="crd:crdVersions=v1"
   
   GOBIN=${TOOLS_BIN_DIR} ${GO_INSTALL} sigs.k8s.io/controller-tools/cmd/controller-gen ${CONTROLLER_GEN_BIN} ${CONTROLLER_GEN_VER}
   
   CODEGEN_PKG=${CODEGEN_PKG:-$(cd "${SCRIPT_ROOT}"; ls -d -1 ./vendor/k8s.io/code-generator 2>/dev/null || echo ../code-generator)}
   
   bash "${CODEGEN_PKG}"/generate-internal-groups.sh \
     "deepcopy,conversion,defaulter" \
     sigs.k8s.io/scheduler-plugins/pkg/generated \
     sigs.k8s.io/scheduler-plugins/apis \
     sigs.k8s.io/scheduler-plugins/apis \
     "config:v1,v1beta3" \
     --trim-path-prefix sigs.k8s.io/scheduler-plugins \
     --output-base "./" \
     --go-header-file "${SCRIPT_ROOT}"/hack/boilerplate/boilerplate.generatego.txt
   
   bash "${CODEGEN_PKG}"/generate-groups.sh \
     all \
     sigs.k8s.io/scheduler-plugins/pkg/generated \
     sigs.k8s.io/scheduler-plugins/apis \
     "scheduling:v1alpha1" \
     --go-header-file "${SCRIPT_ROOT}"/hack/boilerplate/boilerplate.generatego.txt
   
   ${CONTROLLER_GEN} object:headerFile="hack/boilerplate/boilerplate.generatego.txt" \
     paths="./apis/scheduling/..."
   
   ${CONTROLLER_GEN} ${CRD_OPTIONS} rbac:roleName=work-manager webhook \
     paths="./apis/scheduling/..." \
     output:crd:artifacts:config=config/crd/bases
   ~~~

   而后就可以正常生成代码了

#### 修改6

* 问题描述与解决

  而后还是发现出现了一样的问题

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins/manifests/install/charts# kubectl logs neilats-refactor-scheduler-76476bb88f-5m5gt -n k8s-learn
  I0901 08:12:42.629947       1 serving.go:348] Generated self-signed cert in-memory
  W0901 08:12:42.630578       1 client_config.go:618] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
  I0901 08:12:42.880229       1 capacity_scheduling.go:190] "CapacityScheduling start"
  I0901 08:12:42.880354       1 log.go:245] Custom Neilats Args Detected!
  I0901 08:12:42.880493       1 log.go:245] want args to be of type NeilatsRefactorSchedulerArgs, got *runtime.Unknown
  E0901 08:12:42.880534       1 run.go:74] "command failed" err="initializing profiles: creating profile for scheduler name neilats-refactor-scheduler: initializing plugin \"NeilatsRefactorScheduler\": want args to be of type NeilatsRefactorSchedulerArgs, got *runtime.Unknown"
  ~~~

  后来发现，不仅需要自动生成DeepCopy等接口函数，还需要在`scheduler-plugins/apis/config/register.go`中进行类型注册

  并且，除了/config文件夹下的`register.go`和`types.go`，还有`scheduler-plugins/apis/config/v1`、`scheduler-plugins/apis/config/v1beta3`、文件夹下相对应的两个文件，都要有

  但是有一个前提，需要先在对应的`types.go`文件中添加自定义参数类型，使用代码生成工具生成对应的接口函数之后，才能在`register.go`中注册相应自定义参数类型

  示例`register.go`文件如下

  ~~~go
  /*
  Copyright 2020 The Kubernetes Authors.
  
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  
      http://www.apache.org/licenses/LICENSE-2.0
  
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
  */
  
  package config
  
  import (
          "k8s.io/apimachinery/pkg/runtime"
          "k8s.io/apimachinery/pkg/runtime/schema"
          schedconfig "k8s.io/kubernetes/pkg/scheduler/apis/config"
  )
  
  // SchemeGroupVersion is group version used to register these objects
  var SchemeGroupVersion = schema.GroupVersion{Group: schedconfig.GroupName, Version: runtime.APIVersionInternal}
  
  var (
          localSchemeBuilder = &schedconfig.SchemeBuilder
          // AddToScheme is a global function that registers this API group & version to a scheme
          AddToScheme = localSchemeBuilder.AddToScheme
  )
  
  // addKnownTypes registers known types to the given scheme
  func addKnownTypes(scheme *runtime.Scheme) error {
          scheme.AddKnownTypes(SchemeGroupVersion,
                  &CoschedulingArgs{},
                  &NeilatsRefactorSchedulerArgs{},	// 这里注册了自定义参数类型，仅仅修改此处即可
                  &NodeResourcesAllocatableArgs{},
                  &TargetLoadPackingArgs{},
                  &LoadVariationRiskBalancingArgs{},
                  &LowRiskOverCommitmentArgs{},
                  &NodeResourceTopologyMatchArgs{},
                  &PreemptionTolerationArgs{},
                  &TopologicalSortArgs{},
                  &NetworkOverheadArgs{},
                  &SySchedArgs{},
          )
          return nil
  }
  
  func init() {
          // We only register manually written functions here. The registration of the
          // generated functions takes place in the generated files. The separation
          // makes the code compile even when the generated files are missing.
          localSchemeBuilder.Register(addKnownTypes)
  }
  ~~~

#### 修改7

* 问题描述与修改

  在经历过上述的自定义参数结构体在`register.go`中注册后，成功识别了传入的yaml参数，但同时也遇到了新的问题，输出的日志如下：

  ~~~sh
  E0901 10:49:36.611663       1 run.go:74] "command failed" err="strict decoding error: decoding .profiles[0].pluginConfig[1]: strict decoding error: decoding args for plugin NeilatsRefactorScheduler: strict decoding error: unknown field \"kubeNodeAddressAndSecret.master.nodeAddress\", unknown field \"kubeNodeAddressAndSecret.master.nodeSecret\", unknown field \"kubeNodeAddressAndSecret.node1.nodeAddress\", unknown field \"kubeNodeAddressAndSecret.node1.nodeSecret\", unknown field \"kubeNodeAddressAndSecret.node2.nodeAddress\", unknown field \"kubeNodeAddressAndSecret.node2.nodeSecret\", unknown field \"kubeNodeAddressAndSecret.node3.nodeAddress\", unknown field \"kubeNodeAddressAndSecret.node3.nodeSecret\""
  ~~~

  是一个 **Kubernetes API 严格解码（strict decoding）错误**，意味着你传递给插件的配置字段 **无法被正确反序列化到 Go 结构体中**。

  **错误信息表明：Kubernetes 的解码器在尝试将 YAML 解析为 `NeilatsRefactorSchedulerArgs` 时，认为 `nodeAddress` 和 `nodeSecret` 是“未知字段”**。

  这说明：**结构体字段的 `json` tag 与 YAML 字段名不匹配**，或结构体未正确注册。

  Go 默认通过字段名导出（首字母大写），但 **JSON/YAML 反序列化依赖 `json` tag**。如果没有 `json` tag，Go 使用字段名原样映射（即 `NodeAddress` → `NodeAddress`），而你的 YAML 写的是 `nodeAddress`（小写 `n`）。

  由于结构体中没有`json`tag，Go默认期望

  ~~~yaml
  "NodeAddress": "192.168.3.226"
  ~~~

  解决方法就是在结构体中添加`json`tag，与yaml中的小写字段名匹配

  ~~~go
  type UserAddressSecretMap struct {
  	NodeAddress string `json:"nodeAddress"`
  	NodeSecret  string `json:"nodeSecret"`
  }
  ~~~

  经过上述修改，成功解决自定义参数通过yaml传入的问题了，输出日志如下

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins/manifests/install/charts# kubectl logs neilats-refactor-scheduler-9b5955d75-zf55h -n k8s-learn
  I0901 11:05:41.767563       1 serving.go:348] Generated self-signed cert in-memory
  W0901 11:05:41.768326       1 client_config.go:618] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
  I0901 11:05:41.993284       1 capacity_scheduling.go:190] "CapacityScheduling start"
  I0901 11:05:41.993319       1 log.go:245] Custom Neilats Args Detected!
  I0901 11:05:41.993325       1 log.go:245] Config EnableSLA is True!
  I0901 11:05:41.993332       1 log.go:245] Init Node "master" Latency Test Tool
  I0901 11:05:45.702194       1 log.go:245] Command executed successfully
  I0901 11:05:45.702276       1 log.go:245] Install Latency Test Tool Success
  I0901 11:05:45.835907       1 log.go:245] Command executed successfully
  I0901 11:05:45.836015       1 log.go:245] Node "master" Create Latency Test Directory Success
  I0901 11:05:45.964716       1 log.go:245] Init Node "node1" Latency Test Tool
  I0901 11:05:50.188561       1 log.go:245] Command executed successfully
  I0901 11:05:50.188637       1 log.go:245] Install Latency Test Tool Success
  I0901 11:05:50.321908       1 log.go:245] Command executed successfully
  I0901 11:05:50.322004       1 log.go:245] Node "node1" Create Latency Test Directory Success
  I0901 11:05:50.487983       1 log.go:245] Init Node "node2" Latency Test Tool
  I0901 11:05:53.658316       1 log.go:245] Command executed successfully
  I0901 11:05:53.658382       1 log.go:245] Install Latency Test Tool Success
  I0901 11:05:53.791554       1 log.go:245] Command executed successfully
  I0901 11:05:53.791596       1 log.go:245] Node "node2" Create Latency Test Directory Success
  I0901 11:05:53.924766       1 log.go:245] Init Node "node3" Latency Test Tool
  I0901 11:05:58.274997       1 log.go:245] Command executed successfully
  I0901 11:05:58.275122       1 log.go:245] Install Latency Test Tool Success
  I0901 11:05:58.417181       1 log.go:245] Command executed successfully
  I0901 11:05:58.417277       1 log.go:245] Node "node3" Create Latency Test Directory Success
  I0901 11:05:58.559265       1 log.go:245] Init Node Latency Test Tool Success!
  I0901 11:05:58.725751       1 log.go:245] Command executed successfully
  I0901 11:05:58.725810       1 log.go:245] Success Start Node "node2" Latency Test!
  I0901 11:05:58.906171       1 log.go:245] Command executed successfully
  I0901 11:05:58.906235       1 log.go:245] Success Start Node "node3" Latency Test!
  I0901 11:05:59.076401       1 log.go:245] Command executed successfully
  I0901 11:05:59.076481       1 log.go:245] Success Start Node "master" Latency Test!
  I0901 11:05:59.250246       1 log.go:245] Command executed successfully
  I0901 11:05:59.250320       1 log.go:245] Success Start Node "node1" Latency Test!
  I0901 11:05:59.250330       1 log.go:245] Start Latency Test Success!
  I0901 11:05:59.250334       1 log.go:245] Final Neilats Config is:
  I0901 11:05:59.250362       1 log.go:245] {{ } http://192.168.3.226:31739 map[master:ens18 node1:ens18 node2:ens18 node3:ens18] map[master:/dev/mapper/ubuntu--vg-ubuntu--lv node1:/dev/mapper/ubuntu--vg-ubuntu--lv node2:/dev/mapper/ubuntu--vg-ubuntu--lv node3:/dev/mapper/ubuntu--vg-ubuntu--lv] true map[master:{192.168.3.226 mobisys912} node1:{192.168.3.229 mobisys912} node2:{192.168.3.224 mobisys912} node3:{192.168.3.228 mobisys912}]}
  I0901 11:05:59.250927       1 server.go:154] "Starting Kubernetes Scheduler" version="v0.28.9"
  I0901 11:05:59.250941       1 server.go:156] "Golang settings" GOGC="" GOMAXPROCS="" GOTRACEBACK=""
  I0901 11:05:59.254007       1 requestheader_controller.go:169] Starting RequestHeaderAuthRequestController
  I0901 11:05:59.254024       1 configmap_cafile_content.go:202] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
  I0901 11:05:59.254024       1 configmap_cafile_content.go:202] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::client-ca-file"
  I0901 11:05:59.254056       1 shared_informer.go:311] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
  I0901 11:05:59.254058       1 shared_informer.go:311] Waiting for caches to sync for RequestHeaderAuthRequestController
  I0901 11:05:59.254058       1 shared_informer.go:311] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
  I0901 11:05:59.254496       1 secure_serving.go:213] Serving securely on [::]:10259
  I0901 11:05:59.254561       1 tlsconfig.go:240] "Starting DynamicServingCertificateController"
  I0901 11:05:59.354403       1 shared_informer.go:318] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
  I0901 11:05:59.354405       1 shared_informer.go:318] Caches are synced for RequestHeaderAuthRequestController
  I0901 11:05:59.354483       1 shared_informer.go:318] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
  ~~~

  



# Version2.0迭代

> 之前的1.0版本，仅仅包含了LBScore的节点负载均衡得分计算，并不包含论文中的关于网络链路延迟的ADF平稳性得分计算、未来通信链路预测并计算未来通信链路得分的计算
>
> 因此，在2.0版本中添加后面的两个模块
>
> 由于网络链路ADF平稳性得分计算需要使用到Python的包、未来通信链路得分也需要使用LSTM对未来通信链路延迟值进行预测，并且考虑到在Neilats的Scheduling-framework中自己实现比较麻烦，且使得调度插件繁重，因此，最终决定后两个模块均使用Python实现，以Operator形式部署在集群中，通过提供Sevice的形式，为Neilats调度插件提供服务
>
> 关于后两个模块涉及到的部分，请查看如下文档
>
> 1. Obsidian文档
>
>    [[LSTM长短期记忆网络：用于时间序列预测]]
>
>    [[[将LSTM时间序列预测服务构建为应用+打包为Docker镜像运行]]
>
> 2. Github项目链接
>
>    https://github.com/WyYong7240/Neilats_LSTM_ADF_Module
>
>    https://github.com/WyYong7240/LSTMServerOperator

## 代码变更

### 1.自定义参数结构体

修改`/apis/config/types.go`、`/apis/config/v1/types.go`、`/apis/config/v1beta3/types.go`

新结构体

~~~go
type NeilatsRefactorSchedulerArgs struct {
	// 如果自定义调度器有参数，需要参数结构体添加该内联类型，是Kubernetes库的要求
	metav1.TypeMeta          `json:",inline"`
	PrometheusAddress        string                          `json:"prometheusAddress"`
	NetworkDevice            map[string]string               `json:"networkDevice"`
	StorageDevice            map[string]string               `json:"storageDevice"`
	EnableSLA                bool                            `json:"enableSLA"`
	KubeNodeAddressAndSecret map[string]UserAddressSecretMap `json:"kubeNodeAddressAndSecret"`
	LstmAdfModuleAddress     string                          `json:"lstmAdfModuleAddress"`
}
~~~

增加了LSTM、ADF和未来得分计算模块的服务地址参数

然后使用`./hack/update-codegen.sh`更新DeepCopy等实现

### 2.自定义插件扩展点

由于多了ADFScore、FutureScore两个得分，**论文中需要对这三个得分先进行节点间的归一化，然后再求三种得分的均方，作为节点的最终得分**

但是，由于Schedule-Framework的Score扩展点的局限，**在Score扩展点中，只能访问到一个节点的信息，无法进行节点间的分数归一化**，因此，新增`PreScore`扩展点

`PreScore`扩展点，**可以访问当前所有节点的信息，因此可以在Score扩展点之前预先计算好所有节点的三种分数，并且做好节点间的归一化操作**

* PreScore扩展点实现

  ~~~go
  // 预先计算每个节点的三种得分
  func (neilats *NeilatsRefactorScheduler) PreScore(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodes []*v1.Node) *framework.Status {
  	// 初始化各个节点不同得分Map
  	nodeNum := len(nodes)
  	LBScoreMap = make(map[string]float64, nodeNum)
  	ADFScoreMap = make(map[string]float64, nodeNum)
  	FutureScoreMap = make(map[string]float64, nodeNum)
  
  	// 找出各个分数值的极值，用于归一化
  	var LBMax, ADFMax, FutureMax float64 = float64(math.MinInt64), float64(math.MinInt64), float64(math.MinInt64)
  	var LBMin, ADFMin, FutureMin float64 = math.MaxFloat64, math.MaxFloat64, math.MaxFloat64
  
  	for _, node := range nodes {
  		// 预先计算各个节点的负载均衡得分
  		nodeLBscore, err := neilats.LBScore(pod, node.Name)
  		if err != nil {
  			log.Printf("Pod %s compute Node %s LBScore Failed: %v", pod.Name, node.Name, err)
  		}
  		LBScoreMap[node.Name] = float64(nodeLBscore)
  		LBMin = math.Min(LBMin, nodeLBscore)
  		LBMax = math.Max(LBMax, nodeLBscore)
  
  		if neilats.config.EnableSLA {
  			// 安全检查已经在PreFilter阶段通过了，不用检查了
  			neiNodeValue, _ := pod.GetLabels()["nei_node"]
  			SLAConstraintValue, _ := strconv.ParseFloat(pod.GetLabels()["sla"], 64)
  			// 获取ADF分数
  			ADFScore, err := neilats.ADFScore(neiNodeValue, node.Name)
  			if err != nil {
  				log.Printf("Get ADFScore Failed, PodName:%s, NodeFrom:%s, NodeTo:%s.", pod.Name, neiNodeValue, node.Name)
  				ADFScore = 0
  			}
  			ADFScoreMap[node.Name] = ADFScore
  			ADFMin = math.Min(ADFMin, ADFScore)
  			ADFMax = math.Max(ADFMax, ADFScore)
  
  			// 获取未来网络延迟预测分数
  			FutureScore, err := neilats.FutureScore(neiNodeValue, node.Name, SLAConstraintValue)
  			if err != nil {
  				log.Printf("Get FutureScore Failed, PodName:%s, NodeFrom:%s, NodeTo:%s, SLAConstraint:%f.", pod.Name, neiNodeValue, node.Name, SLAConstraintValue)
  				FutureScore = 0
  			}
  			FutureScoreMap[node.Name] = FutureScore
  			FutureMin = math.Min(FutureMin, FutureScore)
  			FutureMax = math.Max(FutureMax, FutureScore)
  		}
  	}
  
  	// 将三种类型的分数分别归一化, 如果某一种分数的值都一样，统一设置为100
  	for _, node := range nodes {
  		LBScore := LBScoreMap[node.Name]
  		if LBMax == LBMin {
  			LBScoreMap[node.Name] = 100
  		} else {
  			LBScoreMap[node.Name] = ((LBScore-LBMin)/(LBMax-LBMin) + 1) * 100
  		}
  
  		if neilats.config.EnableSLA {
  			ADFScore := ADFScoreMap[node.Name]
  			if ADFMax == ADFMin {
  				ADFScoreMap[node.Name] = 100
  			} else {
  				ADFScoreMap[node.Name] = ((ADFScore-ADFMin)/(ADFMax-ADFMin) + 1) * 100
  			}
  
  			FutureScore := FutureScoreMap[node.Name]
  			if FutureMax == FutureMin {
  				FutureScoreMap[node.Name] = 100
  			} else {
  				FutureScoreMap[node.Name] = ((FutureScore-FutureMin)/(FutureMax-FutureMin) + 1) * 100
  			}
  		}
  	}
  	return framework.NewStatus(framework.Success, "LBScore、ADFScore、FutureScore Compute Complete.")
  }
  ~~~

* LBScore负载均衡度得分计算，未改变

* ADFScore网络稳定性得分计算

  ~~~go
  func (neilats *NeilatsRefactorScheduler) ADFScore(nodeFrom, nodeTo string) (float64, error) {
  	nodeFromUASMap := neilats.config.KubeNodeAddressAndSecret[nodeFrom]
  	nodeToUASMap := neilats.config.KubeNodeAddressAndSecret[nodeTo]
  	// 获取两个节点之间的最后30行延迟数据，依据此得到ADF分数
  	lastNLatency, err := getLastLinesLatencyFromSSH(nodeFrom, nodeFromUASMap.NodeAddress, nodeFromUASMap.NodeSecret, nodeToUASMap.NodeAddress, 30)
  	if err != nil {
  		log.Printf("ADF Get Last N Lines Latency Failed, From Node %s To %s:%v", nodeFrom, nodeTo, err)
  		return 0, err
  	}
  	// 定义获取ADF分数的结构体
  	type ADFRequest struct {
  		Latency []float64 `json:"latency"`
  	}
  
  	reqBody := ADFRequest{
  		Latency: lastNLatency,
  	}
  	jsonData, err := json.Marshal(reqBody)
  	if err != nil {
  		log.Printf("Error ADF Marshaling JSON: %v\n", err)
  		return 0, err
  	}
  	ADFScore, err := neilats.sendHTTPRequest(jsonData, "/adf_score")
  	if err != nil {
  		log.Printf("Error ADF Get ADFScore from RemoteServer: %v", err)
  		return 0, err
  	}
  	return ADFScore, nil
  }
  ~~~

* FutureScore未来链路得分计算

  ~~~go
  func (neilats *NeilatsRefactorScheduler) FutureScore(nodeFrom, nodeTo string, slaTime float64) (float64, error) {
  	nodeFromUASMap := neilats.config.KubeNodeAddressAndSecret[nodeFrom]
  	nodeToUASMap := neilats.config.KubeNodeAddressAndSecret[nodeTo]
  	lastNLatency, err := getLastLinesLatencyFromSSH(nodeFrom, nodeFromUASMap.NodeAddress, nodeFromUASMap.NodeSecret, nodeToUASMap.NodeAddress, 30)
  	if err != nil {
  		log.Printf("ADF Get Last N Lines Latency Failed, From Node %s To %s:%v", nodeFrom, nodeTo, err)
  		return 0, err
  	}
  	// 定义未来链路得分结构体
  	type FutureRequest struct {
  		Latency []float64 `json:"latency"`
  		SLATime float64   `json:"sla_time"`
  	}
  	// 创建请求体
  	reqBody := FutureRequest{
  		Latency: lastNLatency,
  		SLATime: slaTime,
  	}
  	// 将请求体JSON化
  	jsonData, err := json.Marshal(reqBody)
  	if err != nil {
  		log.Printf("Error FutureScore Marshaling JSON: %v\n", err)
  		return 0, err
  	}
  	// 利用LSTM延迟预测和未来链路得分计算服务得到FutureScore
  	FutureScore, err := neilats.sendHTTPRequest(jsonData, "/predict/"+nodeFrom+"2"+nodeTo)
  	if err != nil {
  		log.Printf("Error Future Get FutureScore from RemoteServer: %v", err)
  		return 0, err
  	}
  	return FutureScore, nil
  }
  ~~~

* 对Neilats_LSTM_ADF_Module模块进行HTTP访问

  ~~~go
  func (neilats *NeilatsRefactorScheduler) sendHTTPRequest(reqBodyData []byte, serverAddress string) (float64, error) {
  	url := neilats.config.LstmAdfModuleAddress + serverAddress
  
  	// 创建请求
  	request, err := http.NewRequest("POST", url, bytes.NewBuffer(reqBodyData))
  	if err != nil {
  		log.Printf("Error Creating requests: %v\n", err)
  		return 0, err
  	}
  
  	// 设置Header
  	request.Header.Set("Content-Type", "application/json")
  
  	// 发送请求
  	client := &http.Client{}
  	response, err := client.Do(request)
  	if err != nil {
  		log.Printf("Error Sending Request: %v\n", err)
  		return 0, err
  	}
  	defer response.Body.Close()
  
  	// 定义响应体结构体，用于接收数据
  	type ResponseType struct {
  		Score float64
  	}
  
  	// 读取响应体
  	body, err := io.ReadAll(response.Body)
  	if err != nil {
  		log.Printf("Error Read Response Body: %v\n", err)
  	}
  	// 解析响应体
  	var result ResponseType
  	err = json.Unmarshal(body, &result)
  	if err != nil {
  		log.Printf("Error Failed to pares JSON: %v\n", err)
  		log.Printf("Raw response %s\n", string(body))
  		return 0, err
  	}
  	return result.Score, nil
  }
  ~~~

* 变更之后的Score扩展点

  ~~~go
  func (neilats *NeilatsRefactorScheduler) Score(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) (int64, *framework.Status) {
  	if neilats.config.EnableSLA {
  		// 归一化已经完成， 只需要求该节点各个分数的均方值
  		lbscore := LBScoreMap[nodeName]
  		adfscore := ADFScoreMap[nodeName]
  		futurescore := FutureScoreMap[nodeName]
  		return int64(math.Cbrt(lbscore * adfscore * futurescore)), nil
  	} else {
  		return int64(LBScoreMap[nodeName]), nil
  	}
  }
  ~~~

### 3.工具函数getLastLinesLatencyFromSSH

原本的getMeanOfLatencyFromSSH工具函数被拆为两个工具函数

1. getMeanOfLatencyFromSSH

   ~~~go
   func getMeanOfLatencyFromSSH(nodeFromName, nodeFromAddress, nodeFromSecret, nodeToAddress string) (float64, error) {
   	lastNLatency, err := getLastLinesLatencyFromSSH(nodeFromName, nodeFromAddress, nodeFromSecret, nodeToAddress, 30)
   	if err != nil {
   		log.Printf("Get last N Lines Latency Data Failed:%v\n", err)
   	}
   
   	var latencySum float64 = 0
   	for _, line := range lastNLatency {
   		latencySum += line
   	}
   
   	// 返回延迟平均值
   	return latencySum / float64(len(lastNLatency)), nil
   }
   ~~~

2. getLastLinesLatencyFromSSH

   ~~~go
   func getLastLinesLatencyFromSSH(nodeFromName, nodeFromAddress, nodeFromSecret, nodeToAddress string, lastNLines int) ([]float64, error) {
   	resultFilePath := fmt.Sprintf("/root/ws/network-latency-test/latency_results_%s.txt", nodeToAddress)
   
   	// 连接到远程服务器
   	client, err := getSshClient(nodeFromName, nodeFromAddress, nodeFromSecret)
   	if err != nil {
   		log.Printf("failed to login Node %q: %v\n", nodeFromName, err)
   		return nil, err
   	}
   	defer client.Close()
   
   	// 创建SFTP客户端
   	sftpClient, err := sftp.NewClient(client)
   	if err != nil {
   		log.Printf("after Login Node %q, Failed Create Sftp Client: %v\n", nodeFromName, err)
   		return nil, err
   	}
   	defer sftpClient.Close()
   
   	// 打开远程主机上的文件
   	resultFile, err := sftpClient.Open(resultFilePath)
   	if err != nil {
   		log.Printf("after Create Sftp Client on Node %q, Failed Open remoteFile:%v\n", nodeFromName, err)
   		return nil, err
   	}
   	defer resultFile.Close()
   
   	// 保存每行数据字符串
   	var lines []string
   	scanner := bufio.NewScanner(resultFile)
   	for scanner.Scan() {
   		lines = append(lines, scanner.Text())
   		if len(lines) > lastNLines {
   			// 仅保留最后lastN行数据，代码含义为切片从下标为1的元素开始到最后一个元素作为一个新切片
   			lines = lines[1:]
   		}
   	}
   	if err := scanner.Err(); err != nil {
   		log.Printf("get Last N Lines in ResultFile Failed:%v\n", err)
   		return nil, err
   	}
   
   	// 保存最后N行的延迟数据值
   	var lastNLatency []float64
   
   	for _, line := range lines {
   		splitedLine := strings.Split(line, " ")
   		dataStr := splitedLine[len(splitedLine)-2]
   		dataStr, _ = strings.CutPrefix(dataStr, "(")
   		latencyData, err := strconv.ParseFloat(dataStr, 64)
   		if err != nil {
   			log.Printf("convert Data String to Float Failed:%v\n", err)
   			return nil, err
   		}
   		lastNLatency = append(lastNLatency, latencyData)
   	}
   	// 返回最后N行的延迟值
   	return lastNLatency, nil
   }
   ~~~






## 测试与修改

## 25.12.8

### 修改1：NeiNode==CurrentNode问题、Pod标签无法识别问题、日志信息增加

> 1. 当Pod的`nei_node`、`sla`标签写错位置，例如写在Deployment下时，又同时开启SLA了，就会出现
>
>    ~~~
>    I1208 03:32:57.108501       1 log.go:245] failed to login Node "": dial tcp :22: connect: connection refused
>    I1208 03:32:57.108509       1 log.go:245] failed to login Node "": dial tcp :22: connect: connection refused
>    I1208 03:32:57.108514       1 log.go:245] ADF Get Last N Lines Latency Failed, From Node  To node1:dial tcp :22: connect: connection refused
>    I1208 03:32:57.108519       1 log.go:245] Get ADFScore Failed, PodName:nginx-deployment-test-neilats-f7b7fdcfb-znvqk, NodeFrom:, NodeTo:node1.
>    I1208 03:32:57.108565       1 log.go:245] failed to login Node "": dial tcp :22: connect: connection refused
>    I1208 03:32:57.108573       1 log.go:245] failed to login Node "": dial tcp :22: connect: connection refused
>    I1208 03:32:57.108581       1 log.go:245] ADF Get Last N Lines Latency Failed, From Node  To node1:dial tcp :22: connect: connection refused
>    I1208 03:32:57.108586       1 log.go:245] Get FutureScore Failed, PodName:nginx-deployment-test-neilats-f7b7fdcfb-znvqk, NodeFrom:, NodeTo:node1, SLAConstraint:0.000000.
>    I1208 03:32:57.108687       1 log.go:245] Node %!s(int=0) Normalized Score is 0
>    I1208 03:32:57.108695       1 log.go:245] Node %!s(int=1) Normalized Score is 100
>    I1208 03:32:57.108698       1 log.go:245] Node %!s(int=2) Normalized Score is 0
>    I1208 03:32:57.108817       1 log.go:245] Pod nginx-deployment-test-neilats-f7b7fdcfb-wzq75 haven't NeiNodeLable And SLAConstraintLable, pass Prefilter
>    ~~~
>
> 2. 当Pod的邻近节点是当前的测试节点，即FromNode和ToNode相同时，处理出错，会找不到延迟文件
>
> 3. ADF、FutureScore两个得分建议增加相关日志信息；对最终多个节点在NormalizationScore日志进行优化

* 针对问题1，做出如下代码修改

  在`PreScore`扩展点中，关于计算ADF、FutureScore的地方，增加if语句条件

  ~~~go
  		// 虽然还是检查过了，但是还是防止开启SLA的情况下，普通Pod触发ADF、FutureScore的计算
  		if neilats.config.EnableSLA && neiNodeExist && slaExist {
  			// 安全检查已经在PreFilter阶段通过了，不用检查了
  			SLAConstraintValue, _ := strconv.ParseFloat(slaConstraintValue, 64)
  			// 获取ADF分数
  			ADFScore, err := neilats.ADFScore(neiNodeValue, node.Name)
              .......
              
          	// 将三种类型的分数分别归一化, 如果某一种分数的值都一样，统一设置为100
  	for _, node := range nodes {
  		LBScore := LBScoreMap[node.Name]
  		if LBMax == LBMin {
  			LBScoreMap[node.Name] = 100
  		} else {
  			LBScoreMap[node.Name] = ((LBScore-LBMin)/(LBMax-LBMin) + 1) * 100
  		}
  
  		if neilats.config.EnableSLA && neiNodeExist && slaExist {
  			ADFScore := ADFScoreMap[node.Name]
  			if ADFMax == ADFMin {
  				ADFScoreMap[node.Name] = 100
  			} else {
  ~~~

  在Score扩展点中，对开启SLA的情况增加if条件

  ~~~go
  	// 检查Pod是否存在邻近节点标签
  	_, neiNodeExist := p.GetLabels()["nei_node"]
  	// 检查Pod是否存在SLA约束要求标签
  	_, slaExist := p.GetLabels()["sla"]
  
  	var finalScore int64
  	if neilats.config.EnableSLA && neiNodeExist && slaExist {
  		// 归一化已经完成， 只需要求该节点各个分数的均方值
  		lbscore := LBScoreMap[nodeName]
  		adfscore := ADFScoreMap[nodeName]
  		futurescore := FutureScoreMap[nodeName]
  
  		finalScore = int64(math.Cbrt(lbscore * adfscore * futurescore))
  ~~~

* 针对问题2，修改`Filter`扩展点，设计将邻近节点作为不可调度节点

  ~~~go
  		// 如果邻近节点与当前节点一致，不予通过，即Pod不可调度到邻近节点上
  		if neiNodeValue == nodeInfo.Node().Name {
  			log.Printf("Pod Nei_Node %s is Same As Current Node!", neiNodeValue)
  			return framework.NewStatus(framework.UnschedulableAndUnresolvable, fmt.Sprintf("Failed Build RttMatrix:%v", err))
  		}
  ~~~

* 针对问题3，修改ADFScore、FutureScore，增加日志输出信息语句

  ~~~go
  	log.Printf("ADF Get Last N Lines Latency Success, From Node %s To %s\n", nodeFrom, nodeTo)
  	log.Printf("ADF Get ADFScore Success from RemoteServer, from Node %s To Node %s: %f", nodeFrom, nodeTo, ADFScore)
  
  	log.Printf("FutureScore Get Last N Lines Latency Success, From Node %s To %s\n", nodeFrom, nodeTo)
  	log.Printf("FutureScore Get Success from RemoteServer, from Node %s To Node %s: %f", nodeFrom, nodeTo, FutureScore)
  ~~~

### 修改2：ADF和FutureScore得分解析问题、FutureScore访问端口错乱问题

> 1. 当服务访问正常后，LSTM_ADF_Module返回的信息解析出现得分为0的情况，即ADF、FutureScore解析错误
>
>    ~~~sh
>    I1208 08:42:39.882356       1 log.go:245] ADF Get Last N Lines Latency Success, From Node node1 To node2
>    I1208 08:42:39.920462       1 log.go:245] ADF Get ADFScore Success from RemoteServer, from Node node1 To Node node2: 0.000000
>    I1208 08:42:40.057018       1 log.go:245] FutureScore Get Last N Lines Latency Success, From Node node1 To node2
>    I1208 08:42:40.112988       1 log.go:245] FutureScore Get Success from RemoteServer, from Node node1 To Node node2: 0.00000
>    ~~~
>
> 2. 有时候访问FutureScore端口会错误，因为从master到node1，不止有master2node1，还会使node12master
>
>    ~~~shell
>    I1208 08:42:39.728834       1 log.go:245] Get FutureScore Failed, PodName:nginx-deployment-test-neilats-586b9b9ccf-rwt2r, NodeFrom:node1, NodeTo:master, SLAConstraint:0.300000.
>    ~~~

* 针对问题1，首先将调度插件中，对返回体的定义增加json解析字段；其次，将LSTM_ADF_Module的返回字段统一设置为`score`

  解析错误是因为，调度插件中的返回体定义没增加json解析字段，并且返回体成员变量名与服务返回体中的字段名不一致，导致解析错误

  为了让同一类型的返回体能同时接收adf_score和future_score，因此将LSTM_ADF_Module返回体字段统一设置为`score`

  1. 调度插件`sendHTTPRequest`修改

     ~~~go
     	// 定义响应体结构体，用于接收数据
     	type ResponseType struct {
     		Score float64 `json:"score"`
     	}
     ~~~

  2. `LSTM_ADF_Module:predict_flask_app.py`修改

     ~~~python
             # 计算未来通信链路得分
             print(model_scaler_name + " predict latency:")
             print(pred_inv.tolist())
             future_score = def_future_score(pred_inv.tolist(), sla_time)
             print(model_scaler_name + " future_score:")
             print(future_score)
             return jsonify({"score": future_score})
         
     	# 计算稳定性得分
         print(data["latency"])
         adf_score = adf(data["latency"])
         print("adf_score:")
         print(adf_score)
         return jsonify({"score": adf_score})
     ~~~

     具体见：

* 针对问题2，将LSTM_ADF_Module访问端口统一为一个模式

  修改`LSTM_ADF_Module:predict_flask_app.py`

  ~~~python
  @app.route("/predict/<path:pattern>", methods=["POST"])
  def predict_master2node1(pattern):
      model_scaler_name = ""
      if pattern in ["master2node1", "node12master"]:
          model_scaler_name = "master2node1"
      elif pattern in ["master2node2", "node22master"]:
          model_scaler_name = "master2node2"
      elif pattern in ["master2node3", "node32master"]:
          model_scaler_name = "master2node3"
      elif pattern in ["node12node2", "node22node1"]:
          model_scaler_name = "node12node2"
      elif pattern in ["node12node3", "node32node1"]:
          model_scaler_name = "node12node3"
      elif pattern in ["node22node3", "node32node2"]:
          model_scaler_name = "node22node3"
      try:
          # 获取数据
          input_tensor, sla_time = get_input_tensor(model_scaler_name)
          # 推理
          with torch.no_grad():
              pred_scaled = MODELS[model_scaler_name](input_tensor)
              pred_scaled = pred_scaled.cpu().numpy()
          # 反归一化
          pred_inv = SCALERS[model_scaler_name].inverse_transform(pred_scaled.reshape(-1, 1))
  
          # 计算未来通信链路得分
          print(model_scaler_name + " predict latency:")
          print(pred_inv.tolist())
          future_score = def_future_score(pred_inv.tolist(), sla_time)
          print(model_scaler_name + " future_score:")
          print(future_score)
          return jsonify({"score": future_score})
  
          # return jsonify({"prediction": pred_inv.tolist()})
      except Exception as e:
          return jsonify({"error":str(e)}), 500
  ~~~

  具体见：

### Version2调度成功

测试yaml

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-test-neilats
  namespace: k8s-learn
  labels:
    app: nginx
    nei_node: "node1"
    sla: "0.3"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        nei_node: "node1"
        sla: "0.3"
    spec:
      schedulerName: neilats-refactor-scheduler
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
~~~




~~~sh
(base) root@master:~/work/SchedulingFramework# kubectl logs neilats-refactor-scheduler-7b4f7bc5bb-cq7t7 -n k8s-learn
I1209 03:31:34.223607       1 serving.go:348] Generated self-signed cert in-memory
W1209 03:31:34.224313       1 client_config.go:618] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I1209 03:31:34.447581       1 log.go:245] Custom Neilats Args Detected!
I1209 03:31:34.447596       1 log.go:245] Config EnableSLA is True!
I1209 03:31:34.447605       1 log.go:245] Init Node "master" Latency Test Tool
I1209 03:31:35.053211       1 log.go:245] Command executed successfully
I1209 03:31:35.053321       1 log.go:245] Install Latency Test Tool Success
I1209 03:31:35.175329       1 log.go:245] Command executed successfully
I1209 03:31:35.175381       1 log.go:245] Node "master" Create Latency Test Directory Success
I1209 03:31:35.304325       1 log.go:245] Init Node "node1" Latency Test Tool
I1209 03:31:35.950964       1 log.go:245] Command executed successfully
I1209 03:31:35.951015       1 log.go:245] Install Latency Test Tool Success
I1209 03:31:36.075532       1 log.go:245] Command executed successfully
I1209 03:31:36.075584       1 log.go:245] Node "node1" Create Latency Test Directory Success
I1209 03:31:36.202473       1 log.go:245] Init Node "node2" Latency Test Tool
I1209 03:31:36.856950       1 log.go:245] Command executed successfully
I1209 03:31:36.857002       1 log.go:245] Install Latency Test Tool Success
I1209 03:31:36.985870       1 log.go:245] Command executed successfully
I1209 03:31:36.985918       1 log.go:245] Node "node2" Create Latency Test Directory Success
I1209 03:31:37.108609       1 log.go:245] Init Node "node3" Latency Test Tool
I1209 03:31:37.755220       1 log.go:245] Command executed successfully
I1209 03:31:37.755260       1 log.go:245] Install Latency Test Tool Success
I1209 03:31:37.883433       1 log.go:245] Command executed successfully
I1209 03:31:37.883494       1 log.go:245] Node "node3" Create Latency Test Directory Success
I1209 03:31:38.022859       1 log.go:245] Init Node Latency Test Tool Success!
I1209 03:31:38.156646       1 log.go:245] Command executed successfully
I1209 03:31:38.156692       1 log.go:245] Success Start Node "master" Latency Test!
I1209 03:31:38.278145       1 log.go:245] Command executed successfully
I1209 03:31:38.278193       1 log.go:245] Success Start Node "node1" Latency Test!
I1209 03:31:38.401019       1 log.go:245] Command executed successfully
I1209 03:31:38.401072       1 log.go:245] Success Start Node "node2" Latency Test!
I1209 03:31:38.530669       1 log.go:245] Command executed successfully
I1209 03:31:38.530717       1 log.go:245] Success Start Node "node3" Latency Test!
I1209 03:31:38.530728       1 log.go:245] Start Latency Test Success!
I1209 03:31:38.530733       1 log.go:245] Final Neilats Config is:
I1209 03:31:38.530773       1 log.go:245] {{ } http://192.168.3.226:31739 map[master:ens18 node1:ens18 node2:ens18 node3:ens18] map[master:/dev/mapper/ubuntu--vg-ubuntu--lv node1:/dev/mapper/ubuntu--vg-ubuntu--lv node2:/dev/mapper/ubuntu--vg-ubuntu--lv node3:/dev/mapper/ubuntu--vg-ubuntu--lv] true map[master:{192.168.3.226 mobisys912} node1:{192.168.3.229 mobisys912} node2:{192.168.3.224 mobisys912} node3:{192.168.3.228 mobisys912}] http://192.168.3.212:30080}
I1209 03:31:38.536033       1 capacity_scheduling.go:190] "CapacityScheduling start"
I1209 03:31:38.536530       1 server.go:154] "Starting Kubernetes Scheduler" version="v0.0.20251208"
I1209 03:31:38.536544       1 server.go:156] "Golang settings" GOGC="" GOMAXPROCS="" GOTRACEBACK=""
I1209 03:31:38.539693       1 configmap_cafile_content.go:202] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
I1209 03:31:38.539684       1 configmap_cafile_content.go:202] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::client-ca-file"
I1209 03:31:38.539749       1 shared_informer.go:311] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I1209 03:31:38.539761       1 requestheader_controller.go:169] Starting RequestHeaderAuthRequestController
I1209 03:31:38.539775       1 shared_informer.go:311] Waiting for caches to sync for RequestHeaderAuthRequestController
I1209 03:31:38.539750       1 shared_informer.go:311] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
I1209 03:31:38.539876       1 secure_serving.go:213] Serving securely on [::]:10259
I1209 03:31:38.539923       1 tlsconfig.go:240] "Starting DynamicServingCertificateController"
I1209 03:31:38.640604       1 shared_informer.go:318] Caches are synced for RequestHeaderAuthRequestController
I1209 03:31:38.640665       1 shared_informer.go:318] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I1209 03:31:38.640678       1 shared_informer.go:318] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
I1209 03:34:13.209378       1 log.go:245] Pod nginx-deployment-test-neilats-586b9b9ccf-q2zvq NeiNodeLable node1 And SLAConstraintLabel "0.3", pass Prefilter
I1209 03:34:13.209471       1 log.go:245] Pod Nei_Node node1 is Same As Current Node!
I1209 03:34:15.656366       1 log.go:245] Node node2 rtt Satisfy Pod nginx-deployment-test-neilats-586b9b9ccf-q2zvq SLA Constraint
I1209 03:34:15.680792       1 log.go:245] Node node3 rtt Satisfy Pod nginx-deployment-test-neilats-586b9b9ccf-q2zvq SLA Constraint
I1209 03:34:15.846183       1 log.go:245] Node master rtt Satisfy Pod nginx-deployment-test-neilats-586b9b9ccf-q2zvq SLA Constraint
I1209 03:34:15.848119       1 log.go:245] Total CPU of node: node2 is 8000.000000
I1209 03:34:15.849331       1 log.go:245] CPU idle rate of node: node2 is 0.988549
I1209 03:34:15.850032       1 log.go:245] Total memory of node: node2 is 15991.878906 MB
I1209 03:34:15.850598       1 log.go:245] Memory available of node: node2 is 0.851887
I1209 03:34:15.851699       1 log.go:245] Network available of node: node2 is 3.842759
I1209 03:34:15.852249       1 log.go:245] Total disk of node: node2 is 78306.820312 MB
I1209 03:34:15.852962       1 log.go:245] Disk available of node: node2 is 54.236790
I1209 03:34:15.852970       1 log.go:245] Node Name: node2 LB Score: -205949.796068
I1209 03:34:16.029373       1 log.go:245] ADF Get Last N Lines Latency Success, From Node node1 To node2
I1209 03:34:16.252353       1 log.go:245] ADF Get ADFScore Success from RemoteServer, from Node node1 To Node node2: 0.000000
I1209 03:34:16.451452       1 log.go:245] FutureScore Get Last N Lines Latency Success, From Node node1 To node2
I1209 03:34:16.661850       1 log.go:245] FutureScore Get Success from RemoteServer, from Node node1 To Node node2: -4.913113
I1209 03:34:16.662937       1 log.go:245] Total CPU of node: node3 is 8000.000000
I1209 03:34:16.663989       1 log.go:245] CPU idle rate of node: node3 is 0.994408
I1209 03:34:16.664672       1 log.go:245] Total memory of node: node3 is 15991.820312 MB
I1209 03:34:16.665335       1 log.go:245] Memory available of node: node3 is 0.939870
I1209 03:34:16.666431       1 log.go:245] Network available of node: node3 is 3.859632
I1209 03:34:16.666926       1 log.go:245] Total disk of node: node3 is 78306.820312 MB
I1209 03:34:16.667503       1 log.go:245] Disk available of node: node3 is 52.942462
I1209 03:34:16.667511       1 log.go:245] Node Name: node3 LB Score: -195618.290413
I1209 03:34:16.843504       1 log.go:245] ADF Get Last N Lines Latency Success, From Node node1 To node3
I1209 03:34:16.862384       1 log.go:245] ADF Get ADFScore Success from RemoteServer, from Node node1 To Node node3: 9.533285
I1209 03:34:17.048354       1 log.go:245] FutureScore Get Last N Lines Latency Success, From Node node1 To node3
I1209 03:34:17.101764       1 log.go:245] FutureScore Get Success from RemoteServer, from Node node1 To Node node3: -4.923121
I1209 03:34:17.102699       1 log.go:245] Total CPU of node: master is 8000.000000
I1209 03:34:17.103638       1 log.go:245] CPU idle rate of node: master is 0.965129
I1209 03:34:17.104428       1 log.go:245] Total memory of node: master is 15991.886719 MB
I1209 03:34:17.105074       1 log.go:245] Memory available of node: master is 0.698509
I1209 03:34:17.106392       1 log.go:245] Network available of node: master is 3.842724
I1209 03:34:17.107088       1 log.go:245] Total disk of node: master is 78306.820312 MB
I1209 03:34:17.107855       1 log.go:245] Disk available of node: master is 20.877731
I1209 03:34:17.107864       1 log.go:245] Node Name: master LB Score: -27703.548345
I1209 03:34:17.292575       1 log.go:245] ADF Get Last N Lines Latency Success, From Node node1 To master
I1209 03:34:17.306574       1 log.go:245] ADF Get ADFScore Success from RemoteServer, from Node node1 To Node master: 0.000000
I1209 03:34:17.492224       1 log.go:245] FutureScore Get Last N Lines Latency Success, From Node node1 To master
I1209 03:34:17.548984       1 log.go:245] FutureScore Get Success from RemoteServer, from Node node1 To Node master: -4.295151
I1209 03:34:17.549089       1 log.go:245] Node node2 Final Score before Normalized 100
I1209 03:34:17.549100       1 log.go:245] Node master Final Score before Normalized 158
I1209 03:34:17.549120       1 log.go:245] Node node3 Final Score before Normalized 128
I1209 03:34:17.549161       1 log.go:245] Node 1 Normalized Score is 0
I1209 03:34:17.549168       1 log.go:245] Node 2 Normalized Score is 0
I1209 03:34:17.549172       1 log.go:245] Node 3 Normalized Score is 100
I1209 03:34:17.549382       1 trace.go:236] Trace[550718497]: "Scheduling" namespace:k8s-learn,name:nginx-deployment-test-neilats-586b9b9ccf-q2zvq (09-Dec-2025 03:34:13.209) (total time: 4340ms):
Trace[550718497]: ---"Computing predicates done" 2636ms (03:34:15.846)
Trace[550718497]: ---"Prioritizing done" 1703ms (03:34:17.549)
Trace[550718497]: [4.340047949s] [4.340047949s] END
I1209 03:34:17.549586       1 log.go:245] Pod nginx-deployment-test-neilats-586b9b9ccf-gfcpl NeiNodeLable node1 And SLAConstraintLabel "0.3", pass Prefilter
I1209 03:34:17.549973       1 log.go:245] Pod Nei_Node node1 is Same As Current Node!
I1209 03:34:19.832455       1 log.go:245] Node node2 rtt Satisfy Pod nginx-deployment-test-neilats-586b9b9ccf-gfcpl SLA Constraint
I1209 03:34:19.833681       1 log.go:245] Node master rtt Satisfy Pod nginx-deployment-test-neilats-586b9b9ccf-gfcpl SLA Constraint
I1209 03:34:19.833713       1 log.go:245] Node node3 rtt Satisfy Pod nginx-deployment-test-neilats-586b9b9ccf-gfcpl SLA Constraint
I1209 03:34:19.835071       1 log.go:245] Total CPU of node: node2 is 8000.000000
I1209 03:34:19.836099       1 log.go:245] CPU idle rate of node: node2 is 0.988549
I1209 03:34:19.836846       1 log.go:245] Total memory of node: node2 is 15991.878906 MB
I1209 03:34:19.837619       1 log.go:245] Memory available of node: node2 is 0.851887
I1209 03:34:19.839092       1 log.go:245] Network available of node: node2 is 3.842759
I1209 03:34:19.839546       1 log.go:245] Total disk of node: node2 is 78306.820312 MB
I1209 03:34:19.840349       1 log.go:245] Disk available of node: node2 is 54.236790
I1209 03:34:19.840356       1 log.go:245] Node Name: node2 LB Score: -205949.796068
I1209 03:34:20.022470       1 log.go:245] ADF Get Last N Lines Latency Success, From Node node1 To node2
I1209 03:34:20.043228       1 log.go:245] ADF Get ADFScore Success from RemoteServer, from Node node1 To Node node2: 3.221916
I1209 03:34:20.228560       1 log.go:245] FutureScore Get Last N Lines Latency Success, From Node node1 To node2
I1209 03:34:20.298119       1 log.go:245] FutureScore Get Success from RemoteServer, from Node node1 To Node node2: -4.914557
I1209 03:34:20.299151       1 log.go:245] Total CPU of node: master is 8000.000000
I1209 03:34:20.300129       1 log.go:245] CPU idle rate of node: master is 0.957878
I1209 03:34:20.300869       1 log.go:245] Total memory of node: master is 15991.886719 MB
I1209 03:34:20.301715       1 log.go:245] Memory available of node: master is 0.693416
I1209 03:34:20.303093       1 log.go:245] Network available of node: master is 2.780608
I1209 03:34:20.303696       1 log.go:245] Total disk of node: master is 78306.820312 MB
I1209 03:34:20.304331       1 log.go:245] Disk available of node: master is 20.877547
I1209 03:34:20.304340       1 log.go:245] Node Name: master LB Score: -28386.004252
I1209 03:34:20.475914       1 log.go:245] ADF Get Last N Lines Latency Success, From Node node1 To master
I1209 03:34:20.508610       1 log.go:245] ADF Get ADFScore Success from RemoteServer, from Node node1 To Node master: 12.578812
I1209 03:34:20.691741       1 log.go:245] FutureScore Get Last N Lines Latency Success, From Node node1 To master
I1209 03:34:20.750974       1 log.go:245] FutureScore Get Success from RemoteServer, from Node node1 To Node master: -4.187064
I1209 03:34:20.752174       1 log.go:245] Total CPU of node: node3 is 8000.000000
I1209 03:34:20.753353       1 log.go:245] CPU idle rate of node: node3 is 0.994408
I1209 03:34:20.754262       1 log.go:245] Total memory of node: node3 is 15991.820312 MB
I1209 03:34:20.755143       1 log.go:245] Memory available of node: node3 is 0.939870
I1209 03:34:20.756697       1 log.go:245] Network available of node: node3 is 3.859632
I1209 03:34:20.757507       1 log.go:245] Total disk of node: node3 is 78306.820312 MB
I1209 03:34:20.758394       1 log.go:245] Disk available of node: node3 is 52.942462
I1209 03:34:20.758407       1 log.go:245] Node Name: node3 LB Score: -195618.290413
I1209 03:34:20.934777       1 log.go:245] ADF Get Last N Lines Latency Success, From Node node1 To node3
I1209 03:34:20.960013       1 log.go:245] ADF Get ADFScore Success from RemoteServer, from Node node1 To Node node3: 10.683871
I1209 03:34:21.137137       1 log.go:245] FutureScore Get Last N Lines Latency Success, From Node node1 To node3
I1209 03:34:21.206170       1 log.go:245] FutureScore Get Success from RemoteServer, from Node node1 To Node node3: -4.912061
I1209 03:34:21.206279       1 log.go:245] Node node2 Final Score before Normalized 100
I1209 03:34:21.206301       1 log.go:245] Node master Final Score before Normalized 200
I1209 03:34:21.206333       1 log.go:245] Node node3 Final Score before Normalized 124
I1209 03:34:21.206376       1 log.go:245] Node 1 Normalized Score is 0
I1209 03:34:21.206381       1 log.go:245] Node 2 Normalized Score is 100
I1209 03:34:21.206384       1 log.go:245] Node 3 Normalized Score is 0
I1209 03:34:21.206433       1 trace.go:236] Trace[1019969039]: "Scheduling" namespace:k8s-learn,name:nginx-deployment-test-neilats-586b9b9ccf-gfcpl (09-Dec-2025 03:34:17.549) (total time: 3656ms):
Trace[1019969039]: ---"Computing predicates done" 2284ms (03:34:19.833)
Trace[1019969039]: ---"Prioritizing done" 1372ms (03:34:21.206)
Trace[1019969039]: [3.65687836s] [3.65687836s] END

~~~

