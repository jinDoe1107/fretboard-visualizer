# Fretglyph

ギター / ベースのスケール・コード可視化アプリ(フロントエンドのみ、Vue 3 + Vite)。

## セットアップ

```bash
npm install
npm run dev
```

ブラウザで `http://localhost:5173` を開いてください。

## ビルド

```bash
npm run build
```

`dist/` フォルダに静的ファイルが出力されます。Cloudflare Pages などにそのままデプロイできます。

- ビルドコマンド: `npm run build`
- 出力ディレクトリ: `dist`

## 主な機能

- 楽器選択: ギター(6弦) / ベース(4弦) / ベース(5弦)
- 表示パターン: スケール / コード
- スケール: 30種類(基本・ペンタ・チャーチモード・ジャズ系・シンメトリック・ビバップ・和風など)
- コード: メジャー / マイナー / sus2 / sus4 / dim / aug / パワーコード + テンション複数選択
- ラベル切替: 音名 / 度数
- 24フレット対応
- 度数による色分け(ルート:赤 / 3度:青 / 5度:緑 / その他:琥珀)
- Marshallアンプ風のビジュアルデザイン

## ファイル構成

```
fretboard-visualizer/
├── index.html
├── package.json
├── vite.config.js
└── src/
    ├── main.js       # エントリーポイント
    └── App.vue       # アプリ本体(ロジック + UI)
```

## 技術スタック

- Vue 3 (Composition API / `<script setup>`)
- Vite 5
- SVGによるフレットボード描画(外部ライブラリ不使用)
