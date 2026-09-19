# サンプル設計: ヘッダーのロゴリンク先を修正する

## 決定事項
ヘッダーのロゴ画像のリンク先が `/home` のままになっている。トップページのパスは
`/` に変更済みなので、ロゴのリンク先も `/` に修正する。表示・スタイルは変更しない。

## 対象ファイル
- `src/components/Header.tsx`

## 完了条件
- `src/components/Header.tsx` 内のロゴの `href` が `/home` から `/` になっている
- `grep -rn "href=\"/home\"" src/components/Header.tsx` が0件
- `npx tsc --noEmit` がエラー0
- `npm run build` が成功する

## 触ってはいけない領域
- `Header.tsx` 内のロゴ以外の要素（ナビゲーションリンク、スタイル定義）
- `src/components/Header.tsx` 以外のファイル
