# 威胁模型：19379645344/JMComic-Crawler-Python 与 19379645344/desktop-tutorial

状态：已纳入用户对使用方式、关键资产和范围外条目的回答（2026-11-16）。标注“用户确认”的条目来自用户本人，是调整发现优先级的依据；其余内容根据代码推断。

## 1. 系统概述与信任边界

- 使用方式（用户确认）：这是用户的个人 fork，主要通过 GitHub Actions 下载本子，仓库 secrets 中存有禁漫账号和邮箱密码。用户不以上游维护者身份向他人分发 PyPI 包。
- `JMComic-Crawler-Python` 是 hect0x7/JMComic-Crawler-Python 的公开 fork，提供访问禁漫天堂（网页端与移动端 API）的 Python 库 `jmcomic`、命令行工具（`jmcomic = jmcomic.cl:main`），以及在 GitHub Actions 中下载本子、导出收藏夹的工作流。
- `desktop-tutorial` 只包含 GitHub Desktop 教程的 README，没有可执行代码，不构成攻击面。
- 两个仓库都没有基础设施即代码（Terraform、Kubernetes 等）；唯一的部署面是 `.github/workflows/` 中的 GitHub Actions。

信任边界：
1. 禁漫服务器（多个可切换域名）与本地进程之间：HTML 页面、API 响应（AES 加密，`jm_client_impl.py` 中解密）以及图片数据都来自远端，本子标题、作者、章节名等字段会被用于拼接下载路径（`jm_option.py` 的 `DirRule`）。
2. option 配置文件（YAML）与进程之间：配置决定客户端实现、代理、下载目录规则和插件；插件参数支持 `${ENV}` 环境变量替换（`jm_toolkit.py` 的 `JmcomicText.match_os_env`）。
3. GitHub Actions 触发者与 Runner 之间：`workflow_dispatch` 输入（本子 ID、`DIR_RULE`、`CLIENT_IMPL`、`ZIP_NAME` 等）进入环境变量，由 `usage/workflow_download.py` 读取；仓库 secrets 注入同一 Runner。
4. 可选的本地 HTTP 服务（`JmServerPlugin`，依赖外部库 `plugin_jm_server`，默认监听 `0.0.0.0:80`）与局域网或公网访问者之间。

## 2. 组件

| 组件 | 覆盖代码 |
|---|---|
| Client（客户端与网络层） | `src/jmcomic/jm_client_impl.py`、`src/jmcomic/jm_client_interface.py`、`src/jmcomic/jm_config.py` |
| Downloader（下载与落盘） | `src/jmcomic/jm_downloader.py`、`src/jmcomic/jm_entity.py`、`src/jmcomic/jm_toolkit.py`、`src/jmcomic/jm_option.py` 中的 `DirRule` 与路径决策 |
| Option（配置与命令行） | `src/jmcomic/jm_option.py`（配置加载、插件调度）、`src/jmcomic/api.py`、`src/jmcomic/cl.py` |
| Plugins（插件） | `src/jmcomic/jm_plugin.py` |
| Workflows（GitHub Actions 与发布） | `.github/workflows/`、`.github/release.py`、`usage/`、`assets/option/` |
| Tutorial（教程仓库） | `19379645344/desktop-tutorial` 全部内容 |

## 3. 入口点及可达者

- Python API 与命令行：本机用户调用；参数为本子 ID 与 option 文件路径。
- option YAML：由用户本人编写（用户确认），属于可信输入。
- 禁漫服务器响应：服务器运营者、域名劫持者或中间人可控制（客户端使用 `curl_cffi`，模拟 Chrome 指纹）。
- `workflow_dispatch` 输入：只有对仓库有写权限的人能触发。
- `push` 触发的 `download.yml` 与 `release_auto.yml`：有写权限的人。
- `close_specific_pr.yml` 使用 `pull_request_target`，任何 GitHub 用户提交 PR 即可触发；该工作流检出的是基础分支代码，PR 标题只经 `jq` 读取后回显。
- `JmServerPlugin` 启动的 HTTP 服务：能访问该端口的网络访问者，受 `password` 参数保护。

## 4. 资产与敏感度

- 仓库 secrets（用户确认为最重要的资产）：`JM_USERNAME`、`JM_PASSWORD`、`EMAIL_FROM`、`EMAIL_TO`、`EMAIL_PASS`、`ZIP_PASSWORD`、`PYPI_JMCOMIC`、`GITHUB_TOKEN`（`release_auto.yml` 中有 `contents: write` 与 `id-token: write`）。高。
- 运行下载器的本机或 Actions Runner：执行任意代码即可读取上述 secrets 或本地文件。高。
- 浏览器 cookies：`AutoSetBrowserCookiesPlugin` 读取本地浏览器 cookies（仅保留 `yuo1`、`remember_id`、`remember`）。中。
- 下载内容与收藏夹导出：上传为 Actions artifact，公开仓库的 artifact 对任何登录用户可下载。中（推断）。

## 5. 攻击者画像

1. 恶意或被劫持的禁漫服务器 / 中间人：返回构造的标题、章节名、图片或 API 数据，目标是路径穿越写文件、解析或解码崩溃、欺骗客户端。
2. 公开仓库的外部贡献者：通过 PR 触发 `pull_request_target`，目标是窃取 secrets 或 `GITHUB_TOKEN`。
3. 能访问 `JmServerPlugin` 端口的网络访问者：目标是绕过密码浏览或读取 `base_dir` 之外的文件。
4. 恶意 option 配置文件作者：范围外（用户确认）。插件可执行 shell 命令（`execute_cmd` 使用 `os.system`，`execute_multi_line_cmd` 使用 `shell=True`），`DirRule` 使用 `str.format`，这些按配置执行的行为属于预期。
5. 上游供应链：工作流在非 `dev` 分支从 PyPI 安装最新 `jmcomic`，依赖 `commonX`、`plugin_jm_server` 等由上游维护的包。

## 6. 现有控制

- 文件名与目录名经过 `fix_windir_name`、`fix_filepath` 处理（`jm_option.py` 的 `DirRule.apply_rule_to_path`）。
- `JmServerPlugin` 支持密码，非 debug 模式在独立线程运行。
- 工作流输入中的凭据项已注释掉，凭据通过 secrets 注入；`close_specific_pr.yml` 不检出 PR 代码。
- 写权限是触发 `workflow_dispatch` 与 `push` 工作流的前提。

## 7. 范围外与可接受的风险

用户确认：
- option 配置文件由用户本人编写，属于可信输入；插件按配置执行命令、读取环境变量、按 `DirRule` 格式化路径都属于预期行为。只有在攻击者无需控制配置文件（例如经由禁漫服务器响应、PR 或网络请求）就能触发时，相关发现才保持原有严重性。

根据代码推断（不作为调整依据）：
- `desktop-tutorial` 无代码，预计不会产生发现。
- `workflow_dispatch` 输入只有仓库写权限者能提供，等同于用户本人。

## 8. 待确认问题

- 是否使用 `JmServerPlugin`，以及服务是否暴露在局域网之外。
- 下载结果 artifact 在公开仓库中可被任何登录用户下载，用户是否在意这些内容的隐私。
