# Neovim Setup Guide

Reproduces this exact Neovim environment: **LazyVim** distribution with a matrix colorscheme, GitHub Copilot + CopilotChat, DAP debugging, and LSP/tooling for TypeScript, Python, Java, and C#/.NET.

---

## Requirements

### Neovim

Install **v0.12+** (this setup uses v0.12.2). The recommended way on Linux is via the [official releases](https://github.com/neovim/neovim/releases):

```bash
# Option A: AppImage (easiest, works on any distro)
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.appimage
chmod +x nvim-linux-x86_64.appimage
sudo mv nvim-linux-x86_64.appimage /usr/local/bin/nvim

# Option B: tarball
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.tar.gz
tar xzf nvim-linux-x86_64.tar.gz
sudo mv nvim-linux-x86_64 /opt/nvim
sudo ln -sf /opt/nvim/bin/nvim /usr/local/bin/nvim
```

### System dependencies

```bash
# Ubuntu/Debian
sudo apt install git gcc g++ make ripgrep fd-find fzf lazygit curl unzip

# Fedora/RHEL
sudo dnf install git gcc g++ make ripgrep fd-find fzf lazygit curl unzip

# Arch
sudo pacman -S git gcc make ripgrep fd fzf lazygit curl unzip
```

> On Debian/Ubuntu `fd` installs as `fdfind`. Symlink it: `sudo ln -sf $(which fdfind) /usr/local/bin/fd`

### Language runtimes

Install the runtimes for the language extras you want to use.

#### Node.js (TypeScript/JS) — via nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
# restart shell, then:
nvm install 24
nvm use 24
nvm alias default 24
```

#### Python — via pyenv

```bash
curl https://pyenv.run | bash
# add to ~/.zshrc or ~/.bashrc:
#   export PYENV_ROOT="$HOME/.pyenv"
#   export PATH="$PYENV_ROOT/bin:$PATH"
#   eval "$(pyenv init -)"
# restart shell, then:
pyenv install 3.14.2
pyenv global 3.14.2
pip install debugpy        # required for Python DAP
```

#### Java — via SDKMAN

```bash
curl -s "https://get.sdkman.io" | bash
# restart shell, then:
sdk install java 25.0.3-tem   # or any LTS (21, 17) — jdtls supports all
```

#### .NET / C# — via Microsoft package feed

```bash
# Ubuntu 22.04+
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt update && sudo apt install dotnet-sdk-10

# Install csharp-ls (LSP server used instead of broken OmniSharp)
dotnet tool install --global csharp-ls
# Make sure ~/.dotnet/tools is on your PATH
```

---

## Neovim configuration

### Back up and wipe any existing config

```bash
mv ~/.config/nvim ~/.config/nvim.bak 2>/dev/null || true
mv ~/.local/share/nvim ~/.local/share/nvim.bak 2>/dev/null || true
mv ~/.local/state/nvim ~/.local/state/nvim.bak 2>/dev/null || true
mv ~/.cache/nvim ~/.cache/nvim.bak 2>/dev/null || true
```

### Create the config skeleton

```bash
mkdir -p ~/.config/nvim/lua/{config,plugins}
```

### `~/.config/nvim/init.lua`

```lua
require("config.lazy")
```

### `~/.config/nvim/lua/config/lazy.lua`

Bootstraps lazy.nvim and declares all plugin specs, language extras, and AI extras.

```lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not (vim.uv or vim.loop).fs_stat(lazypath) then
  local lazyrepo = "https://github.com/folke/lazy.nvim.git"
  local out = vim.fn.system({ "git", "clone", "--filter=blob:none", "--branch=stable", lazyrepo, lazypath })
  if vim.v.shell_error ~= 0 then
    vim.api.nvim_echo({
      { "Failed to clone lazy.nvim:\n", "ErrorMsg" },
      { out, "WarningMsg" },
      { "\nPress any key to exit..." },
    }, true, {})
    vim.fn.getchar()
    os.exit(1)
  end
end
vim.opt.rtp:prepend(lazypath)

require("lazy").setup({
  spec = {
    -- LazyVim base distribution
    { "LazyVim/LazyVim", import = "lazyvim.plugins" },
    -- AI
    { import = "lazyvim.plugins.extras.ai.copilot" },
    { import = "lazyvim.plugins.extras.ai.copilot-chat" },
    -- DAP (debugger)
    { import = "lazyvim.plugins.extras.dap.core" },
    -- Language extras
    { import = "lazyvim.plugins.extras.lang.typescript" },
    { import = "lazyvim.plugins.extras.lang.python" },
    { import = "lazyvim.plugins.extras.lang.java" },
    { import = "lazyvim.plugins.extras.lang.dotnet" },
    -- Custom plugins (files in lua/plugins/)
    { import = "plugins" },
  },
  defaults = {
    lazy = false,
    version = false,
  },
  install = { colorscheme = { "tokyonight", "habamax" } },
  checker = { enabled = true, notify = false },
  performance = {
    rtp = {
      disabled_plugins = { "gzip", "tarPlugin", "tohtml", "tutor", "zipPlugin" },
    },
  },
})
```

### `~/.config/nvim/lua/config/options.lua`

```lua
-- LazyVim defaults are already good.
-- Add any personal overrides here.
```

### `~/.config/nvim/lua/config/keymaps.lua`

```lua
-- LazyVim defaults are already good.
-- Add any personal keymaps here.
```

### `~/.config/nvim/lua/config/autocmds.lua`

```lua
-- LazyVim defaults are already good.
-- Add any personal autocmds here.
```

---

## Custom plugins

Create each file under `~/.config/nvim/lua/plugins/`.

### `colorscheme.lua` — Matrix green-on-black theme

```lua
return {
  {
    "iruzo/matrix-nvim",
    lazy = false,
    priority = 1000,
  },
  {
    "LazyVim/LazyVim",
    opts = {
      colorscheme = "matrix",
    },
  },
}
```

### `copilot-chat.lua` — CopilotChat with agentic tools

```lua
return {
  "CopilotC-Nvim/CopilotChat.nvim",
  opts = {
    tools = "copilot",       -- enable all built-in tools (file, url, buffer, shell, edit, workspace)
    trusted_tools = true,    -- auto-approve tool calls
    model = "gpt-4.1",
  },
}
```

### `dap.lua` — DAP UI + Python + persistent breakpoints

```lua
return {
  -- Keep DAP UI open after session ends
  {
    "rcarriga/nvim-dap-ui",
    opts = { auto_close = false },
    config = function(_, opts)
      local dap = require("dap")
      local dapui = require("dapui")
      dapui.setup(opts)
      dap.listeners.after.event_initialized["dapui_config"] = function()
        dapui.open({})
      end
    end,
  },
  -- Python debugger via debugpy + pytest runner
  {
    "mfussenegger/nvim-dap-python",
    config = function()
      require("dap-python").setup("debugpy-adapter")
      require("dap-python").test_runner = "pytest"
    end,
  },
  -- Persist breakpoints across sessions
  {
    "Weissle/persistent-breakpoints.nvim",
    event = "BufReadPost",
    opts = { load_breakpoints_event = { "BufReadPost" } },
    keys = {
      { "<leader>db", "<cmd>PBToggleBreakpoint<cr>",         desc = "Toggle Breakpoint" },
      { "<leader>dB", "<cmd>PBSetConditionalBreakpoint<cr>", desc = "Conditional Breakpoint" },
      { "<leader>dC", "<cmd>PBClearAllBreakpoints<cr>",      desc = "Clear All Breakpoints" },
    },
  },
}
```

### `lang-csharp.lua` — Replace broken OmniSharp with csharp-ls

```lua
-- OmniSharp has been broken since mid-2025; csharp-ls is the replacement.
-- Install it first: dotnet tool install --global csharp-ls
return {
  {
    "neovim/nvim-lspconfig",
    opts = {
      servers = {
        omnisharp  = { enabled = false },
        csharp_ls  = {},
      },
    },
  },
}
```

---

## `.neoconf.json` (optional, per-project or global)

Place in `~/.config/nvim/.neoconf.json` for global Lua LS settings:

```json
{
  "neodev": {
    "library": { "enabled": true, "plugins": true }
  },
  "neoconf": {
    "plugins": { "lua_ls": { "enabled": true } }
  }
}
```

---

## First launch

```bash
nvim
```

lazy.nvim will clone itself, then download and install all plugins. This takes 1–3 minutes.

After the initial install, Mason will be available. Install all LSP servers and tools:

```
:MasonInstall lua-language-server pyright ruff debugpy stylua shfmt vtsls js-debug-adapter jdtls csharp-language-server csharpier netcoredbg fsautocomplete fantomas
```

Or let LazyVim auto-install via `MasonAutoInstall` — just open a file of each language type and Mason will prompt.

---

## GitHub Copilot authentication

On first use, run:

```
:Copilot auth
```

Follow the device-flow URL it prints to link your GitHub account.

---

## Mason-managed tools (full list from this setup)

| Tool | Purpose |
|---|---|
| `lua-language-server` | Lua LSP |
| `pyright` | Python LSP |
| `ruff` | Python linter/formatter |
| `debugpy` | Python DAP adapter |
| `stylua` | Lua formatter |
| `shfmt` | Shell formatter |
| `vtsls` | TypeScript/JS LSP |
| `js-debug-adapter` | JS/TS DAP adapter |
| `jdtls` | Java LSP |
| `csharp-language-server` | C# LSP (csharp-ls) |
| `csharpier` | C# formatter |
| `netcoredbg` | .NET DAP adapter |
| `fsautocomplete` | F# LSP |
| `fantomas` | F# formatter |
| `tree-sitter-cli` | Treesitter parser compilation |

---

## Quick sanity check

```
:checkhealth
:LazyHealth
:Mason
```

All items should be green or have a clear install instruction.

---

## Key bindings (LazyVim defaults — most important)

| Key | Action |
|---|---|
| `<leader>` | `Space` |
| `<leader>ff` | Find files |
| `<leader>fg` | Live grep |
| `<leader>fb` | Buffers |
| `<leader>e` | File explorer |
| `<leader>gg` | Lazygit |
| `<leader>db` | Toggle breakpoint |
| `<leader>dc` | Start / continue debugger |
| `<leader>da` | CopilotChat open |
| `gd` | Go to definition |
| `gr` | Go to references |
| `K` | Hover docs |
| `<leader>ca` | Code action |
| `<leader>cr` | Rename symbol |
