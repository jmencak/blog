# Demystifying node-level tuning on OpenShift

by Jiří Mencák

Typically,
[Red Hat OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift/container-platform)
(RHOCP) administrators do not need to worry about node-level tuning because
the platform comes installed with reasonable defaults to run general-purpose
workloads. However, there are scenarios where intervention is needed to
improve workload performance. While most of the time, this intervention will
be done by cluster administrators as a post-installation (day 2) configuration
change, it might even be necessary
[at cluster installation](https://docs.openshift.com/container-platform/4.11/scalability_and_performance/sno-du-enabling-workload-partitioning-on-single-node-openshift.html)
(day 1).

The aim of this blog post is to give an overview of the default node-level
tuning done on an OpenShift cluster and the extra options cluster
administrators have to apply node-level tuning to tailor the performance of
the platform to their needs.

## Option 0: Do nothing
The OpenShift platform, like Red Hat Enterprise Linux, comes tuned by default
for general-purpose workloads. The system tuning is primarily performed by
[the Node Tuning Operator](https://github.com/openshift/cluster-node-tuning-operator)
(NTO), which is one of the core OpenShift operators.

Many of the tunables in the parent
[`openshift`](https://github.com/redhat-performance/tuned/blob/master/profiles/openshift/tuned.conf)
profile raise certain kernel limits. This improves how the system functions
during higher system loads and cluster scale. On the other hand, the changes
mostly come at the cost of increased kernel memory consumption.

```
[main]
summary=Optimize systems running OpenShift (parent profile)
include=${f:virt_check:virtual-guest:throughput-performance}

[selinux]
avc_cache_threshold=8192      # rhbz#1548428, PR10027

[net]
nf_conntrack_hashsize=1048576 # PR413 (the default limit is too low for OpenShift)

[sysctl]
net.ipv4.ip_forward=1         # Forward packets between interfaces
kernel.pid_max=>4194304       # PR79, for large-scale workloads; systemd sets kernel.pid_max to 4M since v243
fs.aio-max-nr=>1048576        # PSAP-900
net.netfilter.nf_conntrack_max=1048576
net.ipv4.conf.all.arp_announce=2           # rhbz#1758552 pod communication due to ARP failures
net.ipv4.neigh.default.gc_thresh1=8192
net.ipv4.neigh.default.gc_thresh2=32768
net.ipv4.neigh.default.gc_thresh3=65536    # rhbz#1384746 gc_thresh3 limits no. of nodes/routes
net.ipv6.neigh.default.gc_thresh1=8192
net.ipv6.neigh.default.gc_thresh2=32768
net.ipv6.neigh.default.gc_thresh3=65536
vm.max_map_count=262144                    # rhbz#1793714 ElasticSearch (logging)

[sysfs]
/sys/module/nvme_core/parameters/io_timeout=4294967295
/sys/module/nvme_core/parameters/max_retries=10

[scheduler]
# see rhbz#1979352; exclude containers from aligning to house keeping CPUs
cgroup_ps_blacklist=/kubepods\.slice/
# workaround for rhbz#1921738
runtime=0
```

In the [`openshift`](https://github.com/redhat-performance/tuned/blob/master/profiles/openshift/tuned.conf)
profile, we mostly build on the
[`throughput-performance`](https://github.com/redhat-performance/tuned/blob/master/profiles/throughput-performance/tuned.conf)
profile, which is the default profile
[recommended](https://github.com/redhat-performance/tuned/blob/420267fa3463ef4a06885847c7ca4cd24c781dc8/recommend.conf#L56)
for servers. Moreover, we include other functional and performance tunables. The functional ones:

- Fix OpenShift
  [pod communication issues](https://bugzilla.redhat.com/show_bug.cgi?id=1758552)
  due to ARP failures between a node and its pods
- [Increase `vm.max_map_count`](https://bugzilla.redhat.com/show_bug.cgi?id=1793714)
  to enable clean start of Elasticsearch pods
- [Adjust the `kernel.pid_max`](https://github.com/openshift/cluster-node-tuning-operator/pull/79)
  for large-scale workloads (also helps with pod density)

The performance tunables:

- Allow large-size clusters and more than 1000 of
  [routes](https://docs.openshift.com/container-platform/4.11/rest_api/network_apis/route-route-openshift-io-v1.html)
- Adjust the sizes of
  [netfilter connection tracking hash table](https://github.com/openshift/cluster-node-tuning-operator/pull/79)
  and its maximum entries
- Improve node performance (CPU utilization) by adjusting the AVC cache threshold
  ([1](https://bugzilla.redhat.com/show_bug.cgi?id=1548428),
   [2](https://github.com/openshift/openshift-ansible/pull/10027))
- Enable [more VMs running on RHOCP](https://issues.redhat.com/browse/PSAP-900)
  nodes
- [Prevent the TuneD `[scheduler]` plug-in to align containers to housekeeping CPUs](https://bugzilla.redhat.com/show_bug.cgi?id=1979352)
- Disable the [dynamic behavior of the TuneD `[scheduler]` plug-in](https://bugzilla.redhat.com/show_bug.cgi?id=1921738)

In the current versions of RHOCP, the
[`openshift-control-plane`](https://github.com/redhat-performance/tuned/blob/master/profiles/openshift-control-plane/tuned.conf)
simply inherits from the `openshift` profile. As for
[`openshift-node`](https://github.com/redhat-performance/tuned/blob/master/profiles/openshift-node/tuned.conf)
profile, both `fs.inotify` settings are functional settings that, at this point,
only mirror the settings
[already provided](https://github.com/openshift/machine-config-operator/blob/master/templates/common/_base/files/sysctl-inotify.conf.yaml)
by MCO to enable their application
before kubelet start. The `net.ipv4.tcp_fastopen=3` setting reduces network
latency by enabling data exchange during the sender's initial TCP SYN on
client and server connections.

## Option 1: I need _some_ custom tuning
In a [previous blog post](https://cloud.redhat.com/blog/node-tuning-operator-and-friends-in-openshift-4.5),
we covered how to apply custom node-level
configuration using NTO. Here we will give an example Tuned CR for tuning
a system with a 10 Gigabit Intel(R) network interface card for throughput,
as suggested by
[the Linux kernel documentation for the ixgb driver](https://www.kernel.org/doc/html/latest/networking/device_drivers/ethernet/intel/ixgb.html).

```yaml
apiVersion: tuned.openshift.io/v1
kind: Tuned
metadata:
  name: openshift-network-tuning
  namespace: openshift-cluster-node-tuning-operator
spec:
  profile:
  - data: |
      [main]
      summary=Increase throughput for NICs using ixgb driver
      include=openshift-node
      [sysctl]
      ### CORE settings (mostly for socket and UDP effect)
      # Set maximum receive socket buffer size.
      net.core.rmem_max = 524287
      # Set maximum send socket buffer size
      net.core.wmem_max = 524287
      # Set default receive socket buffer size.
      net.core.rmem_default = 524287
      # Set default send socket buffer size.
      net.core.wmem_default = 524287
      # Set maximum amount of option memory buffers.
      net.core.optmem_max = 524287
      # Set number of unprocessed input packets before kernel starts dropping them.
      net.core.netdev_max_backlog = 300000
    name: openshift-network-tuning
  recommend:
  - match:
    - label: node-role.kubernetes.io/worker
    priority: 20
    profile: openshift-network-tuning
```

Note that the core network settings only are included for brevity.

## Option 2: I need low-latency/real-time tuning
Some specialized workloads require low-latency/real-time tuning, such as
Telco 5G Core User Plane Function (UPF), Financial Services Industry (FSI),
and some High-Performance Computing (HPC) workloads. However, such tuning
requires sacrifices. Be it a loss of overall throughput when using the
real-time kernel, using more power or statically partitioning your system
into housekeeping and workload partitions. Static partitioning counteracts
the OpenShift Kubernetes platform’s efficient use of computing resources
and might oversubscribe the housekeeping partitions. Partitioning needs to
happen on many different levels and by coordination of various components:

- [Separating management and workload](https://docs.openshift.com/container-platform/latest/scalability_and_performance/sno-du-enabling-workload-partitioning-on-single-node-openshift.html)
  pods
- Using [Guaranteed pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
  for workloads
- Separating system processes away from workload CPUs
- Moving kernel threads to housekeeping CPUs
- Moving network interface controller (NIC) IRQs to housekeeping CPUs

Apart from partitioning, there are other ways of reducing latency in software:
- Using real-time kernel
- Using huge pages (per NUMA node) to avoid the cost of TLB misses
- [Disabling of CPU load balancing for DPDK](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Disabling CPU CFS quota](https://docs.openshift.com/container-platform/4.11/scalability_and_performance/cnf-low-latency-tuning.html#disabling-cpu-cfs-quota_cnf-master)
- Possibly disabling hyperthreading to reduce variations in latency
- BIOS tuning

All of the above can be performed manually, however, great care needs to be
taken to perform them coherently. This is where NTO’s Performance Profile
controller comes in. It acts as an orchestrator that takes the burden out of
manual configuration and makes sure that all the components (kernel, TuneD,
Kubelet [CPU, Topology and Memory manager], CRI-O) necessary to perform the
above tasks are properly configured based on a given
[PerformanceProfile](https://docs.openshift.com/container-platform/4.11/scalability_and_performance/cnf-low-latency-tuning.html#disabling-cpu-cfs-quota_cnf-master).

This is an example of a PerformanceProfile
[single-node OpenShift](https://docs.openshift.com/container-platform/4.11/scalability_and_performance/cnf-low-latency-tuning.html#disabling-cpu-cfs-quota_cnf-master)
deployment:

```yaml
apiVersion: performance.openshift.io/v2
kind: PerformanceProfile
metadata:
  name: performance
spec:
  cpu:
    isolated: "2-31,34-63"
    reserved: "0-1,32-33"
  globallyDisableIrqLoadBalancing: false
  hugepages:
    defaultHugepagesSize: "1G"
    pages:
    - size: "1G"
      count: 16
      node: 0
  net:
    userLevelNetworking: false
    devices: []
  nodeSelector:
    node-role.kubernetes.io/master: ""
  numa:
    topologyPolicy: "best-effort"
  realTimeKernel:
    enabled: true
```

The PerformanceProfile above will assign CPUs 2-31 and 34-63 for
low-latency workloads and reserve the remaining 4 CPUs for system
housekeeping tasks. In some cases, the reserved CPUs are insufficient
to handle device interrupts. For this reason, the example above
allows interrupt processing on isolated (tuned and ready for sensitive
workload) CPUs by setting `globallyDisableIrqLoadBalancing` to `false`.
However, the IRQs load balancing can be disabled per pod CPUs when
`using irq-load-balancing.crio.io/cpu-quota.crio.io=disable` annotations.
Additionally, this example profile also provides 16 1GiB huge pages on NUMA
node 0;
[it does not restrict the number of NIC queues to the number of reserved CPUs](https://docs.openshift.com/container-platform/4.11/scalability_and_performance/cnf-low-latency-tuning.html#disabling-cpu-cfs-quota_cnf-master),
enables
[Topology Manager](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)
`best-effort` NUMA alignment policy and enables real-time kernel. Other system
changes performed by the PerformanceProfile controller are implicitly
configured by the controller. For example:

- Setting [CPU Manager](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/)
  policy static to enable exclusive allocation of CPUs
- Enforcing allocation of
  [full physical cores](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy-options)
  when topology policy is restricted or single-numa-node
- Setting CPU Manager reconcile period (the shorter the reconcile period,
  the faster the CPU Manager prevents non-Guaranteed pods to run on isolated
  CPUs, at cost of more system resources)
- Setting [Memory Manager](https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/)
  policy (when topology policy is restricted or single-numa-node) to pin
  memory and huge pages closer to the allocated CPUs
- Creating [high-performance handler](https://github.com/openshift-kni/performance-addon-operators/blob/master/build/assets/configs/99-runtimes.conf)
  and RuntimeClass for CRI-O (note the `runtimeClassName` in the pod specification below)
- Enabling and starting [stalld](https://gitlab.com/rt-linux-tools/stalld)
  via TuneD

At this point the reader might believe that all the configuration to achieve
low latency goes into PerformanceProfile. However, this is not the case.
We also need to make appropriate adjustments to the low latency workload
pod specification itself. Specifically, making sure the pod is placed into
the Guaranteed resource class, adding user-requested CRI-O annotations,
and specifying the predefined runtime class.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
  # Disable CFS cpu quota accounting
  cpu-quota.crio.io: "disable"
  # Disable CPU balance with CRIO
  cpu-load-balancing.crio.io: "disable"
  # Opt-out from interrupt handling
  irq-load-balancing.crio.io: "disable"
spec:
  # Map to the correct performance class
  runtimeClassName: get-from-performance-profile
  ...
  containers:
  - name: container-name
    image: image-registry/image
    resources:
      limits:
        memory: "2Gi"
        cpu: "16"
  ...
```

It is important to note that NTO's Performance Profile controller will
overwrite any custom Kubelet changes. However, it is possible to add the
custom Kubelet changes to the
[PerformanceProfile annotation](https://access.redhat.com/solutions/5532341).
Similarly, it is also possible to add
[extra TuneD configuration](https://access.redhat.com/solutions/5532341)
to override or build on top of the Performance Profile controller generated
one.

Configuration by PerformanceProfiles adds more partitioning to the RHOCP
system. This makes sense for DPDK applications that process network packets
in user space and cannot afford HW interruptions. Similarly, it will also apply
to other latency-sensitive applications. However, the extra partitioning has
a cost. Reserved cores can be wasted unnecessarily or not be sufficient to
run the OS and/or RHOCP management pods. Therefore, careful planning and
testing is always necessary when partitioning RHOCP in this way.

## Summary
The RHOCP administrators have multiple options to tune their nodes
for performance. There are a few key considerations to keep in mind.
Firstly, can the node-level tuning be performed after cluster installation,
or do we need to think about the tuning at cluster installation time?
The vast majority of tuning can be performed as a post-installation step.
Custom tuning by NTO and Tuned Profiles falls into this category. Secondly,
can we afford to trade off underutilization or overcommitment of some CPUs
by strictly partitioning our RHOCP cluster to avoid noisy neighbors? If so,
consider using NTO's PerformanceProfiles. And lastly, we need to ask ourselves
how many times will our nodes reboot when choosing our way of tuning.
