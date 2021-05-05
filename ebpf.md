#	[eBPF](https://ebpf.io/) introduction
titanxxh@gmail.com
--
kubecon 2020 eBPF 相关session
-	Beyond the Buzzword BPF’s Unexpected Role in Kubernetes
-	Designing a gRPC Interface for Kernel Tracing with eBPF
-	<span class="fragment highlight-blue">eBPF and Kubernetes Little Helper Minions for Scaling Microservices —— Cilium</span>
-	Hubble - eBPF Based Observability for Kubernetes
-	Tutorial Using BPF in Cloud Native environments
--
Agenda
-	what is eBPF?
-	what can eBPF do?
---
what is eBPF?
--
Linux **内核**中一个高度灵活与高效的类虚拟机组件，
它以一种**安全**的方式在许多 **hook** 点执行**受限**的 bytecode。 

Note:
eBPF in kernel -> js in HTML, actually more like the v8 engine to run js code.
--
the virtual machine with a total of <span class="fragment highlight-red">11 64-bit registers</span>, a program counter and a <span class="fragment highlight-red">512 byte fixed-size stack</span>.

maximum instruction limit per eBPF program is restricted to <span class="fragment highlight-red">4096</span>.
--
-	r0:	stores return values, both for function calls and the current program exit code
-	r1-r5:	used as function call arguments, upon program start r1 contains the "context" argument pointer
-	r6-r9:	these get preserved between kernel function calls
-	r10:	read-only pointer to the per-eBPF program 512 byte stack
--
-	BPF map：高效的 key/value 仓库
-	辅助函数（helper function）：可以更方便地利用内核功能或与内核交互
-	尾调用（tail call）：高效地调用其他 BPF 程序
-	支持 BPF offload（例如 offload 到网卡）的基础设施
-	...
--
![](images/linux_ebpf_internals.png)
--
![eBPF hooks](images/eBPF hooks.png)
--
Trivia

-	eBPF 是近年来linux的重大特性，一开始linus拒绝合入，后来将commit拆分后才合入。<!-- .element: class="fragment" -->
-	eBPF 能合入linux主线主要归功于在kernel内部建立一系列向后兼容的抽象，正如kernel和userspace之间的接口一样。<!-- .element: class="fragment" -->
-	eBPF 近年来已经成为了kernel中最活跃的子系统。<!-- .element: class="fragment" -->
--
extended readings
-	[BPF Features by Linux Kernel Version](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)
-	[an ebpf overview](https://www.collabora.com/news-and-blog/blog/2019/04/05/an-ebpf-overview-part-1-introduction/)

---
what can eBPF do?
--
-	networking
-	tracing
-	more...
---
replace netfilter/iptables
--
![iptables](images/hooks-and-tables.png)
--
iptables的问题:
-	只能全量更新
-	O(n)复杂度的rules应用
-	基于IP和端口，无法感知L7的协议
-	CPU消耗高、延迟大
--
even worse in K8S
-	kube-proxy - the component which implements Services and load balancing by DNAT iptables rules
-	the most of CNI plugins are using iptables for Network Policies
--
![](images/flow-iptables.png)
--
![](images/flow-local.png)
--
![](images/flow-forward.png)

--
how eBPF solve this?
-	eBPF hash map (per cpu)
-	kTLS
-	bypass netfilter
-	offload to hardware(XDP)
--
extended readings
-	[深入理解 Cilium 的 eBPF 收发包路径](https://arthurchiao.art/blog/understanding-ebpf-datapath-in-cilium-zh/)
---
accelerate sidecar
--
![sidecar without eBPF](images/sidecar no eBPF.png)<!-- .element height="80%" width="80%" -->
--
![sidecar with eBPF](images/sidecar eBPF.png)<!-- .element height="80%" width="80%" -->
---
Cilium's service LB
--
![Cilium's service load balancing](images/cilium.png)
--
东西向流量在Socket层的BPF处理
-	将 Service 的 `IP:Port` 映射到具体的后端的 `PodIP:Port`，并做负载均衡。
-	当应用发起 `connect`、`sendmsg`、`recvmsg` 等请求（系统调用）时，拦截这些请求，并根据请求的 `IP:Port` 映射到后端 pod，直接发送过去。反向进行相反的变换。
-	不需要包级别（packet leve）的地址转换（NAT）。在系统调用时，还没有创建包，因此性能更高。
-	省去了 kube-proxy 路径中的很多中间节点（intermediate node hops）
--
南北向流量在XDP或者tc层的BPF处理
-	与 socket 层的差不多，将 Service 的 `IP:Port` 映射到后端的 `PodIP:Port`。
-	进行DNAT转换，可以在XDP做也可以在tc做，XDP做性能更高
--
-	将东西向流量放在离 socket 层尽量近的地方做。
-	将南北向流量放在离驱动（XDP 和 tc）层尽量近的地方做。
--
![](images/perf1.png)
--
![](images/perf2.png)
--
![](images/perf3.png)

Note:
kube-proxy 不管是 iptables 还是 ipvs 模式，都在处理软中断上消耗了大量 CPU。
--
extended readings
-	[基于 BPF/XDP 实现 K8s Service 负载均衡](https://arthurchiao.art/blog/cilium-k8s-service-lb-zh/)
---
bcc & bpftrace
--
用户可以写一个python程序和一个c程序
-	python程序，用户态运行，收集数据输出。
-	c程序会编译后加载到kernel中，收集event或者写入map。
--
![](images/bcc.png)
--
bcc tools
![](images/bcc_tracing_tools_2019.png)
--
![](images/execnoop.png)
![](images/uvp.png)
--
[off-cpu analysis](http://www.brendangregg.com/offcpuanalysis.html)

where time is spent waiting while blocked on I/O, locks, timers, paging/swapping, etc.
--
![off cpu thread states](http://www.brendangregg.com/Perf/thread_states.png)
--
[example of off&wake](http://www.brendangregg.com/FlameGraphs/offwake-mysqld1.svg)

![](http://www.brendangregg.com/FlameGraphs/offwake-mysqld1.png)<!-- .element height="50%" width="50%" -->
--
![](images/ebpf_tracing_landscape_jan2019.png)
--
bpftrace probe types

![](images/bpftrace_probes_2018.png)<!-- .element height="80%" width="80%" -->
--
extended readings
-	[Linux Extended BPF (eBPF) Tracing Tools](http://www.brendangregg.com/ebpf.html)
-	[Learn eBPF Tracing: Tutorial and Examples](http://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html)
-	[on/off cpu flamegraphs](http://www.brendangregg.com/flamegraphs.html)
-	[bcc tutorial](https://github.com/iovisor/bcc/blob/master/docs/tutorial.md)
-	[bpftrace tutorial](https://github.com/iovisor/bpftrace/blob/master/docs/tutorial_one_liners.md)
---
## Thanks!
--
<!-- .slide: data-background-image="images/replace-linux.png" -->
