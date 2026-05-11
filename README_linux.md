# Neovim Setup Guide — Linux (Ubuntu/Debian/Pop!_OS)

Reproduces this exact Neovim environment: **LazyVim** distribution with a matrix colorscheme, GitHub Copilot + CopilotChat, DAP debugging, and LSP/tooling for TypeScript, Python, Java, and C#/.NET.

---

## Requirements

### Neovim

Install **v0.12+** (this setup uses v0.12.2).

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
sudo apt install git gcc g++ make ripgrep fd-find fzf lazygit curl unzip
```

> `fd` installs as `fdfind` on Debian/Ubuntu. Symlink it:
> ```bash
> sudo ln -sf $(which fdfind) /usr/local/bin/fd
> ```

### Language runtimes

#### Node.js (TypeScript/JS) — via nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
# restart shell, then:
nvm install 24
nvm use 24
nvm alias default 24
```

> **Neovim doesn't source nvm's shell init**, so plugins that need `node` (Copilot, CopilotChat)
> will fail with "could not determine node version". Fix it by adding this to
> `~/.config/nvim/lua/config/options.lua`:
>
> ```lua
> local nvm_dir = os.getenv("NVM_DIR") or (os.getenv("HOME") .. "/.nvm")
> local node_bin = vim.fn.trim(vim.fn.system("ls -d " .. nvm_dir .. "/versions/node/*/bin 2>/dev/null | sort -V | tail -1"))
> if node_bin ~= "" then
>   vim.env.PATH = node_bin .. ":" .. vim.env.PATH
> end
> ```
>
> This picks the latest installed version automatically and doesn't break when you upgrade Node.

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

#### Maven (required for Java test keybindings)

`<leader>ja` and `<leader>jA` shell out to `mvn`.

```bash
sdk install maven          # recommended
# or
sudo apt install maven
```

#### .NET / C# — via Microsoft packages

```bash
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt update && sudo apt install dotnet-sdk-10
```

Then install csharp-ls (LSP server used instead of broken OmniSharp):

```bash
dotnet tool install --global csharp-ls
```

`dotnet tool install --global` puts binaries in `~/.dotnet/tools`. Add it to your shell profile:

```bash
# ~/.zshrc or ~/.bashrc
export PATH="$HOME/.dotnet/tools:$PATH"
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

### `lang-csharp.lua` — Replace broken OmniSharp with csharp-ls + DAP

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
  {
    "mfussenegger/nvim-dap",
    optional = true,
    config = function()
      local dap = require("dap")
      dap.adapters.coreclr = {
        type = "executable",
        command = vim.fn.stdpath("data") .. "/mason/bin/netcoredbg",
        args = { "--interpreter=vscode" },
      }
      -- Run dotnet build before every coreclr debug session (replaces preLaunchTask,
      -- which nvim-dap does not support).
      dap.listeners.before.launch["dotnet_build"] = function(session)
        if session.config.type ~= "coreclr" then return end
        vim.notify("dotnet: building...", vim.log.levels.INFO)
        local out = vim.fn.system("dotnet build --configuration Debug --verbosity quiet 2>&1")
        if vim.v.shell_error ~= 0 then
          vim.notify("Build failed:\n" .. out, vim.log.levels.ERROR)
        end
      end
      dap.configurations.cs = {
        {
          type = "coreclr",
          name = "Launch (netcoredbg)",
          request = "launch",
          program = function()
            local csproj = vim.fn.glob(vim.fn.getcwd() .. "/*.csproj")
            if csproj ~= "" then
              local proj = vim.fn.fnamemodify(csproj, ":t:r")
              local tf = "net10.0"
              local f = io.open(csproj, "r")
              if f then
                local content = f:read("*a")
                f:close()
                tf = content:match("<TargetFramework>(.-)</TargetFramework>") or tf
              end
              local dll = vim.fn.getcwd() .. "/bin/Debug/" .. tf .. "/" .. proj .. ".dll"
              if vim.fn.filereadable(dll) == 1 then return dll end
            end
            return vim.fn.input("Path to .dll: ", vim.fn.getcwd() .. "/bin/Debug/", "file")
          end,
          cwd = "${workspaceFolder}",
          stopAtEntry = false,
        },
        {
          type = "coreclr",
          name = "Debug Tests (netcoredbg)",
          request = "launch",
          -- xUnit v3 test DLLs are self-executable — launch directly,
          -- no vstest child process that the debugger can't follow.
          program = function()
            local dlls = vim.fn.glob(vim.fn.getcwd() .. "/**/*.Tests/bin/Debug/*/*.Tests.dll", false, true)
            if #dlls > 0 then return dlls[1] end
            return vim.fn.input("Path to .Tests.dll: ", vim.fn.getcwd() .. "/", "file")
          end,
          args = {},
          cwd = function()
            local dirs = vim.fn.glob(vim.fn.getcwd() .. "/**/*.Tests", false, true)
            return #dirs > 0 and dirs[1] or "${workspaceFolder}"
          end,
          stopAtEntry = false,
        },
        {
          type = "coreclr",
          name = "Attach (netcoredbg)",
          request = "attach",
          processId = require("dap.utils").pick_process,
        },
      }
    end,
  },
}
```

#### Test project — `hello.Tests/hello.Tests.csproj`

Use **xUnit v3** (`xunit.v3`), not `xunit` v2. xUnit v3 generates a proper `Main` entry
point in the test DLL so netcoredbg can launch it directly. With xUnit v2, `dotnet test`
spawns a separate `testhost` child process — netcoredbg attaches to the wrong process and
all tests are skipped.

> **`using Xunit;` is required.** xUnit v3 does not inject itself via `ImplicitUsings`.
> Add it explicitly at the top of every test file or `[Fact]`, `[Theory]`, and `Assert`
> won't resolve — even though the package is referenced.

```xml
<ItemGroup>
  <PackageReference Include="coverlet.collector"         Version="6.0.4" />
  <PackageReference Include="Microsoft.NET.Test.Sdk"     Version="17.14.1" />
  <PackageReference Include="xunit.v3"                   Version="3.2.2" />
  <PackageReference Include="xunit.runner.visualstudio"  Version="3.1.5" />
</ItemGroup>
```

The main project must also exclude the test subdirectory from its own glob and expose
internals:

```xml
<!-- hello.csproj -->
<ItemGroup>
  <Compile Remove="hello.Tests/**" />
</ItemGroup>
<ItemGroup>
  <InternalsVisibleTo Include="hello.Tests" />
</ItemGroup>
```

#### Project — `.vscode/launch.json`

nvim-dap auto-loads `.vscode/launch.json` on demand — no manual `load_launchjs` call needed.

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch hello.dll",
      "type": "coreclr",
      "request": "launch",
      "program": "${workspaceFolder}/bin/Debug/net10.0/hello.dll",
      "args": [],
      "cwd": "${workspaceFolder}",
      "stopAtEntry": false,
      "console": "internalConsole"
    },
    {
      "name": "xUnit: All Tests",
      "type": "coreclr",
      "request": "launch",
      "program": "${workspaceFolder}/hello.Tests/bin/Debug/net10.0/hello.Tests.dll",
      "args": [],
      "cwd": "${workspaceFolder}/hello.Tests",
      "stopAtEntry": false,
      "console": "internalConsole"
    }
  ]
}
```

### `lang-java.lua` — nvim-jdtls plugin spec

```lua
return {
  {
    "mfussenegger/nvim-jdtls",
    ft = "java",
    -- actual config lives in ftplugin/java.lua, which nvim-jdtls recommends
    -- so that start_or_attach runs per-buffer with the correct root_dir
    dependencies = { "mfussenegger/nvim-dap" },
  },
}
```

### `ftplugin/java.lua` — per-buffer jdtls config

Create at `~/.config/nvim/ftplugin/java.lua`. nvim-jdtls's recommended pattern: this file
runs for every Java buffer so `start_or_attach` always gets the correct `root_dir`.

```lua
local jdtls = require("jdtls")
local mason_packages = vim.fn.stdpath("data") .. "/mason/packages"

local root_dir = jdtls.setup.find_root({ ".git", "mvnw", "gradlew", "pom.xml", "build.gradle" })
if not root_dir then return end

local project_name = vim.fn.fnamemodify(root_dir, ":t")
local workspace_dir = vim.fn.stdpath("data") .. "/jdtls-workspaces/" .. project_name

local java_home = vim.fn.getenv("JAVA_HOME")
if java_home == vim.NIL or java_home == "" then
  java_home = vim.fn.expand("~/.sdkman/candidates/java/current")
end

local bundles = {
  vim.fn.glob(mason_packages .. "/java-debug-adapter/extension/server/com.microsoft.java.debug.plugin-*.jar", true),
}
vim.list_extend(bundles, vim.split(
  vim.fn.glob(mason_packages .. "/java-test/extension/server/*.jar", true),
  "\n", { plain = true, trimempty = true }
))

-- Register the java DAP adapter immediately (before on_attach) so keybindings
-- that trigger dap.run() work as soon as the file opens.
jdtls.setup_dap({ hotcodereplace = "auto" })

local map = function(keys, func, desc)
  vim.keymap.set("n", keys, func, { buffer = true, desc = desc })
end

map("<leader>ji",  function() jdtls.organize_imports() end,    "Java: Organize Imports")
map("<leader>jt",  function() jdtls.test_class() end,          "Java: Test Class")
map("<leader>jm",  function() jdtls.test_nearest_method() end, "Java: Test Nearest Method")
map("<leader>jev", function() jdtls.extract_variable() end,    "Java: Extract Variable")
map("<leader>jem", function() jdtls.extract_method() end,      "Java: Extract Method")

-- Run all tests (no debugger)
map("<leader>ja", function()
  Snacks.terminal("cd " .. root_dir .. " && mvn test", { cwd = root_dir })
end, "Java: Run All Tests")

-- Debug all tests:
--   forkCount=0  → tests run inside Maven's own JVM (no surefire fork)
--   MAVEN_OPTS   → JDWP agent on that same JVM
--   on_stdout    → auto-attach the moment Maven prints "Listening for transport"
--
-- Why not -Dmaven.surefire.debug?  That flag forks a child JVM; surefire waits
-- for the booter handshake while the child is frozen by JDWP → timeout → exit 2.
map("<leader>jA", function()
  local dap = require("dap")
  local attached = false
  local output = ""

  vim.cmd("botright split | resize 12")
  local term_buf = vim.api.nvim_create_buf(false, true)
  vim.api.nvim_win_set_buf(0, term_buf)

  local jdwp = "-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=localhost:5005"
  local cmd = "cd " .. root_dir
    .. " && MAVEN_OPTS=" .. vim.fn.shellescape(jdwp)
    .. " mvn test -DforkCount=0"

  vim.fn.termopen(cmd, {
    on_stdout = function(_, data)
      if attached then return end
      output = output .. table.concat(data, "")
      if output:match("Listening for transport") then
        attached = true
        vim.schedule(function()
          jdtls.setup_dap({ hotcodereplace = "auto" })
          dap.run({
            type     = "java",
            request  = "attach",
            name     = "Debug All Tests",
            hostName = "localhost",
            port     = 5005,
          })
        end)
      end
    end,
  })
  vim.cmd("wincmd p")
end, "Java: Debug All Tests")

-- Debug main program: setup_dap_main_class_configs is async; on_ready fires once
-- jdtls has scanned the classpath, then dap.continue() shows the picker.
map("<leader>jd", function()
  require("jdtls.dap").setup_dap_main_class_configs({
    on_ready = function() require("dap").continue() end,
  })
end, "Java: Debug Program")

jdtls.start_or_attach({
  cmd = {
    vim.fn.stdpath("data") .. "/mason/bin/jdtls",
    "--jvm-arg=-Xmx1g",
    "-data", workspace_dir,
  },
  root_dir = root_dir,
  settings = {
    java = {
      configuration = {
        runtimes = {
          { name = "JavaSE-25", path = java_home },
        },
      },
      format     = { enabled = true },
      saveActions = { organizeImports = true },
    },
  },
  init_options = { bundles = bundles },
  on_attach = function() end,
})
```

#### Project — `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Launch Greeter",
      "request": "launch",
      "mainClass": "com.fit.hello.Greeter",
      "projectName": "hello"
    }
  ]
}
```

> Test debugging does **not** use a launch.json entry. `<leader>jt` / `<leader>jm`
> go through the `java-test` Mason bundle via jdtls. `<leader>jA` uses
> `MAVEN_OPTS` + `forkCount=0` and attaches automatically.

---

## Setting up a solution file (.NET)

`dotnet test` run from the project root requires a `.sln` that references both projects.
Without it you must `cd` into the test folder manually every time.

```bash
cd MyProject
dotnet new sln --name MyProject
dotnet sln add MyProject.csproj
dotnet sln add MyProject.Tests/MyProject.Tests.csproj
dotnet test   # now works from the root
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

After the initial install, Mason will be available. **Wait for the Lazy UI to show "Done" before
running `:MasonInstall`** — typing it while plugins are still loading produces "Not an editor
command". Install all LSP servers and tools:

```
:MasonInstall lua-language-server pyright ruff debugpy stylua shfmt vtsls js-debug-adapter jdtls java-debug-adapter java-test csharp-language-server csharpier netcoredbg fsautocomplete fantomas
```

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
| `java-debug-adapter` | Provides `com.microsoft.java.debug.plugin-*.jar` for DAP |
| `java-test` | JUnit runner JARs for in-editor test execution |
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

### Java — jdtls first-run indexing delay

When you open a Java file for the first time in a project, jdtls indexes the entire
classpath. This takes **30–60 seconds**. All `<leader>j*` keybindings silently do nothing
during this window. Wait for the `jdtls: server ready` notification in the status line
before using any debug or refactor commands.

### Java — when jdtls gets stuck

If jdtls behaves strangely after a package rename, branch switch, or dependency change,
wipe its workspace cache and restart:

```
:JdtWipeDataAndRestart
```

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

### Java-specific keybindings (`ftplugin/java.lua`)

| Key | Action |
|---|---|
| `<leader>jd` | Debug main program (auto-detects `main()` via jdtls) |
| `<leader>jt` | Debug tests in current class |
| `<leader>jm` | Debug nearest test method |
| `<leader>ja` | Run all tests (`mvn test`) |
| `<leader>jA` | Debug all tests (MAVEN_OPTS + forkCount=0, auto-attach) |
| `<leader>ji` | Organize imports |
| `<leader>jev` | Extract variable |
| `<leader>jem` | Extract method |

> **`<leader>jt` and `<leader>jm` require JUnit 5 (Jupiter).** These keybindings go through
> the `java-test` Mason bundle, which only discovers JUnit 5 tests. A project still on
> JUnit 4 will show no tests found. Use `junit-jupiter` as your test dependency.

---

## Design decisions

### Why xUnit v3 for .NET tests?

xUnit v2 tests are run by a vstest `testhost` child process. netcoredbg attaches to the
parent `dotnet test` process — the wrong one — so all tests are skipped. xUnit v3 builds
the test DLL with its own `Main`, making it directly launchable by netcoredbg.

### Why `forkCount=0` + `MAVEN_OPTS` for Java debug-all-tests?

`-Dmaven.surefire.debug` injects JDWP into surefire's *forked* child JVM. Surefire's
parent then waits for the booter handshake, but the child is frozen (`suspend=y`). This
race causes exit code 2. With `forkCount=0` the tests run inside Maven's own JVM;
`MAVEN_OPTS` injects JDWP there, and the handshake problem disappears.

### Why `jdtls.setup_dap()` at the top level of `ftplugin/java.lua`?

If called only inside `on_attach`, the `java` DAP adapter isn't registered until jdtls
finishes indexing (~10–30 s). Pressing any debug key before that produces
"Config references missing adapter `java`". Calling it at ftplugin load time registers
the adapter immediately; actual jdtls communication only happens when a debug session
starts, by which time jdtls is ready.

### Why the `on_ready` callback in `<leader>jd`?

`setup_dap_main_class_configs` is async — it queries jdtls for classes with a `main`
method. Calling `dap.continue()` immediately after would race against an empty config
list. The `on_ready` callback fires only once the list is populated.
