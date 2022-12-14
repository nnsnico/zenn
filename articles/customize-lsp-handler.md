---
title: "nvim-lspの設定をlsp-handlerでカスタマイズする"
emoji: "🐸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim"]
published: true
---

:::message
この記事は Vim Advent Calendar 2022 その 2 の 14 日目の記事です🎄
https://qiita.com/advent-calendar/2022/vim

アドベントカレンダーは初参加です。よろしくお願いします！
:::

昨今は Neovim を使用してコーディングする場合、builtin LSP の機能を設定に盛り込んで開発する方が多いのではないかと思います。
builtin LSP にはデフォルトで LSP の機能を実現するための実装は最低限されていますが、これを自分好みにカスタマイズしたいときもあるでしょう。
そういった方のために、LSP の各ハンドラをオーバーライドできる設定があります。それが `vim.lsp.handlers` です。
今回は実際の Neovim の help 内に書いてある内容を追いつつオーバーライドする方法を見てみます。

## デフォルトで設定されている機能

前提としてデフォルトで定義されている機能はどのくらいあるのでしょう？
それは `:help lsp-method` で見ることができます。
書いてある通り、 `textDocument/hover` や `textDocument/definition` などはデフォルトで定義されています。
そのため、こちらで何かしら機能追加したりプラグインを追加したりすることなく、[`neovim/nvim-lspconfigの設定例`](https://github.com/neovim/nvim-lspconfig#suggested-configuration)のように、キーマップを設定してあげるだけで十分に機能を果たせます。

<!-- textlint-disable -->
:::details デフォルトで設定されている LSP の機能一覧
<!-- textlint-enable -->

- callHierarchy/incomingCalls
- callHierarchy/outgoingCalls
- textDocument/codeAction
- textDocument/completion
- textDocument/declaration
- textDocument/definition
- textDocument/documentHighlight
- textDocument/documentSymbol
- textDocument/formatting
- textDocument/hover
- textDocument/implementation
- textDocument/publishDiagnostics
- textDocument/rangeFormatting
- textDocument/references
- textDocument/rename
- textDocument/signatureHelp
- textDocument/typeDefinition
- window/logMessage
- window/showMessage
- window/showDocument
- window/showMessageRequest
- workspace/applyEdit
- workspace/symbol

:::

## 機能をオーバーライドする

上記の機能をオーバーライドするには 2 通りあります。

### function(err, result, ctx, config)を定義する

`:help lsp-handler` で詳細が見られます。

ざっくり言うと各引数はそれぞれ以下のようになっています。

- `err`: LSP のリクエストの失敗時に返るレスポンス値。通常は nil
- `result`: LSP のリクエストの成功時に返るレスポンス値
- `ctx`: ハンドラーの付加情報。メソッド名やバッファ番号など
- `config`: ハンドラーに対する設定

上記の引数を受け取る関数を `vim.lsp.handlers` に渡すことでオーバーライド可能です。

```lua
vim.lsp.handlers['textDocument/definition'] = function(err, result, ctx, config)
    ...
end
```

### vim.lsp.with を使う

`:help vim.lsp.with()` で詳細が見られます。

`vim.lsp.with()` の実装の中身を見れば、やっていることは分かりやすいです。
`handler` で上記の関数をとって、`override_config` で設定をマージしています。

```lua
function lsp.with(handler, override_config)
  return function(err, result, ctx, config)
    return handler(err, result, ctx, vim.tbl_deep_extend('force', config or {}, override_config))
  end
end
```

## 試してみる

`textDocument/definition` を例に試してみます。

まずは `function(err, result, ctx, config)` で定義するパターンです。
レスポンスから得られる対象が 1 つの場合、素直にそのまま split して開くようにして、それ以上の場合は location list で一覧を開くようにしてみます。(デフォルト設定の場合、split ではなく edit 同様、現在のウィンドウに遷移先を描画します)
以下がその例です。

`textDocument/definition` や `textDocument/typeDefinition` 、 `textDocument/references` 等のレスポンスは [`Location`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#location)や [`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#locationLink)のリストの型で返ってきます。
これらは `vim.lsp.util.locations_to_items()` 関数を使うことで quickfix/location list で扱える形に変換してくれます。

```lua
vim.lsp.handlers['textDocument/definition'] = function(_, result, _, _)
  local loclist = vim.lsp.util.locations_to_items(results, 'utf-8')
  if #loclist == 1 then
    vim.cmd('split ' .. loclist[1].filename)
  else
    vim.fn.setloclist(0, loclist, ' ')
    vim.cmd('lopen')
  end
end
```

`vim.lsp.with()` でも例を書いてみます。
上記の挙動をローカル関数にし、config として `open_handler` を追加することでファイルを開く方法を選べるようにしてみます。
これにより、デフォルトは `edit` で開きつつ、`open_handler` の設定次第で `split` や `vsplit` で開くことができるようになりました。

```lua
local function handle_definition(_, results, _, config)
  local loclist = vim.lsp.util.locations_to_items(results, 'utf-8')
  if #loclist == 1 then
    local open_handler = config.open_handler or 'edit'
    vim.cmd(open_handler .. ' ' .. loclist[1].filename)
  else
    vim.fn.setloclist(0, loclist, ' ')
    vim.cmd('lopen')
  end
end

vim.lsp.handlers['textDocument/definition'] = vim.lsp.with(handle_definition, {
  open_handler = 'vsplit' -- vsplit で遷移先ファイルを開くようにする
})
```

## おわり

builtin LSP のデフォルト機能をオーバーライドすることで lsp-handler をカスタマイズする方法を見てみました。
通常の設定でも機能としては十分果たせますが、今回紹介した機能を使えばより自分の使いやすい形にカスタマイズが可能です。
また LSP に限らず、いつでも手元でヘルプを確認でき、かつ自分が満足するカスタマイズを追求できるというのは Vim/Neovim そのものの良さでもあると改めて実感できました ☺️
