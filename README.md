# kb-query 使用说明

> v1.2.1 · 紫鸾知识平台命令行查询工具 · 单文件 / 零依赖 / 仅 Python 标准库

## 1. 安装

```bash
chmod +x kb-query && ./kb-query --version     # Linux / macOS
python kb-query --version                      # Windows（无 python 命令时用 py kb-query --version）
```

## 2. 配置

```bash
kb-query --init        # 生成 ~/.kb-query.json 模板（权限 600，仅 Linux/macOS；Windows 无该权限位），编辑填入真实值
```

```json
{
  "api_key": "平台颁发的 API KEY",
  "recall_base": "http://<网关IP>:<端口>",
  "chat_base": "http://<网关IP>:<端口>",
  "tenant_id": "租户 id",
  "dept_id": "", "default_kb": "",
  "timeout": 30, "chat_timeout": 120, "open_llm": "0",
  "verify_ssl": "1", "admin_contact": "知识平台管理员"
}
```

> recall_base / chat_base 填**裸网关地址**，工具自动追加 `/rag-platform-default`（带或不带**单个**该前缀、尾部有无斜杠都会被规整；注意仅规整一层，请勿填写重复前缀）。两个网关相同时填同一个值——核心配置就是 **KEY + 网关地址 + 租户 id** 三项。也可用环境变量，优先级：命令行 > 环境变量 > 配置文件。环境变量设为空串视为显式覆盖（如 `KB_QUERY_DEFAULT_KB=""` 清空默认库）。

| 配置项       | 环境变量                                             | 必填 | 说明                                              |
| --------- | ------------------------------------------------ | -- | ----------------------------------------------- |
| API KEY   | `KB_QUERY_API_KEY`                               | ✅  | 只走环境变量/配置文件，永不进命令行                              |
| RECALL 网关 | `KB_QUERY_RECALL_BASE`                           | ✅  | 裸网关地址，自动加 `/rag-platform-default`               |
| CHAT 网关   | `KB_QUERY_CHAT_BASE`                             | ✅  | 同上；两个网关一般相同                                     |
| tenant_id | `KB_QUERY_TENANT_ID`                             | ✅  | 租户 id                                           |
| dept_id   | `KB_QUERY_DEPT_ID`                               | 否  | 标准版必传；简版留空                                      |
| 默认知识库     | `KB_QUERY_DEFAULT_KB`                            | 否  | 逗号分隔多个；不设且无 `--kb` 时查全库                         |
| 超时(秒)     | `KB_QUERY_TIMEOUT` / `KB_QUERY_CHAT_TIMEOUT`     | 否  | 检索默认 30，问答默认 120；CLI `--timeout` 会**同时覆盖**两者    |
| 模型兜底      | `KB_QUERY_OPEN_LLM`                              | 否  | 透传给平台 `params.open_llm`（`"0"`/`"1"`），平台侧含义见网关文档 |
| SSL / 联系人 | `KB_QUERY_VERIFY_SSL` / `KB_QUERY_ADMIN_CONTACT` | 否  | 自签证书设 `"0"` 跳过校验；401 报错显示的联系人                   |

## 3. 命令参考

```bash
kb-query "<问题>"                              # 检索模式（默认）：返回 top-N 文档块
kb-query "<问题>" --chat                       # 问答模式：AI 流式生成答案 + 引用来源
kb-query --list                                # 列出知识库
kb-query "<问题>" --kb KLB_aaa --kb KLB_bbb    # 指定知识库（可多个）
kb-query "<问题>" --top-n 3 --max-chars 800    # 返回条数 / 正文截断长度（默认 5 / 500，0=不截断）
kb-query "<问题>" --json                       # JSON 输出（agent 程序化消费）
kb-query "<问题>" --insecure --timeout 60      # 跳过 SSL 校验 / 覆盖超时（同时作用于检索与问答）
kb-query --init / --version / --help
```

**检索模式**返回知识库中的原始文档片段（非 AI 生成）：

```
[1] pg054-7series-pcie.pdf · 得分 0.94 · 章节: 7 Series FPGAs › Ch. 3 › Protocol Layers
    <chunk_text，默认截断 500 字符>
```

**问答模式**使用 SSE 流式接口，答案逐字实时输出，`<think>` 思维链自动过滤；`--json` 攒完输出，含 `"hit": true/false`：

```
PCIe 与 PCI 总线的主要区别在于架构设计……（逐字出现）

引用来源：
[1] pcie-primer.pdf · 得分 0.44 · 知识库: pcie测试 · 目录: /pcie测试
```

> **参数冲突提示：** `--list` 与 `--chat` 同时使用时 `--chat` 被忽略；`--list` 模式下 `--kb` 无意义；`--chat` 模式下 `--top-n` 无效。三者均会打印警告到 stderr 但不影响执行（`--list` 正常输出）。

## 4. Agent 集成（复制即用）

在 `CLAUDE.md` / `AGENTS.md` 中粘贴：

```markdown
## 知识库查询
用 `kb-query "<问题>"` 检索知识库，输出带编号引用的文档块。
用 `kb-query "<问题>" --chat` 获取生成式答案和引用来源（`--json` 含 `"hit"` 判断是否命中）。
退出码：0 成功 / 1 缺配置 / 2 网络不通 / 3 KEY 失效。非 0 时把 stderr 报告给用户。
```

## 5. 作为 Claude Code Skill 使用

kb-query 可封装为 **Claude Code 的 Agent Skill**（Claude Code 2.x 原生支持），让 Claude 在对话中**自动识别知识库问题并调用**，无需手动粘贴提示词。

### 5.1 安装 Skill（约 1 分钟）

```bash
# 1. 确认 Claude Code 版本 ≥ 2.x（2.x 才支持 Agent Skills）
claude --version

# 2. 创建 skill 目录（用户级，所有项目生效；也可用 .claude/skills/ 做项目级）
mkdir -p ~/.claude/skills/kb-query
```

1. 将下面内容保存为 `~/.claude/skills/kb-query/SKILL.md`（**工具路径按实际部署机器调整**，本机 10.0.0.2 已部署在 `~/kb-query/`），重启 Claude Code 即自动生效：

````markdown
---
name: kb-query
description: 查询"紫鸾知识平台"内部知识库。当用户询问需要内部文档支撑的技术问题(如 PCIe、芯片设计、EDA 工具命令、UPF/复位值),或明确要求"查知识库/查文档/检索资料"时使用。关键词:知识库、文档检索、PCIe、TLP、LTSSM、add_pst_state、RAG、芯片设计、复位值、UPF。
---

# kb-query 知识库查询

调用命令行工具 kb-query 检索紫鸾知识平台(本机 10.0.0.2 部署,配置见 ~/.kb-query.json)。

## 调用方式

工具绝对路径:`~/kb-query/kb-query`(若 PATH 已含 kb-query 也可直接用)。**优先使用绝对路径**,避免 PATH 差异。

### 检索模式(默认,返回文档片段,非 AI 生成)
```bash
~/kb-query/kb-query "<问题>" --json
~/kb-query/kb-query "<问题>" --top-n 3 --json
~/kb-query/kb-query "<问题>" --kb KLB_xxx --json
```

### 问答模式(AI 生成答案 + 引用来源)
```bash
~/kb-query/kb-query "<问题>" --chat --json
```

### 其他
```bash
~/kb-query/kb-query --list --json
```

## 输出解读

- `--json` 输出为合法 JSON,解析 STDOUT;STDERR 是诊断信息,不要混入答案。
- 检索模式:`results[]` 按得分降序,字段含 score/doc_title/catalog/chunk_text;`faq_answer` 非空 = FAQ 命中。
- 问答模式:看 `"hit"` 字段——`true` 才可作为答案引用并列出 `sources[]`;`false` = 知识库未命中,如实告知用户"知识库未检索到",可改试检索模式。
- 退出码:0=成功(含无结果);1=缺配置/参数错(转述 stderr);2=网络/超时(可重试一次);3=KEY 失效(提醒用户联系管理员重新签发)。
- chat 模式较慢(10-40s),属正常,耐心等待。

## 示例

用户问"PCIe 事务层的作用":
1. 先执行 `~/kb-query/kb-query "PCIe 事务层的作用" --chat --json`
2. hit=true → 用 answer 回答,并列出 sources 引用来源;
3. hit=false → 改用 `~/kb-query/kb-query "PCIe 事务层的作用" --json` 返回文档片段,引用给用户。
````

### 5.2 使用方法

- **自动触发（主要方式）**：正常提问即可——如 *"PCIe 事务层有什么作用"*、*"查一下知识库里 LTSSM 的资料"*，Claude 命中 skill 的 description 后自动加载并调用 kb-query；
- **主动触发（兜底）**：明确说 *"用 kb-query 查一下 ..."*；
- **验证是否生效**：
  ```bash
  claude -p "用 kb-query 检索:PCIe 链路训练过程"   # 非交互快速验证
  claude                                        # 交互会话,输入 /skills 查看已加载技能
  ```
- **行为预期**：检索模式返回带得分的文档片段（chunk 级内容检索）；问答模式返回生成答案 + `hit` 字段 + 引用来源；未命中时 Claude 会如实说明"知识库未检索到"，不会编造引用。

## 6. 退出码与排错

| 码 | 含义               | 处置                                         |
| - | ---------------- | ------------------------------------------ |
| 0 | 成功（含无结果/FAQ 命中）  | 正常                                         |
| 1 | 缺配置 / 参数错误       | 按 stderr 提示补齐                              |
| 2 | 网络不通 / 超时 / 流式中断 | 确认网络；网关填裸 IP；问答慢调大 `KB_QUERY_CHAT_TIMEOUT` |
| 3 | KEY 无效或过期        | 联系管理员重新签发                                  |

**常见问题：** ① 缺配置 → 逐项补齐，环境变量需 `export`；② 无法连接 → `ping` 网关，地址填裸网关不含前缀；③ 401 → KEY 过期；④ SSL 错 → `--insecure`；⑤ `--chat` 回兜底话术 → 知识库未命中；⑥ 中文乱码 → Windows 终端 `chcp 65001`。

不做 MCP server / 上传写操作 / UI / 会话记忆 / FAQ 库管理。
