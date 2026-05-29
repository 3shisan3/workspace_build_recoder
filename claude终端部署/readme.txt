安装 Claude Code
打开 PowerShell 或 CMD 终端，运行以下安装命令即可。

Windows PowerShell 安装命令：
irm https://claude.ai/install.ps1 | iex
Windows CMD 安装命令：
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
如果以上都不行的话，就用这个命令：
winget install Anthropic.ClaudeCode

macOS, Linux, WSL 安装指令：
curl -fsSL https://claude.ai/install.sh | bash

Homebrew，macOS / Linux 安装命令：
brew install --cask claude-code

官方安装脚本确认 —— claude install命令不接受任何安装路径参数
如果官方安装链接不可用，可使用本地脚本替代：
    Windows CMD：运行 win_install.cmd [stable|latest|VERSION]
    macOS / Linux：运行 ./linux_install.sh [stable|latest|VERSION]

windows下Claude 可执行文件所在的目录需要加入系统 PATH，否则 PowerShell 目前无法识别 claude 命令。
参考路径：
C:\Users\你的用户名\.local\bin

密钥获取，以deepseek为例，且deepseek官方有较完整的接入方案
https://platform.deepseek.com/api_keys
https://api-docs.deepseek.com/zh-cn/

claude配置文件路径：
Windows：C:\Users\用户名文件夹\.claude\settings.json
Linux：~/.claude/settings.json
使用deepseek时claude配置文件内容示例：
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "",
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_EFFORT_LEVEL": "max"
  },
  "theme": "auto"
}
