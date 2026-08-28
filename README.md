# Zenn 記事リポジトリ

小手箱（https://kotebako.github.io/）への流入用の記事置き場。

Zenn の GitHub 連携で、このリポジトリの `articles/*.md` を push すると自動で公開される。
連携は公式機能なので、規約上まったく問題ない。

## セットアップ（初回のみ・3分）

1. GitHub で **`zenn-content`** という名前の空リポジトリを作る（Public）
   - https://github.com/new
   - README・.gitignore・license は**何も付けない**
2. https://zenn.dev/dashboard/deploys を開く
3. 「リポジトリを連携する」→ `zenn-content` を選ぶ
4. ブランチは `main`

これ以降、push するだけで公開される。

## 記事の書き方

`articles/<slug>.md` に置く。slug は半角英数字とハイフンで12〜50文字。

```markdown
---
title: "記事タイトル"
emoji: "🧰"
type: "tech"          # tech（技術記事）または idea（アイデア）
topics: ["javascript", "個人開発"]
published: true       # false にすると下書き
---

本文
```

## Qiita にも同時に出す

同じファイルをそのまま Qiita へ投稿できる。frontmatter は自動変換される。

```
auto/pipeline/post_qiita.py --dry articles/<slug>.md   # 確認
auto/pipeline/post_qiita.py       articles/<slug>.md   # 投稿
auto/pipeline/post_qiita.py --list                     # 投稿済み一覧とLGTM数
```

## 投稿ペースについて

**週1本を上限にする。**

自動投稿の仕組みはあるが、量産はしない。ZennもQiitaも読者コミュニティであって
検索エンジンではないため、質の低い記事を並べると評価が下がり、逆効果になる。
1本の記事に実装の詰まりどころを具体的に書く方が、10本の薄い記事より流入する。
