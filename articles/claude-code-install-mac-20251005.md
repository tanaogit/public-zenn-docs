---
title: "MacにClaude Codeをインストールした時に発生する権限エラーの解決方法"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claude", "claudecode"]
published: false
---


# はじめに
こんにちわ！
少し出遅れた感はありますが、最近になってAI駆動開発について勉強しています。
早速 **Claude Code** を導入してみようと思い、Pro プランを契約して Mac にインストールしてみたところ、いきなり**権限エラー**が発生してインストールに失敗しました。
今回はその解決方法について調べたところ、公式が推奨している解決策の**ネイティブ Claude Code のインストール**に関しての記事があまり見当たらなかったので今回記事にしようと思いました。

## 筆者の環境
 - macOS


# Claude Code の導入
まずは、公式ドキュメントにあるClaude.aiにアクセスして契約プランを決めてアカウントを作成してください。
https://docs.claude.com/ja/docs/claude-code/overview


アカウントが作成できたら、早速 Claude Code をインストールしていきましょう。
```
npm install -g @anthropic-ai/claude-code
```

私はここで権限エラーによっていきなりインストールに失敗しました。
Claude Code を早く使ってみたいという思いと、コマンド1つでインストールできるのは便利だなと思っていた矢先だったので、早くも出鼻をくじかれました。
```
npm error code EACCES
npm error syscall mkdir
npm error path /usr/local/lib/node_modules/@anthropic-ai
npm error errno -13
npm error Error: EACCES: permission denied, mkdir '/usr/local/lib/node_modules/
```

エラー内容を見ると、npm install -g でグローバルインストールしようとして /usr/local/lib/node_modules/ 以下にディレクトリを作成しようとしたときに権限がなくて失敗しています。（EACCES: permission denied, mkdir '/usr/local/lib/node_modules/...）
:::message
npm のグローバルインストールに sudo を使う方法もありますが、これはシステムの権限を直接変更してしまうため推奨されていません。
:::


# 権限エラーの解決方法
こちらの解決方法について調査したところ、公式に対処法が記載してありました。
https://docs.claude.com/ja/docs/claude-code/troubleshooting#linux%E3%81%A8mac%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E5%95%8F%E9%A1%8C%EF%BC%9A%E6%A8%A9%E9%99%90%E3%81%BE%E3%81%9F%E3%81%AF%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89%E3%81%8C%E8%A6%8B%E3%81%A4%E3%81%8B%E3%82%89%E3%81%AA%E3%81%84%E3%82%A8%E3%83%A9%E3%83%BC

どうやら、Claude Code は npm や Node.js に依存しないネイティブインストールが可能みたいです。
:::message
ただし、2025年10月5日時点ではインストーラーはベータ版でした。
:::

とりあえず、Claude Code の安定版をインストールしたかったので下記コマンドを実行しました。
```
# 安定版をインストール（デフォルト）
curl -fsSL https://claude.ai/install.sh | bash
```

インストールが成功しました。正常に動作するか確認するために --version のコマンドを実行してみます。
```
claude --version
```

またも、失敗しました。次は claude コマンドが見つからないみたいです。
```
# エラーログ
claude not found
```

公式では、インストール時にシンボリックリンクを追加しますと記載されていましたがうまくできていなかったみたいです。環境パスを追加します。
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
# 環境パスの変更を反映
source ~/.zshrc
```

環境パスの追加が完了したので、再度 claude --version を実行して動作を確認してみます。
```
2.0.5 (Claude Code)
```

正常にバージョンが返されたのでインストール完了です。
念の為、インストールが正常にされているか下記コマンドで確認できるみたいです。
```
claude doctor
```

# まとめ
最初のインストールでいきなり失敗したので驚きましたが、無事インストールすることができました。最後まで読んでいただきありがとうございました。
