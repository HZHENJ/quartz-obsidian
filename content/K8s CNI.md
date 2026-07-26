%% [[K8s]] %%
# 1.CNI是什么

在k8s CNI，Container Network Interface，即容器的网络的API接口，是一个标准化的网络模型，负责定义容器如何在网络层面进行交互和通信，在k8s中通过CNI来拓展网络功能。

CNI 采用一种插件化机制，支持集成不同的网络方案。在容器创建过程中，Kubelet 组件通过调用所需的 CNI 插件，即可使用相应的网络方案完成容器的网络配置，实现灵活的容器网络连接、网络资源释放等功能。常见的 CNI 插件包括 Calico、Flannel、Cilium 等。

由此，CNI 通过标准化接口和插件化机制，为 Container Runtime（容器运行时）和 CNI 插件之间提供了一个通用接口，增强了Kubernetes 网络的灵活性和可扩展性。

如果没有CNI，那么我们需要手动执行大量需要大量手工工作的事情，例如：创建接口（物理网络接口和虚拟网络接口）、创建 veth pair、设置命名空间网络等

# 2.CNI配置文件（Config File）格式

CNI配置文件采用Json格式，默认放置在：

- 配置文件路径
- 插件二进制路径

配置文件定义了CNI运行时需要调用的插件、插件相关参数、IPAM分配方式等内容。

1. 主配置字段
    1. cniVersion - string：使用CNI规范版本，如 `"0.4.0"` 或 `"1.0.0"`。
    2. name - string：定义当前网络的名称，例如`mynet`
    3. disableCheck - boolean：是否关闭`CHECK`操作，默认为`false`
    4. plugins - list：核心字段，数组，每个元素代表一个CNI插件及其配置，K8s通常依次执行这些插件，例如：
        1. bridge
        2. tuning
        3. portmap
2. 插件配置字段，不同插件支持的字段不同，但所有插件共享的
    1. 必选字段为：
        1. type - string：插件的名字，及对应/opt/cni/bin下的二进制文件名，例如`"bridge"`, "sriov", "macvlan"。
    2. 可选字段为：
        1. capabilities - dict：声明插件支持的拓展能力，例如MAC地址设置，端口映射等
        2. ipMasq - boolean：当容器启用SNAT（地址伪装：即 outbound masquerade），当容器通过宿主机作为网关出网时，该字段通常必须开启，否则容器的私网IP无法被外部识别，回包无法返回，导致包被丢弃
        3. ipam - dictionary：IP地址管理模块（核心），IPAM模块负责为Pod分配IP地址，网关信息，路由信息等，常见的IPAM插件：
            1. host-local
            2. dhcp
            3. static

以host-local为例：

- **type** 指定使用的 IPAM 插件，如 `"host-local"`。
- **subnet** IP 地址段，CIDR 格式，如 `"10.15.10.0/24"`。
- **routes** 容器内部自动写入的路由规则，示例：`{"dst": "0.0.0.0/0"}dst`: 目标网段`gw`: 网关 IP（省略则默认使用网关字段）
- rangeStart / rangeEnd：可分配地址的范围限制。
- gateway：网关地址。若省略，则默认从子网中选择第一个可用地址作为网关。
- dns - dictionary：包括 nameservers、domain、search、options 等。

配置文件：

```JSON
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "keyA": ["some more", "plugin specific", "configuration"],
      "ipam": {
        "type": "host-local",
        "subnet": "10.1.0.0/16",
        "gateway": "10.1.0.1",
        "routes": [
          {"dst": "0.0.0.0/0"}
        ]
      },
      "dns": {
        "nameservers": [ "10.1.0.1" ]
      }
    },
    {
      "type": "tuning",
      "capabilities": { "mac": true },
      "sysctl": {
        "net.core.somaxconn": "500"
      }
    },
    {
      "type": "portmap",
      "capabilities": { "portMappings": true }
    }
  ]
}
```

# 3. 基本工作流程

每当K8s需要创建或者删除Pod时，Kubelet会通过CNI调用插件，完成网络设置

1. CNI配置文件加载：Kubelet在启动Pod时会读取CNI配置
    1. 路径：/etc/cni/net.d/
    2. 决定要执行哪些插件，以及插件执行顺序
    3. 决定如何分配IP（IPAM插件）
2. 调用插件（ADD/CHECK/DEL/VERSION）：Kubelet必须至少调用ADD/DEL用于创建/删除Pod
3. 创建veth Pair虚拟网卡对：CNI插件在宿主机与容器之间创建一个veth pair
    1. 容器侧eth0是容器的默认网卡
    2. 宿主机侧veth接入bridge或overlay网络
4. 调用IPAM分配地址，通用IPAM模块实现：
	- 分配容器IP
	- 分配网关
	- 写入路由规则
	- 分配可能需要的DNS
	例如host-local会维护：`/var/lib/cni/networks/<network-name>/`并记录已分配的IP
5. 配置网络规则，CNI插件会在容器内设置默认路由，例如`default via 10.1.0.1`，根据routes字段写入细粒度路由，根据ipMasq设置SNAT/masquerade规则，一些插件还会创建NAT端口映射规则。最终保证Pod与Pod，Pod与Service，Pod与外部网络（SNAT）通信链路完整可用

# 4.CNI操作

CNI定义了5种操作，ADD/DEL/CHECK/GC/VERSION，这些操作通过`CNI_COMMAND`环境变量来传递给CNI插件。

1. ADD - 将容器添加到网络中，或应用修改。
2. DEL - 从网络中删除容器，或取消应用修改。
3. CHECK - 如果容器的网络出现问题，则返回错误。
4. VERSION - 显示插件的版本支持
5. GC - 清理所有无用陈旧（stale）的资源

具体内容请参考：[cni-operations](https://github.com/containernetworking/cni/blob/master/SPEC.md#cni-operations)

# 5.CNI插件

CNI插件通常有三种实现模式：

1. Overlay
2. 路由
3. Underlay

# 6. 参考

- [cni/SPEC.md at main · containernetworking/cni](https://github.com/containernetworking/cni/blob/main/SPEC.md#add-add-container-to-network-or-apply-modifications)
- [CNI接口介绍 | kubernetes-notes](https://k8s.huweihuang.com/project/network/cni/cni)
- [【K8s】专题十五（2）：Kubernetes 网络之 CNI_kubelet调用cni-CSDN博客](https://blog.csdn.net/2401_82795112/article/details/143569949)
- [从零开始入门 K8s：理解 CNI 和 CNI 插件_服务革新_溪恒_InfoQ精选文章](https://www.infoq.cn/article/6mdfwwghzadihiq9ldst)