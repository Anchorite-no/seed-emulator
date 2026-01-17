# Project Guidelines for AI Agents

## Language Requirement
- **Primary Language**: Simplified Chinese (简体中文).
- All documentation, comments, pull request descriptions, and commit messages MUST be written in Simplified Chinese.
- Code variable names and standard terminology (e.g., Docker, BGP) should remain in English.

## 项目概览 (Project Overview)

*   **核心逻辑**：仿真器的核心逻辑位于 `seedemu/` Python 包中。
*   **编译**：系统将 Python 脚本定义 *编译* 成一组 Docker 配置文件（Dockerfile, docker-compose.yml）。
*   **可视化**：Web UI (`client/`) 用于可视化运行中的仿真。

## 关键目录 (Key Directories)

*   `seedemu/`：**修改此处** 以改变仿真器的核心行为。
    *   `seedemu/core/`：基类 (`Node`, `Network`, `Emulator`)。
    *   `seedemu/compiler/Docker.py`：生成 Dockerfiles 和 docker-compose.yml 的逻辑。**模板位于此处。**
    *   `seedemu/layers/`：网络层 (BGP, OSPF, WebService)。
*   `examples/`：参考脚本。使用 `examples/basic/A00_simple_as/simple_as.py` 作为冒烟测试。
*   `client/`：Web UI。
    *   `backend/`：控制 Docker 的 Node.js 守护进程。
    *   `frontend/`：Web 前端。
*   `docker_images/`：基础 Docker 镜像。

## 开发指南 (Development Guidelines)

1.  **不要编辑生成产物 (Artifacts)**：
    *   当用户运行仿真（如 `simple_as.py`）时，会生成一个 `output/` 目录（或类似目录）。
    *   **切勿直接编辑 output 目录中的文件**（例如 `output/docker-compose.yml`, `output/as100host/Dockerfile`）来修复 bug。
    *   相反，应编辑生成这些文件的 **Compiler** (`seedemu/compiler/Docker.py`) 或 **Layers** (`seedemu/layers/`)。

2.  **架构参考**：
    *   请参阅根目录下的 `ARCHITECTURE_DEEP_DIVE.md`，了解详细的系统架构、执行流程和图表。

3.  **测试更改**：
    *   修改核心 `seedemu` 代码后，必须运行仿真脚本以验证生成的输出是否正确。
    *   示例：
        ```bash
        # 1. 修改 seedemu/compiler/Docker.py
        # 2. 运行示例
        python3 examples/basic/A00_simple_as/simple_as.py
        # 3. 检查 ./output 目录，确认生成的文件符合预期
        ```

4.  **客户端开发**：
    *   如果正在开发 `client/`，请记住它是一个独立的 Node.js 应用程序。
    *   后端逻辑位于 `client/backend/src`。
    *   它通过 `docker exec` 与仿真节点通信。

## 常见任务 (Common Tasks)

*   **添加新服务**：在 `seedemu/services/` 中实现一个新类（继承自 `Service`），并可能在 `seedemu/compiler/Docker.py` 中添加相应的 Dockerfile 模板。
*   **更改网络配置**：检查 `seedemu/layers/Base.py` 或 `seedemu/core/Network.py`。
*   **修改生成的 Dockerfiles**：查看 `seedemu/compiler/Docker.py` 中的 `DockerCompilerFileTemplates`。
