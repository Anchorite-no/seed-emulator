# SEED Emulator 技术白皮书：源码级深度逆向分析

## 1. 上帝视角 (Architecture Overview)

SEED Emulator 是一个典型的“定义-渲染-编译”架构的网络仿真框架。它将复杂的网络拓扑定义（Python 对象图）与底层的执行环境（Docker 容器编排）彻底解耦。

### 核心类架构

系统的核心由 `Emulator`（大脑）、`Layer`（功能分层）、`Service`（应用服务）和 `Node`（执行单元）构成。

```mermaid
classDiagram
    class Emulator {
        +Registry registry
        +LayerDatabase layers
        +BindingDatabase bindings
        +render()
        +compile()
    }

    class Layer {
        <<abstract>>
        +configure(emulator)
        +render(emulator)
    }

    class Service {
        +install(node)
    }

    class Node {
        +str name
        +int asn
        +List~Interface~ interfaces
        +List~File~ files
        +appendStartCommand()
    }

    class Router {
        +addProtocol()
        +addTable()
    }

    class Compiler {
        <<interface>>
        +compile(emulator)
    }

    class DockerCompiler {
        +_doCompile()
        +_compileNode()
    }

    Emulator *-- Layer
    Emulator *-- Node
    Layer <|-- Service
    Layer <|-- Base
    Layer <|-- Routing
    Node <|-- Router
    Service ..> Node : Installs on
    Emulator ..> Compiler : Uses
    Compiler <|-- DockerCompiler
```

### 目录结构解析

*   **大脑 (The Brain):** `seedemu/core/`
    *   `Emulator.py`: 总指挥。负责管理所有的层（Layers）和节点（Nodes），并调度渲染流程。
    *   `Node.py`: 容器的抽象表示。它不直接操作 Docker，而是存储“我需要运行什么命令”、“我有哪些文件”等元数据。
    *   `Layer.py`: 功能模块的基类。比如 `Ospf` 层负责给节点添加 OSPF 配置，`Base` 层负责创建基础网络。

*   **四肢 (The Limbs):** `seedemu/compiler/`
    *   `Docker.py`: 这是将 Python 对象转化为现实的关键。它读取 `Emulator` 中的数据，生成 `docker-compose.yml`、`Dockerfile` 和启动脚本。

---

## 2. 核心机制解密 (The Internals)

当用户在 Python 脚本中定义好网络拓扑后，Emulator 并非立即创建容器，而是经历了一个严密的转化过程。

### 从 Graph 到 Container

这一过程主要发生在 `seedemu/compiler/Docker.py` 的 `_doCompile` 方法中：

1.  **网络编译 (`_compileNet`)**:
    Compiler 遍历注册表中的 `Network` 对象，将其转化为 `docker-compose.yml` 中的 `networks` 定义。
    *   *自管理网络 (Self-Managed Network)*: 如果启用了 `selfManagedNetwork`，Compiler 会为网络分配一个“哑前缀”（Dummy Prefix，如 `10.128.0.0/9`），并在容器启动时通过脚本替换为真实的仿真 IP。这解决了 Docker IPAM 难以处理复杂 BGP/MPLS 网络的问题。

2.  **节点编译 (`_compileNode`)**:
    这是最复杂的部分。Compiler 为每个节点创建一个独立的目录（Build Context）。
    *   **Dockerfile 生成**: 调用 `_computeDockerfile`，基于节点的 `BaseSystem`（如 `seedemu-router`）生成 Dockerfile。它会自动注入 `RUN apt-get install` 来安装用户请求的软件。
    *   **文件系统映射**: `Node` 对象中存储的虚拟文件（如 `/etc/bird/bird.conf`）会被写入到宿主机的临时目录，并通过 Dockerfile 的 `COPY` 指令打入镜像。

### 网络层魔法 (Networking)

SEED Emulator 并不是直接调用 `ip link add`，而是利用 Docker 的网络驱动加上精妙的容器内脚本。

**底层 Linux 命令示例**:
当容器启动时，`seedemu/layers/Base.py` 注入的 `/interface_setup` 脚本会被执行。这个脚本读取 `/ifinfo.txt`（由 Compiler 生成的元数据），并执行如下操作：

```bash
# 重命名接口以匹配仿真拓扑（Docker 默认生成 eth0, eth1...）
ip link set eth0 down
ip link set eth0 name net0
ip link set net0 up

# 使用 TC (Traffic Control) 模拟网络延迟和丢包
tc qdisc add dev net0 root handle 1:0 tbf rate 100Mbit buffer 1000000 limit 1000
tc qdisc add dev net0 parent 1:0 handle 10: netem delay 10ms loss 0%
```

---

## 3. 协议栈注入 (BGP/OSPF)

SEED Emulator 的强大之处在于它能自动配置复杂的路由协议。这是通过“模板注入”实现的。

### 路由软件打包

在 `docker_images/seedemu-router/Dockerfile` 中，我们看到路由器的基础镜像安装了 `bird2`：

```dockerfile
FROM handsonsecurity/seedemu-base
RUN apt-get install -y --no-install-recommends bird2
```

### 配置文件生成 (Config Injection)

`bird.conf` 的生成是一个动态累加的过程，主要依赖 `seedemu/core/Node.py` 中的 `Router` 类。

1.  **初始化**: `Routing` 层 (`seedemu/layers/Routing.py`) 在 `configure` 阶段，会调用 `_configure_bird_router`，生成基础的 BIRD 配置（Kernel 协议、Device 协议）。

    ```python
    # seedemu/layers/Routing.py
    rnode.setFile("/etc/bird/bird.conf", RoutingFileTemplates["rnode_bird"].format(...))
    ```

2.  **协议注入**: 当 `Ospf` 或 `Ibgp` 层渲染时，它们会调用 `router.addProtocol()`。

    ```python
    # seedemu/core/Node.py
    def addProtocol(self, protocol, name, body):
        self.appendFile("/etc/bird/bird.conf",
            RouterFileTemplates["protocol"].format(...))
    ```

3.  **挂载**: 最终，`Compiler` 将内存中拼接好的 `bird.conf` 字符串写入宿主机的节点构建目录，并在 Dockerfile 中将其 `COPY` 到 `/etc/bird/bird.conf`。

---

## 4. 实战交互时序图

用户执行 `emu.compile(Docker(), "output")` 后的完整工作流如下：

```mermaid
sequenceDiagram
    participant User Script
    participant Emulator
    participant Layer (Routing/OSPF)
    participant Compiler (Docker)
    participant File System
    participant Docker Daemon

    User Script->>Emulator: render()

    loop Every Layer
        Emulator->>Layer: configure()
        Layer->>Layer: 注册节点，分配 IP
        Emulator->>Layer: render()
        Layer->>Node: addProtocol("ospf", ...)
        Note right of Node: bird.conf 在内存中被修改
        Layer->>Node: appendStartCommand("bird -d")
    end

    User Script->>Emulator: compile(docker_compiler, "./output")
    Emulator->>Compiler: compile()

    Compiler->>Compiler: _doCompile()

    loop Every Node
        Compiler->>File System: mkdir output/node_X
        Compiler->>Node: getFiles() (获取 bird.conf 等)
        Compiler->>File System: Write bird.conf, start.sh
        Compiler->>File System: Write Dockerfile (COPY bird.conf ...)
        Compiler->>File System: Append to docker-compose.yml
    end

    User Script->>Docker Daemon: docker-compose up -d

    Docker Daemon->>Docker Daemon: Build Images (Copy configs)
    Docker Daemon->>Container: Start Container
    Container->>Container: Exec /start.sh
    Container->>Container: Exec /interface_setup (Rename NICs, TC)
    Container->>Container: Exec bird -d (Read /etc/bird/bird.conf)
```
