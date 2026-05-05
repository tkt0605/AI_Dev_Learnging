# ThreeBody — 開発の流れと設計思想の全体まとめ

> 対象リポジトリ: `threebody`  
> スタック: Vite + Vue 3 + TypeScript + Tailwind CSS

---

## 開発フェーズ一覧

| フェーズ | コミット概要 | 主な変更 |
|---|---|---|
| **Phase 0** | ThreeBodyの最初コミット | プロジェクト立ち上げ |
| **Phase 1** | エージェント・ゲートウェイ・CLIの構築 | バックエンド主体の設計 |
| **Phase 2** | サーバー構築・API構成・ディレクトリ変更 | バックエンド整理 |
| **Phase 3** | Vite + Vue.js + TypeScript 環境に切り替え | フロントエンド主体へ転換 |
| **Phase 4** | ベーシックなUIと機能アーキテクチャを設計・実装 | コンポーネント基盤構築 |
| **Phase 5** | 新規会話ボタン削除・設定/ビルドボタン追加 | UIの整理 |
| **Phase 6** | 設定ダイアログをDOMルートへ移動 | `<Teleport>` 採用 |
| **Phase 7** | 音声録音・表示の機能とUIを構築 | `useVoiceInput` 実装 |
| **Phase 8** | 三体問題の三角形UIをインデックスに設計・作成 | `NodeCanvas` 初期実装 |
| **Phase 9** | 三体問題UIの改善・D&D・アイコン追加 | `useTriangleNodes` 分離 |
| **Phase 10** | 設定ダイアログのMCPサーバ設定を削除 | UI簡素化 |

---

## Phase 3 — なぜバックエンドからフロントエンドに転換したか

**判断:** CLI・エージェント・ゲートウェイベースの設計から、Vite + Vue 3 + TypeScript のフロントエンド構成に全面切り替え。

**理由:**
- 初期設計はバックエンドAPIを中心に組んでいたが、AIチャット体験の核心は「UIのインタラクション品質」にあると判断
- ブラウザの Web Speech API・Web Audio API をフル活用するにはクライアントサイドが必然
- Vite によるホットリロード開発体験が、UI試行錯誤のサイクルを大幅に短縮する

---

## Phase 4 — ベーシックアーキテクチャの設計思想

### ディレクトリ構成

```
src/
├── components/      # UIコンポーネント（表示・操作の責務）
├── composables/     # 状態・ロジック（ビジネスロジックの責務）
├── types/           # 型定義
├── views/           # ルート単位のページ
└── router/          # Vue Router
```

**なぜ `composables/` と `components/` を分けたか:**  
Vue 3 の Composition API では、状態とロジックを composable として切り出せる。コンポーネントは「表示と操作の入口」に留め、状態管理は composable に集約することで、複数コンポーネントからの再利用と単体テストを可能にした。

---

### `types/message.ts` — マルチモーダル対応のブロック型設計

```ts
export type ContentBlock = TextBlock | ImageBlock | MapBlock | GameBlock
```

**なぜ `content: string` ではなく `blocks: ContentBlock[]` にしたか:**

将来的に会話の流れから画像・地図・ゲームが「具現化する」UIを目指しているため、メッセージを単なる文字列ではなくブロックの配列として定義した。将来 `type: 'map'` や `type: 'game'` のブロックを追加しても、レンダリング側は `v-if` でブロックタイプを判定するだけでよく、メッセージ型を壊さずに機能拡張できる。

```ts
// 現在は TextBlock のみ使用。将来のブロックタイプはここに追加するだけ
export type ImageBlock = { type: 'image'; url: string; alt?: string }
export type MapBlock   = { type: 'map'; lat: number; lng: number; zoom?: number }
export type GameBlock  = { type: 'game'; gameId: string }
```

---

## Phase 6 — 設定ダイアログを `<Teleport to="body">` に移動した理由

**問題:** 設定ダイアログが `AppAside`（サイドバー）の子として実装されていたため、`overflow: hidden` や `z-index` の重なり順がアサイドに依存してしまい、ダイアログが正しく前面に表示されなかった。

**解決:** `<Teleport to="body">` でダイアログをDOMの最上位に移管。

```html
<Teleport to="body">
  <dialog ref="dialogRef" class="fixed top-1/2 left-1/2 ...">
    ...
  </dialog>
</Teleport>
```

**なぜ `<dialog>` 要素を使ったか:**  
ブラウザネイティブの `<dialog>` は `showModal()` を呼ぶだけで `::backdrop` 擬似要素・フォーカストラップ・ESCキー閉じるが自動で機能する。独自実装より信頼性が高い。

---

### `SettingsDialog` の draft パターン

```ts
// 編集中は settings を直接変更しない — ローカルコピーで操作
const draft = reactive({
  language: settings.language,
  systemPrompt: settings.systemPrompt,
  mcpServers: settings.mcpServers.map(s => ({ ...s })),
})

function save() {
  settings.language = draft.language
  // ...
}
```

**なぜ:** キャンセル時に変更を破棄できるよう、`settings`（グローバル状態）とは独立したローカルコピーで編集する。保存ボタン押下時のみ本体に反映するパターン（Optimistic UI の逆で、Confirm-to-Commit パターン）。

---

## Phase 7 — 音声入力の設計思想（`useVoiceInput.ts`）

### 二つのAPIを組み合わせた理由

| API | 役割 |
|---|---|
| `Web Speech API` (`SpeechRecognition`) | 音声→テキスト変換 |
| `Web Audio API` (`AudioContext` + `AnalyserNode`) | マイク波形のリアルタイム可視化 |

**なぜ両方必要か:**  
`SpeechRecognition` は文字起こしのみ。波形バー表示（録音中フィードバック）には別途 `AudioContext` でマイクストリームを解析する必要がある。

### `continuous: true` + `onend` で再起動する理由

```ts
recognition.continuous = true
recognition.interimResults = true
recognition.onend = () => { if (recording.value) recognition?.start() }
```

ブラウザの `SpeechRecognition` は一定時間無音が続くと自動停止する。`onend` で再起動することで、ユーザーが明示的に停止するまで録音し続ける「連続音声入力」を実現した。

### `stop()` のリソース解放順序

```ts
function stop() {
  recording.value = false
  recognition?.abort()       // 1. 音声認識を中断
  cancelAnimationFrame(raf)  // 2. 波形描画ループを停止
  stream?.getTracks().forEach(t => t.stop())  // 3. マイクストリームを解放
  audioCtx?.close()          // 4. AudioContextを閉じる
  // 5. 最後にテキストを確定してコールバック
  const text = (finalText.value + interimText.value).trim()
  if (text) onFinish(text)
}
```

**なぜこの順序か:** `recognition.abort()` より先にストリームを止めると `onerror` が発火してエラー表示される。認識を先に止め、その後ハードウェアリソースを解放する。

---

## Phase 8・9 — 三体問題UIの設計思想

### 全体アーキテクチャ

```
AppAside（サイドバーパレット）
    ↓ HTML Drag & Drop API（featureId を dataTransfer で渡す）
NodeCanvas（SVGキャンバス）
    ↑↓ useTriangleNodes（モジュールスコープのシングルトン ref）
AppAside（isPlaced でリアクティブにUI更新）
```

---

### `useTriangleNodes.ts` — モジュールスコープシングルトンの理由

```ts
// composable 関数の外側（モジュールスコープ）に宣言
const placedNodes = ref<PlacedNode[]>([])

export function useTriangleNodes() {
  // どのコンポーネントが呼んでも同じ ref インスタンスを返す
  return { placedNodes, isPlaced, addNode, removeNode, updatePos }
}
```

**なぜ Pinia を使わなかったか:**
- `AppAside` と `NodeCanvas` は兄弟関係で共通の親が `App.vue` → `RouterView` のみ
- この規模でグローバルストアを導入するのは過剰設計
- composable のモジュールスコープに `ref` を置くだけで、Vue のリアクティビティを保ちつつ2コンポーネント間の状態共有が実現できる

---

### DnD を2系統に分けた理由

| 系統 | API | 用途 |
|---|---|---|
| **外部DnD** | HTML Drag & Drop API | サイドバー → キャンバスへの新規追加 |
| **内部ドラッグ** | Pointer Events | SVG内でのノード位置変更 |

HTML DnD API は `dragover` の座標がクライアント座標で来るため、SVG の `viewBox` 変換を行わないと正確な配置ができない。SVG内部の移動は Pointer Events + `getScreenCTM().inverse()` で変換することで、SVGがどのサイズに拡縮されていても正確な内部座標を取得できる。

```ts
function toSvg(cx: number, cy: number) {
  const pt = svg.createSVGPoint()
  pt.x = cx; pt.y = cy
  return pt.matrixTransform(svg.getScreenCTM()!.inverse())
}
```

---

### クリックとドラッグの区別（`dragMoved` フラグ）

```ts
let dragMoved = false

// 3px 以上動いたらドラッグと判定（手ブレ対策）
if (dx * dx + dy * dy > 9) dragMoved = true

function onPointerUp(id: DragId) {
  if (!dragMoved) triggerNode(id)  // 動いていなければクリック扱い
}
```

同一ノードで「クリック → ダイアログを開く / 録音開始」と「ドラッグ → 位置移動」を両立させるため。閾値を3px（`dx²+dy²>9`）にしたのは意図しない微小な手ブレをドラッグと誤判定しないため。

---

### `dragEnterCount` カウンターで isDragTarget を制御した理由

```ts
let dragEnterCount = 0
function onDragEnter() { dragEnterCount++ ... }
function onDragLeave() {
  if (--dragEnterCount <= 0) isDragTarget.value = false
}
```

`dragenter`/`dragleave` はDOM上の子要素をまたぐ度にも発火する。ブールフラグでは子要素通過時に `isDragTarget` がチカチカしてUIが不安定になるため、カウンターで「キャンバス全体から出た時だけ `false` にする」正確な判定を実現した。

---

## Phase 10 — MCP設定をコメントアウトした理由

設定ダイアログのMCPサーバー設定UIを削除（コメントアウト）。

**理由:** MCPサーバーの実際の接続・制御ロジックが未実装の段階でUIだけ存在すると、ユーザーが操作してもフィードバックがなく混乱を招く。UIは機能が動作するタイミングで復活させる判断をした。

---

## 設計全体を通じた一貫した判断軸

| 判断 | 採用したアプローチ | 避けたアプローチ |
|---|---|---|
| コンポーネント間の状態共有 | composable モジュールスコープシングルトン | Pinia（規模に対して過剰） |
| ダイアログの配置 | `<Teleport to="body">` + ネイティブ `<dialog>` | 独自モーダル実装 |
| 設定の編集フロー | draft コピー → Confirm-to-Commit | 即時反映（キャンセルできない） |
| SVG座標変換 | `getScreenCTM().inverse()` | 固定オフセット計算（スケール変化で壊れる） |
| メッセージ型 | `ContentBlock[]`（ユニオン型拡張） | `content: string`（将来の拡張が難しい） |
| クリック/ドラッグ区別 | `dragMoved` フラグ（3px閾値） | `click` イベントと `pointerdown` の分離（SVGでは難しい） |
| 音声入力の連続化 | `onend` で再起動 | `continuous: true` だけ（ブラウザが自動停止する） |
