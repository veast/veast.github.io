1.1 什么是 CSI？​​

CSI 是一个标准接口，它允许容器编排系统（如 Kubernetes）与任意存储系统进行通信，而无需将存储逻辑编译进 Kubernetes 核心代码。简单说，它是一个 ​​"**驱动程序模型**"​。
​在 CSI 之前​：Kubernetes 需要为每一种存储（如 AWS EBS、GCE PD、NFS）编写内嵌的代码（称为 "In-Tree" 插件），非常臃肿且难以维护。
​在 CSI 之后​：存储厂商可以独立开发并发布自己的 CSI 驱动程序，Kubernetes 通过统一的接口调用它们（称为 "Out-of-Tree" 插件）。

——有点像mcp的设计理念


当应用程序（如 kubelet）尝试通过系统调用（如 stat, read, open）访问一个文件或目录时，该路径位于一个由**用户态进程管理的文件系统**上。如果这个用户态进程（对于 CSI，就是 **​CSI Node Driver**）与内核之间的通信通道（即 "transport endpoint"）中断了，内核就会返回 ENOTCONN错误，其描述就是 ​​"Transport endpoint is not connected"​。

​通俗比喻​：
​存储卷就像一个网络共享文件夹。
​CSI Node Driver​ 就像是你的电脑上安装的 ​​"Dropbox" 或 "OneDrive" 客户端软件，负责与远程服务器保持连接和同步。
​​"transport endpoint is not connected"​​ 就相当于这个客户端软件崩溃了或网络断了，导致你点击桌面上的共享文件夹图标时，系统弹窗提示 "无法连接到服务器"。