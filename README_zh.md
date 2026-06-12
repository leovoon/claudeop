# claudeop

用**免费的 OpenRouter 模型**启动 Claude Code CLI —— 不需要 Anthropic 订阅。

一个极简的命令行工具：
- 每次启动自动从 OpenRouter 获取最新的免费模型
- 用模糊搜索器（fzf）让你选一个模型
- 直接进入 Claude Code
- **完全不影响**你原来的 `claude` 命令

```
claudeop        → 选一个免费模型，启动 Claude Code
claude          → 照常用你的 Anthropic 订阅（完全不变）
```

---

## 这是干什么的

把它想象成同一个电视的两把遥控器：

| 命令 | 做什么 | 谁付费 |
|------|--------|--------|
| `claude` | 用你的 Anthropic 账户启动 Claude Code | Anthropic 订阅 |
| `claudeop` | 用 OpenRouter 的免费模型启动 Claude Code | 免费 |

两个命令共享同一个 Claude Code 设置、插件、MCP 服务器和项目配置。唯一的区别是**回答你问题的是哪个模型**。

---

## 前置条件

开始之前，你的电脑上需要装好三样东西。别担心 —— 全都是免费的。

### 1. Claude Code CLI

Claude Code 是 Anthropic 的编程助手，在终端里运行。

**安装：**
```bash
# macOS — 在终端粘贴这行并按回车
curl -fsSL https://storage.googleapis.com/claude-code/claude-code-installer.sh | sh
```

**验证安装成功：**
```bash
claude --version
# 应该显示类似：2.1.175
```

> 如果你已经装了 Claude Code，跳过这步。

---

### 2. OpenRouter 账户和 API Key

OpenRouter 是一个让你免费使用各种 AI 模型（包括 Claude）的服务。

1. 去 [openrouter.ai](https://openrouter.ai) **注册免费账户**
2. 登录后，去 [Settings → API Keys](https://openrouter.ai/settings/keys)
3. 点击 **"Create Key"**
4. 复制这个 key（以 `sk-or-v1-...` 开头）
5. 先存好 —— 第 3 步要用

---

### 3. Homebrew（macOS 软件包管理器）

Homebrew 帮你安装其他工具。你可能已经有了。

**检查是否已安装：**
```bash
brew --version
# 如果显示版本号，说明已经有了 —— 跳过安装
```

**安装（如果需要）：**
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

---

## 安装

### 第 1 步：安装 `jq` 和 `fzf`

claudeop 需要这两个小工具。一行命令搞定：

```bash
brew install jq fzf
```

### 第 2 步：把 OpenRouter API Key 添加到你的 Shell

打开你的 Shell 配置文件：

```bash
nano ~/.zshrc
```

滚动到**最底部**，添加这行（把 `sk-or-v1-YOUR_KEY_HERE` 换成你真实的 key）：

```bash
export OPENROUTER_API_KEY="sk-or-v1-YOUR_KEY_HERE"
```

保存并退出：
- 按 `Ctrl + O`，然后按 `Enter` 保存
- 按 `Ctrl + X` 退出

然后重新加载配置：

```bash
source ~/.zshrc
```

**验证：**
```bash
echo $OPENROUTER_API_KEY
# 应该显示你的 key，以 sk-or-v1- 开头
```

### 第 3 步：添加 `claudeop` 命令

再次打开配置文件：

```bash
nano ~/.zshrc
```

滚动到**最底部**，粘贴以下完整代码：

```bash
# ── Claudeop：用免费 OpenRouter 模型启动 Claude Code ──────────
#   claudeop            → 选择免费模型，启动 Claude Code
#   claudeop -m <id>   → 跳过选择器，直接用指定模型
#   claude              → Anthropic 订阅（不变）

# 每次启动时从 OpenRouter 获取 :free 模型
_claudeop_models() {
  curl -s "https://openrouter.ai/api/v1/models" \
    -H "Authorization: Bearer $OPENROUTER_API_KEY" \
    -H "Content-Type: application/json" \
    | jq -r '[.data[] | select(.id | test(":free"))] | sort_by(.id) | .[].id' 2>/dev/null
}

claudeop() {
  if [[ -z "$OPENROUTER_API_KEY" ]]; then
    echo "❌ OPENROUTER_API_KEY not set"
    return 1
  fi
  local model=""
  local skip_picker=0
  local args=("$@")
  while [[ $# -gt 0 ]]; do
    case "$1" in
      -m) model="$2"; skip_picker=1; shift 2 ;;
      --model) model="$2"; skip_picker=1; shift 2 ;;
      *) shift ;;
    esac
  done
  if [[ $skip_picker -eq 0 ]]; then
    local models="$( _claudeop_models )"
    if [[ -z "$models" ]]; then
      echo "❌ No models found on OpenRouter"; return 1
    fi
    model=$(echo "$models" | fzf --prompt="OpenRouter model → " --height=~40% --reverse --no-info)
    if [[ -z "$model" ]]; then
      echo "Cancelled"; return 0
    fi
  fi
  echo "🟢 OpenRouter → [$model]"
  ANTHROPIC_API_KEY="" \
  ANTHROPIC_BASE_URL="https://openrouter.ai/api" \
  ANTHROPIC_AUTH_TOKEN="$OPENROUTER_API_KEY" \
  CLAUDE_CODE_MODEL_OVERRIDE="$model" \
  claude "${args[@]}"
}
```

保存并退出（`Ctrl + O`、`Enter`、`Ctrl + X`），然后重新加载：

```bash
source ~/.zshrc
```

### 第 4 步：测试

```bash
claudeop
```

你应该会看到一个**模型选择器**，列出所有免费模型。用方向键选择，按 `Enter` 确认，Claude Code 就会用那个模型启动。

留意启动时显示的横幅：
```
🟢 OpenRouter → [nex-agi/nex-n2-pro:free]
```

成功！现在可以输入你的问题了。

---

## 使用方法

### 常规用法（每次选模型）
```bash
claudeop
```
出现免费模型列表。方向键浏览，直接打字过滤，`Enter` 选择。

### 指定模型（跳过选择器）
```bash
claudeop -m "google/gemma-3-27b-it:free"
```

### 用你原来的 Claude（订阅）
```bash
claude
```
跟以前一模一样，不受影响。

---

## 原理（给好奇的人）

```
claudeop
  │
  ├─ 从 OpenRouter API 获取免费模型列表
  ├─ 用 fzf（模糊搜索器）展示
  ├─ 你选一个
  └─ 启动 `claude`，附带以下环境变量（仅本次会话有效）：
       ANTHROPIC_BASE_URL = https://openrouter.ai/api
       ANTHROPIC_AUTH_TOKEN = 你的 OpenRouter key
       ANTHROPIC_API_KEY = ""（清空，防止冲突）
       CLAUDE_CODE_MODEL_OVERRIDE = 你选的模型
```

你的 Shell 环境**完全不会被修改**。这些环境变量只存在于那次 Claude Code 进程中。进程一结束，它们就消失了。

---

## 常见问题

### "❌ OPENROUTER_API_KEY not set"
你的 API key 没有加载到 Shell 里。回到第 2 步，确认你在 `~/.zshrc` 里添加了 `export` 那行，并且运行了 `source ~/.zshrc`。

### "❌ No models found on OpenRouter"
- 检查网络连接
- 你的 API key 可能无效 —— 去 [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys) 重新创建一个
- 测试你的 key 是否有效：
  ```bash
  curl -s https://openrouter.ai/api/v1/models \
    -H "Authorization: Bearer $OPENROUTER_API_KEY" \
    | jq '.data | length'
  ```
  如果显示的数字 > 0，说明 key 没问题。

### 选择器是空的 / 没有模型
免费模型在 OpenRouter 上会不定期变动。过一段时间再试 —— 新的免费模型会不断出现。

### Claude Code 显示认证冲突警告
如果你之前用 Anthropic 账户登录过 Claude Code：
1. 在 Claude Code 里运行 `/logout`
2. 退出 Claude Code
3. 重新运行 `claudeop`

### 模型不能用 / 使用时报错
免费模型有功能限制。有些不支持工具调用、长上下文或 Claude Code 的高级功能。试试选择器里的其他免费模型。

---

## 卸载

从 `~/.zshrc` 里删掉 claudeop 的代码块：

```bash
nano ~/.zshrc
# 删除以下内容之间的所有代码（包括注释和函数）：
# "# ── Claudeop：用免费 OpenRouter 模型启动 Claude Code ──────────"
# 到最后的 "}"
```

如果不需要，也可以删掉 API key 那行：

```bash
# 删除这行（如果你其他地方也不用 OpenRouter）：
export OPENROUTER_API_KEY="sk-or-v1-..."
```

重新加载：
```bash
source ~/.zshrc
```

你正常的 `claude` 命令不受任何影响。

---

## 许可证

MIT
