# 字节 AI 云原生一面

## 一、Linux：CPU 利用率不高，但 Load 很高

### 1. CPU 利用率和 Load 分别统计什么？

#### CPU 利用率

CPU 利用率反映 CPU 时间花在了什么地方，常见指标包括：

- `us`：用户态程序消耗的 CPU
- `sy`：内核态消耗的 CPU
- `wa`：等待 I/O 的时间
- `st`：虚拟机被宿主机抢占的时间
- `id`：空闲时间

通常可以粗略理解为：

```text
CPU 利用率 ≈ 100% - idle%
```

但不同工具对 `iowait` 的统计方式可能不同。CPU 等待磁盘时，`top` 可能显示较高的 `wa`，但实际用于计算的 CPU 并不高。

#### Load Average

Load 统计的是：

1. 正在运行或等待运行的任务，通常是 `R` 状态；
2. 处于不可中断睡眠的任务，通常是 `D` 状态。

因此 Load 不等于 CPU 利用率。大量进程等待磁盘、NFS 或其他 I/O 时，虽然没有消耗很多 CPU，但处于 `D` 状态，仍然会增加 Load，从而出现 CPU 利用率低、Load 很高的现象。

Load 还要结合 CPU 核数判断。例如 8 核机器 Load 为 4 通常不算高，而 Load 为 20 则说明排队比较严重。

### 2. 如何定位？

先看总体状态：

```bash
uptime
top
htop
```

重点关注 Load、`us`、`sy`、`wa`、`st`，以及是否存在大量 `D` 状态进程。

查看进程状态：

```bash
ps -eo pid,stat,comm,wchan:32 | awk '$2 ~ /D/'
```

查看 CPU、I/O 和上下文切换：

```bash
vmstat 1
iostat -xz 1
pidstat -d 1
```

如果发现大量 `D` 状态，则继续检查磁盘、网络文件系统和内核日志：

```bash
dmesg
iostat -xz 1
sar -d 1
```

必要时可以使用 `strace -p <pid>` 查看进程卡在哪个系统调用。

### 面试总结

Load 高但 CPU 利用率低，通常说明进程不是在消耗 CPU，而是在等待某种不可中断资源，最常见的是磁盘或网络 I/O。定位时先通过 `top`、`vmstat` 判断是 I/O Wait、进程 `D` 状态还是 CPU steal，再用 `iostat`、`pidstat`、`dmesg`、`strace` 定位具体原因。

---

## 二、Kubernetes：request、limit 和 QoS

### 1. request 的作用

`request` 主要用于调度：

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

调度器会认为该 Pod 至少需要 0.5 个 CPU 和 512 MiB 内存，只有节点可分配资源满足 request 时，Pod 才可能被调度到该节点。CPU request 还会影响 CPU 竞争时的相对权重。

### 2. limit 的作用

`limit` 是容器可以使用的资源上限：

```yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
```

| 资源 | 超过 limit 的表现 |
|---|---|
| CPU | 通常被 CFS throttling 限流，进程变慢但不一定退出 |
| Memory | 可能触发 OOM，容器状态通常显示 `OOMKilled` |

### 3. CFS throttling 是什么？为什么会变慢？

CFS 是 Linux 的 Completely Fair Scheduler。Kubernetes 通常通过 Linux cgroup 的 CPU quota 实现 CPU limit，例如：

```text
cpu.cfs_quota_us
cpu.cfs_period_us
```

如果容器在一个调度周期内提前用完 CPU 配额，剩余时间就不能继续运行，只能等待下一个周期。这段被强制暂停的过程就是 CFS throttling。

因此 CPU 超过 limit 时，通常不会像内存超限那样直接被杀掉，而是被限制运行，表现为请求延迟升高、吞吐下降和程序变慢。可以关注：

```text
container_cpu_cfs_throttled_seconds_total
```

### 4. Pod 没吃满 request 时，其他 Pod 能否使用这部分资源？

可以。request 不是物理 CPU 独占，也不是固定划给 Pod 的 CPU 时间。它主要用于：

- 调度时的资源计算；
- CPU 竞争时的调度权重；
- 内存压力下的驱逐判断。

例如某 Pod 的 CPU request 是 2，但实际只使用 0.5，那么节点上的其他 Pod 可以使用空闲 CPU。

当该 Pod 后续需要更多 CPU 时，Linux 调度器会重新分配 CPU 时间，其他 Pod 可能获得更少的时间片，但不会被 Kubernetes 驱逐或杀掉。因此 request 提供的是资源竞争时的权重保证，而不是物理 CPU 独占。

### 5. QoS 的作用以及是否可以计算

QoS 不是一个连续的资源分数，而是根据 Pod 中所有容器的 request 和 limit 通过规则计算出的分类标签。

#### Guaranteed

每个容器的 CPU 和内存都设置了 request 和 limit，并且二者相等：

```yaml
requests:
  cpu: "1"
  memory: "1Gi"
limits:
  cpu: "1"
  memory: "1Gi"
```

#### BestEffort

所有容器都没有设置 request 和 limit。

#### Burstable

不满足前两种情况的其他 Pod，通常属于 Burstable。

QoS 主要影响：

- 节点内存压力下的驱逐优先级；
- OOM 场景下的优先级；
- Kubernetes 对 Pod 进行资源隔离和管理的策略。

一般可以粗略理解为：

```text
BestEffort 更容易被驱逐
Burstable 居中
Guaranteed 相对更稳定
```

但 Burstable Pod 如果实际使用内存远超 request，也可能比其他 Pod 更早被驱逐。

### 面试总结

request 用于调度，并影响资源竞争时的权重；limit 是容器可以使用的资源上限。CPU 超过 limit 通常被 CFS 限流，内存超过 limit 通常触发 OOMKilled。request 没吃满时，其他 Pod 可以使用节点上的空闲资源。QoS 是根据 request 和 limit 计算出的分类，主要影响资源压力下的驱逐和稳定性，而不是一个具体的资源数量。

---

## 三、Go：goroutine 无限增殖

### 1. 如何定位？

先观察 goroutine 数量：

```go
runtime.NumGoroutine()
```

可以通过 Prometheus 持续监控该指标。如果数量持续增长且不回落，说明可能存在 goroutine 泄漏或无限创建。

开启 pprof：

```go
import _ "net/http/pprof"

go func() {
    http.ListenAndServe(":6060", nil)
}()
```

查看 goroutine 堆栈：

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

重点观察大量 goroutine 是否卡在同一个位置，例如：

- channel 发送或接收；
- `mutex.Lock()`；
- `WaitGroup.Wait()`；
- 网络读写；
- 数据库操作；
- 某个业务循环。

还可以结合 `go tool trace` 分析调度，以及使用 `go test -race` 排查数据竞争。需要注意，Race Detector 不一定能直接发现 goroutine 泄漏。

### 2. 常见原因

- channel 没有发送者或接收者，导致 goroutine 永久阻塞；
- channel 没有关闭，消费者一直等待；
- `context` 没有正确取消；
- 每个请求都启动 goroutine，但没有并发限制；
- 锁竞争导致 goroutine 长期等待；
- HTTP、TCP 或数据库操作没有超时；
- `Ticker`、Timer 或重试任务没有停止；
- WaitGroup 使用错误，导致任务无法结束。

### 3. “不退出”和“无限增殖”的区别

如果只有一个 goroutine 因 channel 阻塞，它只是泄漏一个 goroutine，不会自动无限增长。

无限增长通常还需要存在持续创建 goroutine 的路径，例如：

```go
for {
    go func() {
        ch <- value
    }()
}
```

此时每轮循环都创建一个新的 goroutine，而已有 goroutine 又因 channel 阻塞无法退出，于是数量持续增长。

Go 没有类似操作系统父子进程的“每个父 goroutine 最多创建多少子 goroutine”的默认限制。最终限制通常来自内存、栈空间、调度开销、文件描述符、网络连接或容器 memory limit。

因此需要通过 worker pool、信号量、context cancel、最大并发数、超时和重试上限主动控制生命周期。

### 面试总结

先监控 `NumGoroutine` 趋势，再通过 pprof 查看大量 goroutine 卡在哪个调用栈。Channel 阻塞本身只会导致已有 goroutine 无法退出；要出现无限增殖，还必须存在持续创建 goroutine 的代码路径。

---

## 四、Go：Channel 关闭后的读写行为

### 1. 关闭后可以读

```go
ch := make(chan int)
close(ch)

v, ok := <-ch
fmt.Println(v, ok) // 0 false
```

从已关闭的 channel 接收时：

- 如果是有缓冲 channel，先读完缓冲区中的数据；
- 缓冲区为空后，继续读取会立即返回元素类型的零值和 `ok=false`；
- 如果是无缓冲 channel，由于没有缓冲数据，关闭后直接返回零值和 `ok=false`。

### 2. 关闭后不能写

```go
ch := make(chan int)
close(ch)

ch <- 1 // panic: send on closed channel
```

向已关闭的 channel 发送数据会 panic。重复关闭 channel 也会 panic。

### 3. 无缓冲 Channel 关闭后的原理

无缓冲 channel 不保存数据，发送和接收必须同时匹配。但关闭操作仍然会修改 channel 的 closed 状态。

关闭后，接收操作发现 channel 已关闭且没有数据，就直接返回：

```text
zeroValue, false
```

因此无缓冲 channel 关闭后仍然可以读，只是永远读不到实际业务数据。

### 4. 其他特殊情况

```go
var ch chan int
```

对于 nil channel：

- 读取：永久阻塞；
- 写入：永久阻塞；
- 关闭：panic。

通常遵循：

> 谁负责发送，谁负责关闭 channel；接收方一般不要关闭 channel。

---

## 五、Go：Slice 的实现原理和自动扩容

### 1. Slice 的底层结构

Slice 本身不是数组，而是一个描述底层数组的结构，通常可以抽象为：

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

包含：

- 指向底层数组的指针；
- 当前长度 `len`；
- 容量 `cap`。

例如：

```go
s := make([]int, 2, 4)
```

表示当前有 2 个元素，底层数组容量为 4，还可以继续追加 2 个元素而不扩容。

### 2. Slice 可能共享底层数组

```go
a := []int{1, 2, 3}
b := a[:2]

b[0] = 100
fmt.Println(a) // [100 2 3]
```

`a` 和 `b` 共享同一个底层数组，因此修改 `b` 可能影响 `a`。

### 3. append 如何扩容

执行 `append` 时：

1. 如果当前容量足够，直接复用原底层数组；
2. 如果容量不足，runtime 分配新的更大数组；
3. 将旧数组元素复制到新数组；
4. 追加新元素；
5. 返回新的 slice header。

因此必须接收 `append` 的返回值：

```go
s = append(s, 1)
```

扩容比例由 Go runtime 决定，不是语言规范保证的固定值。容量较小时通常接近 2 倍增长，容量较大时增长比例会逐渐降低，具体策略可能随 Go 版本变化。

### 4. 常见陷阱

如果容量足够，append 可能修改原数组：

```go
a := make([]int, 2, 4)
b := append(a, 3)
```

此时 `a` 和 `b` 可能仍然共享底层数组。

可以使用三下标切片限制容量：

```go
b := a[:2:2]
```

此时 `len(b) == 2` 且 `cap(b) == 2`，继续 append 会强制分配新的底层数组，避免修改 `a` 的后续容量区域。

nil slice 也可以直接 append：

```go
var s []int
s = append(s, 1)
```

### 面试总结

Slice 是由数组指针、长度和容量组成的描述符。append 时容量足够就复用底层数组，否则由 runtime 分配新的更大数组并复制元素。扩容比例由 runtime 决定，不能简单认为永远是两倍。多个 slice 可能共享底层数组，因此需要注意修改和 append 带来的副作用。


