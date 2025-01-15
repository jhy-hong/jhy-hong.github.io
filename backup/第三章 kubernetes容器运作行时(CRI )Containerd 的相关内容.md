### k8s运行时1.24版本起不再直接支持docker，通常我们会选择Containerd作为k8s的运行时
containerd 的生态系统包括一系列与其集成的工具和组件，这些工具和组件扩展了 containerd 的功能并增强了其适用性。

• CRI 插件：与 Kubernetes 紧密集成，通过实现 Container Runtime Interface (CRI)，使 Kubernetes 能够管理容器。
• CNI 插件：使用 Container Network Interface (CNI) 插件进行网络管理，提供容器的网络连接。
• CSI 插件：Container Storage Interface (CSI) 插件用于存储管理，允许容器挂载和管理存储卷。
• 镜像管理：支持 Docker 镜像和 OCI 镜像规范，提供从[镜像仓库](https://cloud.tencent.com/product/tcr?from_column=20065&from=20065)拉取、存储和管理[容器镜像](https://cloud.tencent.com/product/tcr?from_column=20065&from=20065)的能力。
• 插件机制：允许通过插件扩展 containerd 的功能，满足特定的需求。

### containerd 的优点及其在 k8s中的应用
containerd 是一个工业级的容器运行时，专为性能和稳定性设计。它提供了核心的容器管理功能，如镜像管理、容器执行和存储管理。containerd 的设计目标是简单、高效和与 k8s的深度集成。其主要优点包括：

• 轻量级：containerd 是一个专注于核心功能的轻量级运行时，避免了不必要的复杂性。
• 性能高：containerd 在启动速度和资源使用方面表现出色，非常适合在高并发和大规模环境中使用。
• 稳定性：containerd 经过了广泛的测试和验证，具有很高的可靠性和稳定性。

### 选择 containerd 的原因分析

1.  专注于核心功能：containerd 专注于容器管理的核心功能，避免了过多的功能堆积，保持了运行时的轻量和高效。
2. 深度集成：containerd 与 Kubernetes 的深度集成，使得 Kubernetes 可以更加高效地管理容器和镜像。
3. 社区支持：containerd 由 CNCF (Cloud Native Computing Foundation) 托管，得到了广泛的社区支持和贡献，确保了其长期的维护和发展。
4. 独立性：与 Docker 相比，containerd 更加独立，不依赖于其他工具或平台，减少了系统复杂性。

### 安装Containerd

1. 下载 containerd release 包

```bash
wget https://github.com/containerd/containerd/releases/download/v1.7.24/cri-containerd-cni-1.7.24-linux-amd64.tar.gz
```
2. 解压 release 包
```bash
tar xf cri-containerd-cni-1.7.24-linux-amd64.tar.gz
```
3. 查看containerd安装位置
查看containerd.service文件，了解containerd文件安装位置：
```bash
[root@k8s-master01 ~]# cat etc/systemd/system/containerd.service
# Copyright The containerd Authors.
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

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target dbus.service

[Service]
#uncomment to enable the experimental sbservice (sandboxed) version of containerd/cri integration
#Environment="ENABLE_CRI_SANDBOXES=sandboxed"
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd		查看此位置,把containerd二进制文件放置于此处即可完成安装。

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```
4. 复制containerd运行时文件至系统
 - 查看宿主机/usr/local/bin目录，里面没有任何内容。
```bash
ls /usr/local/bin/
```
 - 查看解压后usr/local/bin目录，里面包含containerd运行时文件
```bash
[root@k8s-master01 ~]# ls usr
local
[root@k8s-master01 ~]# ls usr/local/
bin  sbin
[root@k8s-master01 ~]# ls usr/local/bin
containerd  containerd-shim  containerd-shim-runc-v1  containerd-shim-runc-v2  containerd-stress  crictl  critest  ctd-decoder  ctr
```
  - 复制containerd文件至/usr/local/bin目录中，本次可仅复制containerd一个文件也可复制全部文件。
```bash
[root@k8s-master01 ~]# cp usr/local/bin/containerd /usr/local/bin/
```
5.  添加containerd的service文件到系统
```bash
[root@k8s-master01 ~]#cp etc/systemd/system/containerd.service /usr/lib/systemd/system/containerd.service
```

6. 配置Containerd所需的内核：
```bash
[root@k8s-master01 ~]# cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
加载内核
```bash
[root@k8s-master01 ~]# sysctl --system
```
7.  生成默认配置文件
```bash
[root@k8s-master01 ~]# mkdir -p /etc/containerd
[root@k8s-master01 ~]# containerd config default | tee /etc/containerd/config.toml
```
8.  启动Containerd，并配置开机自启动
```bash
[root@k8s-master01 ~]# systemctl daemon-reload
[root@k8s-master01 ~]# systemctl enable --now containerd
```
### containerd CLI 工具推荐
- ctr
定义和功能 ctr 是一个用于测试和调试 containerd 的轻量级 CLI 工具。它提供了一些基本的命令，可以用于快速验证 containerd 的功能。使用场景及实例 ctr 主要用于开发和测试环境，例如开发者需要验证 containerd 是否正确安装和配置时，可以使用 ctr 运行一些基本的测试命令。
- crictl
定义和功能 crictl 是一个**用于与 Kubernetes CRI (Container Runtime Interface) 交互的 CLI 工具**。它提供了一组命令，可以用于检查和管理通过 CRI 运行的容器和镜像。使用场景及实例 crictl 主要用于 Kubernetes 环境下的容器管理。例如，管理员可以使用 crictl 查看当前运行的容器、检查容器日志或执行容器操作。
crictl需要安装:
    - 到github地址为[crictl下载地址](https://github.com/kubernetes-sigs/cri-tools/releases)安装自己需要的版本
    - 解压并安装
    ```bash
       tar -C /usr/local/bin -xzf crictl-xxx.tar.gz
    ```
    - 配置crictl客户端连接的运行时位置
    ```bash
    [root@k8s-master01 ~]cat > /etc/crictl.yaml <<EOF
        runtime-endpoint: unix:///run/containerd/containerd.sock
        image-endpoint: unix:///run/containerd/containerd.sock
        timeout: 10
        debug: false
        EOF
    ```

- nerdctl 
定义和功能 nerdctl 是专为 Containerd 设计的轻量级命令行工具，它的诞生旨在为那些从 Docker 转向 Containerd 的用户提供一个熟悉的操作界面。使用场景及实例 nerdctl 适合那些熟悉 Docker CLI 但希望使用 containerd 的用户。例如，开发者可以使用 nerdctl 来构建和运行容器，而不需要学习全新的命令集合。
- nerdctl 与 Docker 的区别：

1.  底层与上层之别：最本质的区别在于，Docker 是一个包含了完整容器生命周期管理的平台，包含了 Docker daemon 等多个组件。而 nerdctl 是一个纯粹的命令行工具，专注于与 Containerd 交互，后者作为低层级运行时，处理容器和镜像的实际操作。
2.  兼容与创新：nerdctl 虽然兼容 Docker 的 CLI 习惯，但并不意味着它是 Docker 的简单复制品。它在兼容基础上引入了新功能，如延迟拉取镜像 (lazy-pulling) 和镜像加密等，这些是 Docker 本身不具备的。
3. 性能与效率：由于直接操作 Containerd，避免了 Docker daemon 这一层的额外开销，nerdctl 在某些场景下能提供更好的性能和资源利用率。
4. 定位差异：Docker 适合需要一站式解决方案的用户，而 nerdctl+Containerd 组合更适合那些追求轻量、高性能及与 Kubernetes 紧密集成的环境。
###  构建镜像
安装buildctl 及nerdctl 
- buildctl 安装方式
   - 下载预编译二进制文件：
   ```bash
    [root@k8s-master01 ~]wget https://github.com/moby/buildkit/releases/download/v0.19.0-rc2/buildkit-v0.19.0-rc2.darwin-arm64.tar.gz
    [root@k8s-master01 ~]mkdir /tmp/buildkit/
    [root@k8s-master01 ~]tar xf buildkit-v0.19.0-rc2.darwin-arm64.tar.gz -C /tmp/buildkit/
    [root@k8s-master01 ~]mv /tmp/buildkit/bin/* /usr/bin/
   ```
    - 配置service文件
    ```bash
   cat /lib/systemd/system/buildkit.service
   #########################################
   [Unit]
   Description=BuildKit
   Requires=buildkit.socket
   After=buildkit.socket
   Documentation=https://github.com/moby/buildkit
   
   [Service]
   # **注意这个地方的相关配置**
   
   ExecStart=/usr/local/bin/buildkitd --oci-worker=false --containerd-worker=true
   
   [Install]
   WantedBy=multi-user.target
   
    ```
   
   - 启动服务
   ```bash
    [root@k8s-master01 ~]systemctl enable buildkit.service
    [root@k8s-master01 ~]systemctl start buildkit.service
   
- nerdctl 安装方式
   
   ```bash
     [root@k8s-master01 ~]wget https://github.com/containerd/nerdctl/releases/download/v2.0.2/nerdctl-2.0.2-linux-arm64.tar.gz
     [root@k8s-master01 ~]tar xvf nerdctl-2.0.2-linux-arm64.tar.gz
     [root@k8s-master01 ~]cp nerdctl /usr/bin/
    [root@k8s-master01 ~]nerdctl version
  ```
  
- 构建镜像时有可能会出现镜像拉不到可以配置构建时的镜像
  **修改 BuildKit 配置文件**

     1. 创建或编辑 BuildKit 配置文件：

   ```bash
   sudo mkdir -p /etc/buildkit
   sudo vim /etc/buildkit/buildkitd.toml
   ```

  2. 添加以下内容以设置国内镜像源（以阿里云为例）：

   ```bash
   [registry."docker.io"]
      mirrors = ["https://docker.1ms.run","https://docker.nastool.de","https://dockerpull.org"]
  
  [registry."quay.io"]
     mirrors = ["https://quay.m.daocloud.io"]
  
   ```
  当然你也可以用自己仓库的镜像去构建

  3. 保存并退出。
### Containerd 的镜像有命名空间划分
   自己构建的镜像不存在于k8s所管理的命名空间下

  **列出所有命名空间**
       ```bash
       sudo ctr namespaces list
       ```

​      示例输出：

        default
        k8s.io
     

**查看特定命名空间的镜像**
    

```bash
    sudo ctr -n k8s.io images list
    sudo ctr -n default images list
```
确保在 `k8s.io` 命名空间中查看与 Kubernetes 相关的镜像。
    **解决办法**：
       **1. 使用 `--namespace` 参数指定命名空间**

```bash
    nerdctl --namespace k8s.io build -t my-image:latest .
```
​      **参数说明**
​          `--namespace k8s.io`：将镜像构建并存储到 `k8s.io` 命名空间中。
​          `-t my-image:latest`：为构建的镜像指定名称和标签。
​           `.`：构建上下文目录。
​       **2. 设置默认命名空间**
​          如果需要始终在特定命名空间下操作，可以设置环境变量：

```bash
    export CONTAINERD_NAMESPACE=k8s.io
```
​         此后所有 `nerdctl` 命令都会默认使用 `k8s.io` 命名空间：
          ```bash
              nerdctl build -t my-image:latest .`
          ```


​    **3. 将镜像导入到指定命名空间**
​    
​      如果需要将已有镜像导入到不同命名空间：
​    

​       1. **导出镜像**：

```bash
    nerdctl --namespace default save my-image:latest -o my-image.tar
```

​     2.  **切换命名空间并导入镜像**：
```bash
    nerdctl --namespace k8s.io load < my-image.tar
```

​     3. **使用 BuildKit 构建**

结合 BuildKit 构建镜像时也可直接指定命名空间：

```bash
  nerdctl --namespace k8s.io build --buildkit-enabled -t my-buildkit-image:latest .
```
**总结**

  通过 `--namespace` 参数和环境变量设置，可以轻松指定和管理 `containerd` 中的命名空间。在 Kubernetes 环境下使用 `k8s.io` 命名空间有助于与 Kubernetes 管理的容器镜像保持一致，从而提升构建和管理的灵活性与一致性。

### 也可以配置containerd的镜像源
​     参考如下：[containerd的镜像源设置方式](https://zhuanlan.zhihu.com/p/702497587)