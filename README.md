# LazyVim - C# & Razor LSPs
Configuration to use LazyVim together with roslyn and razor LSPs

Based on: https://gist.github.com/navazjm/de072b46e4a203184dd647551502d6c8#file-csharp_lsp_nvim-md
But configured to use LazyVim instead of the new built-in neovim package-manager.

## Default neovim configuration: 

```lua
--- INITIAL NVIM SETUP
vim.g.mapleader = " "

-- Basic editor options
vim.o.number = true
vim.o.relativenumber = true
vim.o.cursorline = true
vim.o.scrolloff = 10
vim.o.list = true
vim.o.confirm = true
vim.o.ignorecase = true
vim.o.smartcase = true

-- Sync clipboard (delayed to UIEnter)
vim.api.nvim_create_autocmd("UIEnter", {
  callback = function()
    vim.o.clipboard = "unnamedplus"
  end,
})

-- Highlight yanked text
vim.api.nvim_create_autocmd("TextYankPost", {
  callback = function()
    vim.hl.on_yank()
  end,
})

-- Terminal navigation keymaps
vim.keymap.set("t", "<Esc>", "<C-\\><C-n>")
vim.keymap.set({ "t", "i" }, "<A-h>", "<C-\\><C-n><C-w>h")
vim.keymap.set({ "t", "i" }, "<A-j>", "<C-\\><C-n><C-w>j")
vim.keymap.set({ "t", "i" }, "<A-k>", "<C-\\><C-n><C-w>k")
vim.keymap.set({ "t", "i" }, "<A-l>", "<C-\\><C-n><C-w>l")
vim.keymap.set("n", "<A-h>", "<C-w>h")
vim.keymap.set("n", "<A-j>", "<C-w>j")
vim.keymap.set("n", "<A-k>", "<C-w>k")
vim.keymap.set("n", "<A-l>", "<C-w>l")

-- Optional: user command for git blame
vim.api.nvim_create_user_command("GitBlameLine", function()
  local line_number = vim.fn.line(".")
  local filename = vim.api.nvim_buf_get_name(0)
  print(vim.fn.system({ "git", "blame", "-L", line_number .. ",+1", filename }))
end, { desc = "Print git blame for current line" })

-- Lazy.nvim Bootstrap
require("config.lazy")

local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git",
    "clone",
    "--filter=blob:none",
   "https://github.com/folke/lazy.nvim.git",
    "--branch=stable",
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

-- help recognize razor files
vim.filetype.add({
  extension = {
    razor = "razor",
    cshtml = "razor",
  },
})

local rzls_base_path = vim.fn.expand("~/tools/razor/artifacts/LanguageServer/Debug/net9.0/linux-x64")
require("rzls").setup({
  path = vim.fs.joinpath(rzls_base_path, "rzls"),
  on_attach = function(client, bufnr) end,
  capabilities = {},
})
vim.lsp.config("roslyn", {
  handlers = require("rzls.roslyn_handlers"),
  cmd = {
    "dotnet",
    vim.fn.expand("~/tools/roslyn/Microsoft.CodeAnalysis.LanguageServer.dll"),
    "--stdio",
    "--logLevel=Information",
    "--extensionLogDirectory=" .. vim.fs.dirname(vim.lsp.log.get_filename()),
    "--razorSourceGenerator=" .. vim.fs.joinpath(rzls_base_path, "Microsoft.CodeAnalysis.Razor.Compiler.dll"),
    "--razorDesignTimePath="
      .. vim.fs.joinpath(rzls_base_path, "Targets", "Microsoft.NET.Sdk.Razor.DesignTime.targets"),
    "--extension",
    vim.fn.expand(
      "~/tools/razor/artifacts/bin/Microsoft.VisualStudioCode.RazorExtension/Debug/net9.0/Microsoft.VisualStudioCode.RazorExtension.dll"
    ),
  },
})
vim.lsp.enable({ "roslyn" })

-- local ensure_installed =
local treesitter = require("nvim-treesitter")
treesitter.setup({
  ensure_installed = { "c_sharp", "html", "razor" },
  sync_install = false,
  auto_install = true,
  ignore_install = {},
  highlight = {
    enable = true,
    -- disable slow treesitter highlight for large files
    disable = function(lang, buf)
      local max_filesize = 100 * 1024 -- 100 KB
      local ok, stats = pcall(vim.loop.fs_stat, vim.api.nvim_buf_get_name(buf))
      if ok and stats and stats.size > max_filesize then
        return true
      end
    end,
    additional_vim_regex_highlighting = false,
  },
})
```

## LazyVim configuration: `~/.config/nvim/lazy.lua`
```lua
-- config/lazy.lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not (vim.uv or vim.loop).fs_stat(lazypath) then
  local lazyrepo = "https://github.com/folke/lazy.nvim.git"
  local out = vim.fn.system({
    "git",
    "clone",
    "--filter=blob:none",
    "--branch=stable",
    lazyrepo,
    lazypath,
  })
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
    { "LazyVim/LazyVim", import = "lazyvim.plugins" },
    -- LSP Core
    { "neovim/nvim-lspconfig" },
    -- Syntax Highlighting
    { "nvim-treesitter/nvim-treesitter", build = ":TSUpdate" },
    -- Razor & Roslyn
    { "tris203/rzls.nvim" },
    { "seblyng/roslyn.nvim" },
    { import = "plugins" }, -- This will include plugins/razor.lua
  },
  defaults = {
    lazy = false,
    version = false,
  },
  install = { colorscheme = { "tokyonight", "habamax" } },
  checker = { enabled = true, notify = false },
  performance = {
    rtp = {
      disabled_plugins = { "gzip", "tarPlugin", "tohtml", "zipPlugin" },
    },
  },
})
```
## Razor LSP configuration: `~/.config/nvim/lua/plugins/razor.lua`
```lua
return {
  -- Razor Language Server (RZLS)
  {
    "tris203/rzls.nvim",
    dependencies = { "neovim/nvim-lspconfig" },
    config = function()
      local rzls_base_path = vim.fn.expand("~/tools/razor/artifacts/LanguageServer/Debug/net9.0/linux-x64")
      require("rzls").setup({
        path = vim.fs.joinpath(rzls_base_path, "rzls"),
        on_attach = function(client, bufnr)
          -- optional buffer-specific setup
        end,
        capabilities = {},
      })
    end,
  },

  -- Roslyn LSP
  {
    "seblyng/roslyn.nvim",
    dependencies = { "tris203/rzls.nvim", "neovim/nvim-lspconfig" },
    config = function()
      local rzls_base_path = vim.fn.expand("~/tools/razor/artifacts/LanguageServer/Debug/net9.0/linux-x64")
      require("roslyn").setup({
        handlers = require("rzls.roslyn_handlers"),
        cmd = {
          "dotnet",
          vim.fn.expand("~/tools/roslyn/Microsoft.CodeAnalysis.LanguageServer.dll"),
          "--stdio",
          "--logLevel=Information",
          "--extensionLogDirectory=" .. vim.fs.dirname(vim.lsp.log.get_filename()),
          "--razorSourceGenerator=" .. vim.fs.joinpath(rzls_base_path, "Microsoft.CodeAnalysis.Razor.Compiler.dll"),
          "--razorDesignTimePath=" .. vim.fs.joinpath(rzls_base_path, "Targets", "Microsoft.NET.Sdk.Razor.DesignTime.targets"),
          "--extension",
          vim.fn.expand("~/tools/razor/artifacts/bin/Microsoft.VisualStudioCode.RazorExtension/Debug/net9.0/Microsoft.VisualStudioCode.RazorExtension.dll"),
        },
      })
    end,
  },
}
```

## Roslyn LSP configuration: `~/.config/nvim/lua/plugins/roslyn.lua`
```lua
return {
  "seblyng/roslyn.nvim",
  dependencies = { "neovim/nvim-lspconfig" },
  config = function()
    -- Disable LazyVim’s default C# server if active
    local lspconfig = require("lspconfig")
    if lspconfig.omnisharp then
      lspconfig.omnisharp = nil
    end

    require("roslyn").setup({
      on_attach = function(client, bufnr)
        -- Optional: custom keymaps for Roslyn
        local opts = { buffer = bufnr }
        vim.keymap.set("n", "gd", vim.lsp.buf.definition, opts)
        vim.keymap.set("n", "K", vim.lsp.buf.hover, opts)
      end,
    })
  end,
}
```
