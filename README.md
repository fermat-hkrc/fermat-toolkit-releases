# Fermat Toolkit

Fermat Toolkit 是面向 Linux x86_64 的代码分析工具包。使用 Toolkit 不需要获取或构建 Fermat 的开发源码。

## 下载

从本仓库的 [Releases](../../releases) 页面下载对应版本：

```text
fermat-toolkit-VERSION-linux-x86_64.tar.gz
```

## 安装

将 `VERSION` 替换为下载的版本号：

```bash
tar xzf fermat-toolkit-VERSION-linux-x86_64.tar.gz
cd fermat-toolkit
source ./setup.sh

fa-agent-dispatcher doctor
```

如果解压工具移除了文件的执行权限，可以先运行：

```bash
bash ./setup.sh --verify
```

Setup 不需要管理员权限，也不会修改 shell profile。不要使用 `sudo setup.sh`，否则启动器和配置会写入 root 用户目录。

## 配置 LLM

运行 Setup 时可以按照提示填写：

- API URL；
- API key；
- 模型 ID；
- 接口协议 `openai` 或 `anthropic`。

配置完成后检查实际连接：

```bash
fa-agent-dispatcher doctor --probe-llm
```

API key 的输入不会显示在终端。不要把 API key 写入项目仓库、Issue 或日志。

服务器或自动化环境可以使用非交互配置：

```bash
FERMAT_SETUP_LLM_URL='https://llm.example.com/v1' \
FERMAT_SETUP_LLM_KEY_FILE='/secure/path/llm-api-key' \
FERMAT_SETUP_MODEL='your-model-id' \
FERMAT_SETUP_PROTOCOL='openai' \
bash ./setup.sh --yes
```

请替换示例 URL、密钥文件路径和模型 ID。密钥文件权限应设置为 `0600`。

## 分析项目

基本命令：

```bash
fa-agent-dispatcher --output /path/to/reports /path/to/repository
```

- `/path/to/reports`：保存报告和日志的目录；
- `/path/to/repository`：需要分析的项目目录。

命令选项必须写在项目目录之前。

项目需要特定构建方法时，可以提供构建说明：

```bash
fa-agent-dispatcher --output /path/to/reports \
  --build-guide /path/to/build-instructions \
  /path/to/repository
```

如果已有能够在当前机器重放的 `compile_commands.json`：

```bash
fa-agent-dispatcher --output /path/to/reports \
  --compile-db /workspace/out/compile_commands.json \
  /workspace/path/to/project
```

## 分析 OpenHarmony 组件

传入完整 OpenHarmony 工作区中的组件路径：

```bash
fa-agent-dispatcher --output /data/fermat-reports \
  /path/to/openharmony/base/security/crypto_framework
```

开始分析前，应使用运行 Toolkit 的同一用户和环境完成项目官方构建。完整工作区需要包含项目使用的产品配置、SDK、依赖、生成文件和预编译工具。

## 查看结果

分析结束后，终端会打印本次报告的完整路径。主要文件包括：

| 文件 | 内容 |
|---|---|
| `fa_report.json` | 最终问题列表和详细信息 |
| `fa_report.sarif` | 可导入支持 SARIF 的平台 |
| `pipeline_result.json` | 本次分析是否完整完成 |
| `producer_health_manifest.json` | 各条分析链路是否正常运行 |
| `dispatcher_diagnostics.json` | 失败原因和处理建议 |

`PARTIAL` 表示只完成了部分分析。`UNVERIFIABLE` 表示没有得到可验证结果。这两种状态都不能理解为“没有发现问题”。

## 退出码

| 退出码 | 含义 |
|---|---|
| `0` | 分析正常完成，或没有适用输入 |
| `2` | 分析结果为 `PARTIAL` 或 `UNVERIFIABLE` |
| `1` | 分析失败 |

发现问题本身不会让程序返回非零。退出码表示分析流程是否完整执行。

## 常见问题

### 找不到 `fa-agent-dispatcher`

执行 Setup 输出的 PATH 命令，通常是：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### LLM 连接检查失败

检查 API URL、API key、模型 ID、接口协议、网络和账户权限，然后重新运行：

```bash
fa-agent-dispatcher doctor --probe-llm
```

### 项目构建失败

使用同一用户和环境先运行项目官方构建，并补齐项目要求的编译器、SDK、依赖、submodule 和生成文件。

### 没有问题报告，但结果不是完整成功

查看 `pipeline_result.json` 和 `dispatcher_diagnostics.json`，确认哪些分析没有完成。
