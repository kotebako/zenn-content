---
title: "外部ライブラリなしで、ブラウザからPDFを生成する"
emoji: "📄"
type: "tech"
topics: ["javascript", "pdf", "canvas", "個人開発", "フロントエンド"]
published: true
---

画像を1つのPDFにまとめるツールを作りました。jsPDFもpdf-libも使わず、PDFのバイト列を手で組み立てています。

やってみたら思ったより短く済んだので、手順を残します。

## なぜライブラリを使わなかったか

pdf-lib は minify 後でも 300KB ほどあります。今回やりたいのは「JPEGを並べてページにする」だけで、フォント埋め込みもフォーム編集もいりません。**300KBのうち使うのは数%です。**

それに、JPEGは**PDFにそのまま埋め込めます**。再エンコードが不要なので、やることは実質「バイト列を正しい順序で並べる」だけになります。

## PDFの最小構造

PDFはテキストベースのフォーマットです。中身はこうなっています。

```
%PDF-1.4
1 0 obj
<< /Type /Catalog /Pages 2 0 R >>
endobj
2 0 obj
<< /Type /Pages /Count 1 /Kids [3 0 R] >>
endobj
...
xref
0 7
0000000000 65535 f 
0000000015 00000 n 
...
trailer
<< /Size 7 /Root 1 0 R >>
startxref
1234
%%EOF
```

必要なのは4種類のオブジェクトだけです。

| オブジェクト | 役割 |
| --- | --- |
| Catalog | 文書のルート。Pagesを指す |
| Pages | ページの一覧 |
| Page | 1ページ。用紙サイズと、使うリソースと、描画命令を指す |
| XObject (Image) | 埋め込む画像そのもの |

加えて、各ページの**描画命令を書いたContentsストリーム**が要ります。

## JPEGはそのまま入る

ここが一番おいしい部分です。PDFの画像オブジェクトは、フィルタに `/DCTDecode` を指定すると**JPEGのバイト列をそのまま格納できます**。デコードもエンコードも不要です。

```js
obj(imgNum,
  '<< /Type /XObject /Subtype /Image' +
  ' /Width ' + w + ' /Height ' + h +
  ' /ColorSpace /DeviceRGB /BitsPerComponent 8' +
  ' /Filter /DCTDecode' +
  ' /Length ' + data.length + ' >>', data);
```

`data` は `canvas.toBlob()` で得たJPEGを `arrayBuffer()` した `Uint8Array` です。中身を解析する必要すらありません。

PNGを入れたい場合は `/FlateDecode` を使いますが、PNGのIDATチャンクをそのまま渡せるわけではない（PDFはPNGのフィルタ方式をそのままは解さない）ので面倒です。**一度canvasでJPEGに揃えてしまう**方が圧倒的に簡単で、書類や写真ならそれで困りません。

## 座標系に注意

PDFの原点は**左下**です。ブラウザのcanvasは左上原点なので、上下が逆になります。

画像の配置は変換行列で指定します。

```js
var content = 'q\n' +
  dw + ' 0 0 ' + dh + ' ' + dx + ' ' + dy + ' cm\n' +
  '/Im0 Do\nQ';
```

`cm` は `[a b c d e f]` の行列で、`a` が横の拡大率、`d` が縦の拡大率、`e f` が平行移動です。画像は 1×1 の単位正方形として描かれるので、**拡大率にそのまま表示サイズ（pt）を入れれば**望むサイズになります。

`q` と `Q` は状態の保存と復元です。複数の画像を置くときに変換が累積しないよう、必ず囲みます。

単位はpt（1/72インチ）です。A4は `595.28 × 841.89`。

## xrefテーブルが鬼門

PDFの最後には、各オブジェクトが**ファイルの先頭から何バイト目にあるか**の一覧が要ります。

```
xref
0 7
0000000000 65535 f 
0000000015 00000 n 
0000000074 00000 n 
```

10桁ゼロ埋め、そのあとスペース、5桁の世代番号、スペース、`n` または `f`、そして**行末にスペースを1つ**。この末尾のスペースを忘れると壊れます（各エントリはちょうど20バイトである必要があります）。

なので、オブジェクトを書き出しながらバイト位置を記録していきます。

```js
var chunks = [], offsets = [0], pos = 0;

function put(data) {
  var u = typeof data === 'string' ? enc.encode(data) : data;
  chunks.push(u);
  pos += u.length;
}

function obj(num, body, stream) {
  offsets[num] = pos;          // ここで位置を控える
  put(num + ' 0 obj\n' + body + '\n');
  if (stream) { put('stream\n'); put(stream); put('\nendstream\n'); }
  put('endobj\n');
}
```

文字列とバイナリが混ざるので、`TextEncoder` で全部 `Uint8Array` に揃えてから連結します。**`String` に一度入れるとバイナリが壊れる**ので、そこだけ気をつけてください。

## 透過の扱い

PDFに埋め込むJPEGは透過を表現できません。透過PNGをそのまま `drawImage` すると、透明部分が黒く潰れます。

canvasの初期状態は透明なので、**先に白で塗ってから**描きます。

```js
x.fillStyle = '#fff';
x.fillRect(0, 0, c.width, c.height);
x.drawImage(img, 0, 0);
```

これは画像の縮小ツールでJPEG出力するときも同じ話で、忘れると背景が真っ黒なJPEGが量産されます。

## 検証方法

自作のPDFが「本当に正しいPDFか」は、ビューアが開けたかどうかでは判断しきれません。壊れていても表示されることがあります。

macOSなら Core Graphics に解釈させるのが確実でした。

```python
import Quartz
url = Quartz.CFURLCreateWithFileSystemPath(None, 'out.pdf', Quartz.kCFURLPOSIXPathStyle, False)
doc = Quartz.CGPDFDocumentCreateWithURL(url)
n = Quartz.CGPDFDocumentGetNumberOfPages(doc)
for i in range(1, n + 1):
    page = Quartz.CGPDFDocumentGetPage(doc, i)
    r = Quartz.CGPDFPageGetBoxRect(page, Quartz.kCGPDFMediaBox)
    print(i, r.size.width, r.size.height)
```

ページ数とMediaBoxが期待通りに取れれば、構造は正しく解釈されています。`qlmanage -t` でサムネイルを出せば、実際の描画も確認できます。

## できたもの

https://kotebako.com/image-to-pdf/

複数の画像をドロップして、順番を入れ替えて、A4かB5か実寸かを選んでPDFにします。全部ブラウザの中で動くので、画像はサーバーに送られません。身分証や契約書の写真のような、外部の変換サイトに上げたくないものを想定しています。

コード全体はここにあります。

https://github.com/kotebako/kotebako.github.io/blob/main/assets/topdf.js

PDF生成部分は120行ほどです。「PDFを作る」と聞くと身構えますが、**画像を並べるだけなら仕様のごく一部で足ります**。ライブラリを入れる前に、必要な範囲を見積もってみると案外自分で書けます。

