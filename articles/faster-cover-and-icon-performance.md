---
title: "astro-notion-blogのヘッダー画像とアイコンの読み込みを速くする"
emoji: "🖼️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [astro-notion-blog, SVG]
published: false
---

## ヘッダー画像とあひるアイコンの読み込みを速くしました

astro-notion-blogをいじって[ブログサイト](https://www.swimu.net/)を作っています。

ファーストビューに出るヘッダー画像とあひるアイコンの読み込みが遅かったので、いくつか手を入れて改善しました。

[TODO: 改善後の画面の動画を追加]

| 項目                                   | Before  | After      |
| -------------------------------------- | ------- | ---------- |
| カバー画像のロード時間                 | 827 ms  | **113 ms** |
| サイトのあひるアイコンのファイルサイズ | 1.04 MB | **9.8 KB** |

## 起きていたこと

ページを開くと、ヘッダーのカバー画像が表示されるまでに毎回1 秒近くかかっていて、ファーストビューが明らかにガタついていました。

[TODO: 改善前の画面の動画を追加]

DevToolsのNetworkタブで計測してみると、こんな状態でした。

- カバー画像: 827 ms / 714 KB
- サイトのあひるアイコン (`favicon.svg`): 1040 ms / 1.04 MB

サイトのあひるアイコンのSVGが1 MBあるのは明らかにおかしいのですが、まずはカバー画像のほうから手をつけていきました。

## 問題1: ファーストビュー画像に`loading="lazy"`が付いていた

最初に気づいたのは、必ず初期画面に出るカバー画像に`loading="lazy"`が付いていたことでした。

```html
<img src="{coverImageURL}" alt="Site cover image" loading="lazy" />
```

`loading="lazy"`は「ビューポート外の画像を遅延ロードする」ための属性なので、必ず最初に表示される画像に付けるのは逆効果になります。代わりに`fetchpriority="high"`を付与して優先度を上げました。

```html
<img src="{coverImageURL}" alt="Site cover image" fetchpriority="high" />
```

### `<head>`にpreloadを追加する

さらにHTMLの`<head>`でプリロードを宣言し、ブラウザがパースを始めた瞬間に画像のダウンロードを開始させるようにしました。

```html
<link rel="preload" as="image" href="{coverImageURL}" fetchpriority="high" />
```

これだけで**827 ms → 113 ms**まで一気に縮みました。ファイルサイズは変えていないので、開始タイミングを早めるだけでこれだけ効くということになります。`<img>`タグまでHTMLのパースが進むのを待つ時間が、それだけ大きかったということでもあります。

## 問題2: SSRでリクエストごとにNotion APIを叩いていた

`Layout.astro`の元実装はこうなっていました。

```astro
const [database, tags] = await Promise.all([getDatabase(), getAllTags()])

if (database.Cover.Type === 'file') {
  coverImageURL = filePath(new URL(database.Cover.Url))
}
```

このサイトは`output: 'server'`でSSRしているので、リクエストのたびに`getDatabase()`が走り、コールドスタート時にはNotion APIを叩いていました。カバー画像のURLを取るためだけに毎回Notionを呼ぶのは無駄です。

### ビルド時に解決済みパスをTS定数に書き出す

ビルド時に動くAstro integrationのなかで、解決済みのローカルパスをTS定数ファイルに書き出すようにしました。

```ts
// src/integrations/cover-image-downloader.ts
const writeGeneratedFile = (url: string) => {
  fs.writeFileSync(
    GENERATED_FILE,
    `export const COVER_IMAGE_URL: string = ${JSON.stringify(url)}\n`,
  );
};

// astro:build:start フック内で
writeGeneratedFile(filePath(url));
return downloadFile(url);
```

`Layout.astro`側は単にimportするだけです。

```astro
import { COVER_IMAGE_URL } from '../generated/cover-image.ts'
const coverImageURL = COVER_IMAGE_URL
```

ViteがSSRバンドル時にインライン化してくれるので、ランタイムでNotionを一切叩かない構成になりました。

### 「NotionのURLは1時間で期限切れするのでは？」

最初に気になったのがここでした。NotionのCover URLは署名付きS3 URLで、署名部分は1時間で期限切れになります。

ただ、保存しているのは`pathname`の部分だけです。

```ts
export const filePath = (url: URL): string => {
  const [dir, filename] = url.pathname.split("/").slice(-2);
  return pathJoin(BASE_PATH, `/notion/${dir}/${filename}`);
};
```

ローカルパス`/notion/{uuid}/{filename}`はファイルIDベースで安定しているので、署名付きURLは「ビルド時に数秒間fetchするため」だけの使い捨てになります。ブラウザは一度もNotionのS3を見にいきません。

## 問題3: サイトのあひるアイコンが1 MBあった

カバー画像が解決した後、Networkタブで残っていたのが`favicon.svg`でした。1.04 MB / 1040 msかかっていました。

```bash
$ ls -la public/favicon.svg
-rw-r--r--  1 user  staff  1088910  favicon.svg
```

中身を覗いてみると、こんな感じです。

```bash
$ grep -c "base64" public/favicon.svg
1
```

**SVGの中にPNGがbase64で埋め込まれていました**。IllustratorやSVG生成ツールから書き出すとよく出てくる罠で、SVGの体裁ですが中身はラスター画像をXMLでくるんだだけのファイルです。

しかもこのfaviconは、70 pxのサイトのあひるアイコンと57 pxのプロフィールアイコンの`<img>`から参照されていました。70 pxの表示のために、毎回1 MBをダウンロードしている状態だったわけです。

### `<img>`の参照を既存のPNGに差し替える

あひるアイコンは、別途小さなPNG形式でも用意していました。

```
favicon-96x96.png   9.8 KB
```

`<img>`の参照だけを差し替えます。

```diff
- <img src={getStaticFilePath('/favicon.svg')} alt="Site icon" ... />
+ <img src={getStaticFilePath('/favicon-96x96.png')} alt="Site icon" ... />
```

ブラウザタブ用の`<link rel="icon" href="/favicon.svg">`は残しましたが、これはバックグラウンドでブラウザが遅延ロードするだけなので描画には影響しません。

これで**1.04 MB → 9.8 KB（約110 倍縮小）**になりました。

## 学んだこと

今回の作業で気づいたことをいくつか書いておきます。

- `loading="lazy"`はファーストビュー外の画像のためのもので、必ず最初に表示される画像に付けるのは逆効果になります
- preloadはファーストビュー画像にとても効きます。サイズを変えず、タイミングだけで700 ms縮められました
- SVGが異常に大きいときは中身を疑ったほうがよさそうです。base64埋め込みはSVGの利点（軽量・スケーラブル）を失っています
- SSRで毎回APIを叩いているなら、ビルド時に解決できる情報はビルド時に閉じ込めるとコールドスタートが速くなります

## まとめ

ファーストビューの画像まわりに対して、

- `loading="lazy"`を外して`fetchpriority="high"`と`<link rel="preload">`を付ける
- ビルド時に解決済みパスをTS定数に書き出して、SSRでNotion APIを呼ばないようにする
- 重い`favicon.svg`ではなく既存の小さなPNGを`<img>`から参照する

という対応をしました。

ファーストビューでカクつく感じが消えて、サイトを開くときのもっさり感もなくなりました。

さらにいい感じなブログサイトに改修でき、うれしいです。
