# nvim-neoclip.lua

_This is a story about Bob_ 👷.

_Bob loves vim_ ❤️.

_Bob likes to yank_ ©️.

_Bob knows about registers but sometimes forgets them_ ®️.

_This is what happens to Bob everyday_ 🚧:

* _Bob yanks some line._ 😀
* _Bob yanks another line._ 🤔
* _Bob realises he actually wanted the first._ 🙁
* _But it is gone and Bob is now sad._ 😢

_Don't be like Bob, use neoclip!_ 🎉

`neoclip` is a clipboard manager for neovim inspired by for example [`clipmenu`](https://github.com/cdown/clipmenu).
It records everything that gets yanked in your vim session (up to a limit which is by default 1000 entries but can be configured).
You can then select an entry in the history using [`telescope`](https://github.com/nvim-telescope/telescope.nvim) which then gets populated in a register of your choice.

That's it!

Oh, some more things, you can define an optional filter if you don't want some things to be saved and custom actions to take.

Hold on, `neoclip` optionally also supports persistent history between sessions powered by [`sqlite.lua`](https://github.com/tami5/sqlite.lua).

![neoclip](https://user-images.githubusercontent.com/23341710/129557093-7724e7eb-7427-4c53-aa98-55e624843589.png)


## Installation
```lua
use {
  "AckslD/nvim-neoclip.lua",
  config = function()
    require('neoclip').setup()
  end,
}
```
When `require('neoclip').setup()` is called, only the autocommand (for `TextYankPost` event) is setup to save yanked things. This means that `telescope` is not required at this point if you lazy load it.

If you want to use persistent history between sessions you also need [`sqlite.lua`](https://github.com/tami5/sqlite.lua) installed, for example by:
```lua
use {
  "AckslD/nvim-neoclip.lua",
  requires = {'tami5/sqlite.lua', module = 'sqlite'},
  config = function()
    require('neoclip').setup()
  end,
}
```

## Configuration
You can configure `neoclip` by passing a table to `setup` (all are optional).
The following are the defaults and the keys are explained below:
```lua
use {
  "AckslD/nvim-neoclip.lua",
  config = function()
    require('neoclip').setup({
      history = 1000,
      enable_persistant_history = false,
      db_path = vim.fn.stdpath("data") .. "/databases/neoclip.sqlite3",
      filter = nil,
      preview = true,
      default_register = '"',
      content_spec_column = false,
      on_paste = {
        set_reg = false,
      },
      keys = {
        i = {
          select = '<cr>',
          paste = '<c-p>',
          paste_behind = '<c-k>',
          custom = {},
        },
        n = {
          select = '<cr>',
          paste = 'p',
          paste_behind = 'P',
          custom = {},
        },
      },
    })
  end,
}
```
* `history`: The max number of entries to store (default 1000).
* `enable_persistant_history`: If set to `true` the history is stored on `VimLeavePre` using [`sqlite.lua`](https://github.com/tami5/sqlite.lua) and lazy loaded when querying.
* `db_path`: The path to the sqlite database to store history if `enable_persistant_history=true`.
  Defaults to `vim.fn.stdpath("data") .. "/databases/neoclip.sqlite3` which on my system is `~/.local/share/nvim/databases/neoclip.sqlite3`
* `filter`: A function to filter what entries to store (default all are stored).
  This function filter should return `true` (include the yanked entry) or `false` (don't include it) based on a table as the only argument, which has the following keys:
  * `event`: The event from `TextYankPost` (see `:help TextYankPost` for which keys it contains).
  * `filetype`: The filetype of the buffer where the yank happened.
  * `buffer_name`: The name of the buffer where the yank happened.
* `preview`: Whether to show a preview (default) of the current entry or not.
  Useful for for example multiline yanks.
  When yanking the filetype is recorded in order to enable correct syntax highlighting in the preview.
  NOTE: in order to use the dynamic title showing the type of content and number of lines you need to configure `telescope` with the `dynamic_preview_title = true` option.
* `default_register`: What register to by default when not specifying (e.g. `Telescope neoclip`).
  Can be a string such as `'"'` (single register) or a table of strings such as `{'"', '+', '*'}`.
* `content_spec_colunm`: Can be set to `true` (default `false`) to use instead of the preview.
  It will only show the type and number of lines next to the first line of the entry.
* `on_paste`:
  * `set_reg`: if the register when pressing the key to paste directly.
* `keys`: keys to use for the different actions in insert `i` and normal mode `n`.
  You can also use the `custom` entry to specify custom actions to take on certain key-presses, see [below](#custom-actions) for more details.
  NOTE: these are only set in the `telescope` buffer and you need to setup your own keybindings to for example open `telescope`.

See screenshot section below for how the settings above might affect the looks.

### Custom actions
You can specify custom actions in the `keys` entry in the settings.
For example you can do:
```lua
require('neoclip').setup({
  ...
  keys = {
    ...
    n = {
      ...
      custom = {
        ['<space>'] = function(opts)
          print(vim.inspect(opts))
        end,
      },
    },
  },
})
```
which when pressing `<space>` in normal mode will print something like:
```
{
  register_names = { '"' },
  entry = {
    contents = { "which when pressing `<space>` in normal mode will print something like:" },
    filetype = "markdown",
    regtype = "l"
  }
}
```
to do your custom action and also populate a register and/or paste you can call `neoclip`s built-in handlers, such as:
```lua
require('neoclip').setup({
  ...
  keys = {
    ...
    n = {
      ...
      custom = {
        ['<space>'] = function(opts)
          -- do your stuff
          -- ...
          local handlers = require('neoclip.handlers')
          -- optionally set the registers with the entry
          -- handlers.set_registers(opts.register_names, opts.entry)
          -- optionally paste entry
          -- handlers.paste(opts.entry, 'p')
          -- optionally paste entry behind
          -- handlers.paste(opts.entry, 'P')
        end,
      },
    },
  },
})
```

## Usage
Yank all you want and then do:
```vim
:Telescope neoclip
```
which will show you a history of the yanks that happened in the current session.
If you pick (default `<cr>`) one this will then replace the current `"` (unnamed) register.

If you instead want to directly paste it you can press by default `<c-p>` in insert mode and `p` in normal.
Paste behind is by default `<c-k>` and `P` respectively.

If you want to replace another register with an entry from the history you can do for example:
```vim
:Telescope neoclip a
```
which will replace register `a`.
The register `[0-9a-z]` and `default` (`"`) are supported.

The following special registers are support:
* `"`: `Telescope neoclip unnamed`
* `*`: `Telescope neoclip star`
* `+`: `Telescope neoclip plus`

and `Telescope neoclip` (and `Telescope neoclip default`) will use what you set `default_register` in the `setup`.

You can also specify more registers to populate in a single command with the `extra` keyword argument which
supports registers separated by comma, for example:
```vim
:Telescope neoclip a extra=star,plus,b
```

### Start/stop
If you temporarily don't want `neoclip` to record anything you can use the following calls:
* `:lua require('neoclip').start()`
* `:lua require('neoclip').stop()`
* `:lua require('neoclip').toggle()`

## Tips
* If you lazy load [`telescope`](https://github.com/nvim-telescope/telescope.nvim) with [`packer`](https://github.com/wbthomason/packer.nvim) with for example the key `module = telescope`, then it's better to use e.g. `:lua require('telescope').extensions.neoclip.default()` than `:Telescope neoclip` (or `:lua require('telescope').extensions.neoclip['<reg>']()` over `:Telescope neoclip <reg>`) for keybindings since it will properly load `telescope` before calling the extension.

## Troubleshooting
* For some plugin managers it seems necessary to do
  ```
  :lua require('telescope').load_extension('neoclip')
  ```
  before being able to call `:Telescope neoclip` (packer does not seem to need this).
  However, `:lua require('telescope').extensions.neoclip.default()` seems to work without having to load.
* If using [`packer`](https://github.com/wbthomason/packer.nvim), don't forget to `PackerCompile` after adding the plugin.

## Thanks
* Thanks @cdown for the inspiration with [`clipmenu`](https://github.com/cdown/clipmenu).
* Thanks @fdschmidt93 for help understanding some [`telescope`](https://github.com/nvim-telescope/telescope.nvim) concepts.

## Screenshots
### `preview = true` and `content_spec_column = false`

### `preview = false` and `content_spec_column = true`

### `preview = false` and `content_spec_column = false`
