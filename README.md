# Fermat Toolkit 使用指南

本指南面向已经下载 Fermat Toolkit Release 完整工具包的用户。使用 Toolkit 不需要获取或构建 Fermat 的源码。

## 下载

从本仓库的 [Releases](../../releases) 页面下载对应版本：

```text
fermat-toolkit-VERSION-linux-x86_64.tar.gz
```

## 快速开始

### 1. 首次安装

每台机器只需要安装一次。将 `VERSION` 替换为下载的工具包版本：

```bash
tar xzf fermat-toolkit-VERSION-linux-x86_64.tar.gz
cd fermat-toolkit
source ./setup.sh

fa-agent-dispatcher doctor
```

Setup 会询问是否配置 LLM。正式扫描必须选择 `y`，填写 API URL、API key 和模型 ID，并通过下面的连接检查。没有完成 LLM 配置时不要开始扫描。

`doctor` 用于确认工具包能否正常运行。它默认只检查 LLM 配置文件，不会访问 LLM 服务。配置完成后，再执行一次真实的兼容性检查：

```bash
fa-agent-dispatcher doctor --probe-llm
```

这条命令会向所选模型发送一个很小的测试请求。缺少系统依赖或 LLM 配置有误时，按照终端提示处理。配置方法见[配置 LLM](#配置-llm)。

### 2. 分析项目

执行下面的命令：

```bash
fa-agent-dispatcher --output /path/to/reports /path/to/repository
```

只需要替换两个路径：

- `/path/to/reports`：保存分析报告和日志的目录；
- `/path/to/repository`：需要分析的项目目录。

同一个报告目录可以用于多个项目，也可以在以后继续使用。Dispatcher 会按项目分别保存结果，不会相互覆盖。

命令选项必须写在项目目录之前。

### 3. 查看报告

分析结束后，终端会打印本次报告的完整路径。最终结果主要保存在：

| 文件 | 内容 |
|---|---|
| `fa_report.json` | 最终问题列表和详细信息 |
| `fa_report.sarif` | 可导入支持 SARIF 的平台 |
| `pipeline_result.json` | 本次分析是否完整完成 |
| `producer_health_manifest.json` | 各个分析工具是否正常运行 |
| `dispatcher_diagnostics.json` | 构建或工具运行失败的原因和处理建议 |

只需要查看最终问题时，先打开 `fa_report.json`。`PARTIAL` 表示只完成了部分分析，`UNVERIFIABLE` 表示没有得到可验证结果；两者都不能理解为“没有发现问题”。“分析完成”只表示已选分析链执行完成，不保证发现全部问题。

## 分析前说明

Toolkit 包含 Fermat 分析工具和内部 LLVM 运行时，但不包含目标项目自己的编译器、SDK、sysroot、系统头文件、第三方依赖和生成文件。

分析 C/C++ 或 OpenHarmony 项目前，请先使用运行 Dispatcher 的同一用户和环境完成项目官方构建。Dispatcher 不会安装项目依赖、下载 SDK、猜测缺失头文件或伪造编译参数。项目官方构建失败时，依赖构建结果的分析可能无法完成，其他分析仍会继续。

未提供额外参数时，MESA 先检查项目中的 CMake、GN、Make、Meson、`build.sh` 和官方构建文档。确定性信息足够时直接构建；target、product 或参数存在歧义时，LLM 读取认证后的 README、构建脚本和配置后选择一个有依据的方案。项目完全没有可读构建证据时，才要求用户提供 `--build-guide`。

如果项目需要特定构建方法，可以把已有的构建命令、README 摘录或使用说明作为可选的 `--build-guide` 传入。文件扩展名和内容格式不受限制；内容应当是 LLM 可以读取的构建说明。不提供时，Dispatcher 继续自动识别常见构建方式。

例如扫描目标在子目录，官方构建脚本在外层 workspace：

```text
/workspace/build.sh --target A/B/C/project
/workspace/A/B/C/project        # 扫描目录
```

可以将上述命令及环境准备方法写入任意文件，然后运行：

```bash
fa-agent-dispatcher --output /path/to/reports \
  --build-guide /path/to/project-build-instructions \
  /workspace/A/B/C/project
```

LLM 会优先使用该说明确定 workspace、工作目录和构建步骤。使用 `--build-guide` 前必须完成 LLM 配置并通过 `doctor --probe-llm`。MESA 仍以实际构建退出码和本轮捕获的真实编译命令判定 Build 是否成功。

如果已经有可重放的 `compile_commands.json`，也可以跳过项目构建：

```bash
fa-agent-dispatcher --output /path/to/reports \
  --compile-db /workspace/out/compile_commands.json \
  /workspace/A/B/C/project
```

编译数据库中的命令必须能在当前机器重放，源码必须位于扫描目录内。外部编译数据库可以支持 IR 和 LLVM 分析，但不代表 Dispatcher 本轮执行了项目官方构建。

分析会执行仓库中的构建脚本，并在项目目录中保留构建产物。分析不可信代码时，请使用隔离的虚拟机或容器。

## 报告目录和项目构建目录

`--output` 只保存报告、日志和少量运行信息，不会在该目录复制源码。

项目构建直接在用户传入的物理代码目录或完整 OpenHarmony workspace 中运行。CMake、Meson、Cargo 和 OpenHarmony 生成的构建产物会留在项目自己的构建目录中，后续分析可以复用。请确保代码目录可写，并在需要时按项目的官方方式清理构建产物。

报告目录可以位于项目目录内部或外部。以终端打印的本次报告完整路径为准。

## 分析 OpenHarmony 组件

请传入完整 OpenHarmony 代码目录中的组件路径：

```bash
fa-agent-dispatcher --output /data/fermat-reports /path/to/openharmony/base/security/crypto_framework
```

完整 OpenHarmony workspace 必须包含 `.repo`、`.gn`、`build.sh`、`build/` 和 `prebuilts/`。组件必须有有效的 `bundle.json`、`BUILD.gn`、产品配置和 production GN label。Dispatcher 会从组件路径向上识别 workspace。

`bundle.json` 中有多个 production label 时会全部纳入分析，不需要手工选择。产品能由现有 GN 覆盖唯一确定时直接使用；存在多个候选时，LLM 会读取认证后的 product config、组件 README 和候选证据，并且只选择一个产品执行。`--ohos-product` 和 `--build-target` 仍可用于用户明确知道产品或 target 时的高级覆盖。`--official-workspace` 只能指向完整 OpenHarmony workspace，不能用于普通外层构建目录。

单独下载的组件仓库，或者缺少完整产品配置、SDK 和 prebuilts 时，与构建有关的分析无法得到可信结果。

## 配置 LLM

正式扫描需要先配置兼容的 LLM 服务，并从服务商处取得下面三项信息：

| 信息 | 应该填写什么 |
|---|---|
| API URL | 模型服务的 API 基础地址，不是聊天网页地址。OpenAI-compatible 服务通常以 `/v1` 结尾；远程服务应使用 HTTPS |
| API key | 服务商控制台创建的访问密钥 |
| 模型 ID | 服务商 API 文档中的模型标识，必须与账户实际可用的名称完全一致 |

还需要知道服务采用哪种接口协议：

- 大多数 OpenAI-compatible 服务使用 `openai`，请求路径为 `/chat/completions`；
- 原生 Anthropic-compatible 服务使用 `anthropic`，请求路径为 `/v1/messages`。

模型名称中包含 Claude，不代表接口一定是 `anthropic`。如果服务商说明它提供 OpenAI-compatible 接口，仍然应选择 `openai`。

### 使用 Setup 配置

在解压后的 Toolkit 目录中重新运行 Setup：

```bash
cd /path/to/fermat-toolkit
source ./setup.sh
```

出现 `是否现在配置 LLM？` 时输入 `y`。Setup 接下来会询问 API URL、API key 和模型 ID。API key 的输入不会显示在终端。

Setup 会根据模型 ID 推断协议：以 `claude` 开头的模型使用 `anthropic`，其他模型使用 `openai`。如果名称以 `claude` 开头的服务实际使用 OpenAI-compatible 接口，请使用下面的服务器配置并显式设置 `FERMAT_SETUP_PROTOCOL=openai`。Setup 不会发送网络请求，也不会替换或补全 API URL。

服务器或自动化环境可以使用非交互配置。下面是 OpenAI-compatible 服务的示例：

```bash
FERMAT_SETUP_LLM_URL='https://llm.example.com/v1' \
FERMAT_SETUP_LLM_KEY_FILE='/secure/path/llm-api-key' \
FERMAT_SETUP_MODEL='your-model-id' \
FERMAT_SETUP_PROTOCOL='openai' \
bash ./setup.sh --yes
```

将示例 URL、密钥文件路径和模型 ID 替换为服务商提供的实际值。`FERMAT_SETUP_PROTOCOL` 只能是 `openai` 或 `anthropic`。如果使用 `bash ./setup.sh`，还需要执行安装结果打印的 `export PATH=...`，让当前 shell 找到启动器。

Setup 默认把配置保存在 `~/.config/fermat/`；设置 `XDG_CONFIG_HOME` 时使用该目录。不要把配置文件提交到 Git，也不要把 API key 粘贴到日志、Issue 或聊天记录中。

### 配置失败时

如果 Toolkit 已经能够运行，但 Setup 的 LLM 配置步骤失败，可以手动创建 Dispatcher 读取的配置文件。先创建配置目录：

```bash
mkdir -p "${XDG_CONFIG_HOME:-$HOME/.config}/fermat"
chmod 700 "${XDG_CONFIG_HOME:-$HOME/.config}/fermat"
```

创建 `${XDG_CONFIG_HOME:-$HOME/.config}/fermat/llm-profile.json`。OpenAI-compatible 服务使用下面的模板：

```json
{
  "schema_version": 4,
  "source": "direct",
  "provider": "my-provider",
  "model": "your-model-id",
  "format": "openai",
  "endpoint_mode": "base_url",
  "endpoint": "https://llm.example.com/v1",
  "credential": "your-api-key",
  "auth_mode": "bearer"
}
```

必须替换 `model`、`endpoint` 和 `credential`。保存后限制文件权限：

```bash
chmod 600 "${XDG_CONFIG_HOME:-$HOME/.config}/fermat/llm-profile.json"
```

如果不希望把 API key 直接写入 JSON，可以把 `credential` 改为密钥文件引用：

```json
"credential": "{file:/secure/path/llm-api-key}"
```

密钥文件只能包含 API key，并且权限必须为 `0600`。

原生 Anthropic-compatible 服务需要把模板中的三个字段改为：

```json
"format": "anthropic",
"endpoint": "https://llm.example.com/v1",
"auth_mode": "api_key"
```

如果服务商给出的是包含 `/chat/completions` 或 `/v1/messages` 的完整请求地址，把 `endpoint_mode` 改为 `exact_endpoint`。

配置完成后运行：

```bash
fa-agent-dispatcher doctor
fa-agent-dispatcher doctor --probe-llm
```

`doctor` 显示“已配置”只表示配置文件可以读取。`doctor --probe-llm` 通过后，才能确认网络、鉴权、模型 ID 和接口格式可用；该检查不验证模型的分析质量。

Setup 不修改 shell profile。如果使用 `bash ./setup.sh`，请执行安装结果打印的 `export PATH=...`。

## 退出码

| 退出码 | 含义 |
|---|---|
| `0` | 分析正常完成，或没有适用输入；后者不表示完成了代码扫描 |
| `2` | `PARTIAL` 或 `UNVERIFIABLE`；报告和诊断仍会保留 |
| `1` | 分析失败 |

发现问题本身不会让程序返回非零。退出码只表示分析流程是否完整执行。

## 问题排查

| 现象 | 处理 |
|---|---|
| 找不到 `fa-agent-dispatcher` | 执行安装结果打印的 `export PATH="$HOME/.local/bin:$PATH"`，或直接使用 `$HOME/.local/bin/fa-agent-dispatcher` |
| `doctor` 显示 LLM 未配置 | 重新运行 `source ./setup.sh` 并选择配置 LLM；服务器环境使用上面的 `FERMAT_SETUP_*` 配置 |
| LLM 检查失败 | 检查 API URL、API key、模型 ID、协议、网络和账户权限，然后重新运行 `doctor --probe-llm` |
| 项目构建失败 | 使用运行 Dispatcher 的同一用户和环境先运行官方构建，再补齐编译器、SDK、依赖、submodule 和生成文件 |
| 构建脚本位于扫描目录上层 | 使用 `--build-guide` 提供官方构建命令和环境准备方法 |
| 没有 finding，但结果不是完整成功 | 查看 `pipeline_result.json` 和诊断；这不表示项目没有问题 |
| 退出码为 `2` | 查看 `pipeline_result.json` 和 `dispatcher_diagnostics.json`，确认哪些分析没有完成 |
