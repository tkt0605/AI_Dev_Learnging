# ThreeBody — 三角形UI の設計思想

> 対象ファイル:
> - `src/composables/useTriangleNodes.ts`
> - `src/components/AppAside.vue`
> - `src/components/NodeCanvas.vue`

---

## 概要

三体問題UIは「サイドバーのパレットから機能ノードをドラッグ&ドロップし、SVGキャンバス上の三角形に配置する」インターフェース。  
設計上の核心は **Pinia（グローバルストア）不使用・prop drilling不使用** で2コンポーネント間の共有状態を実現したことにある。

---

## 設計判断の詳細

### 1. `useTriangleNodes` をモジュールレベルのシングルトンにした

```ts
// composable の外側（モジュールスコープ）に宣言
const placedNodes = ref<PlacedNode[]>([])

export function useTriangleNodes() {
  // どのコンポーネントが呼んでも同じ ref を参照する
  return { placedNodes, isPlaced, addNode, removeNode, updatePos }
}
```

**なぜ:** `AppAside`（サイドバー）と `NodeCanvas`（SVGキャンバス）は親子関係になく、props で状態を渡せない。  
Pinia を導入するほど大規模でもないため、composable のモジュールスコープに `ref` を置くことでシングルトン化し、どちらのコンポーネントが `useTriangleNodes()` を呼んでも同じリアクティブ状態を共有できるようにした。

---

### 2. DnD を2系統に分けた

| 系統 | 使用API | 用途 |
|---|---|---|
| **外部DnD** | HTML Drag & Drop API | サイドバーパレット → キャンバスへ（ノード追加） |
| **内部ドラッグ** | PointerEvents | SVG内でノードの位置を移動 |

**なぜ:** HTML DnD はブラウザ標準でクロス要素の転送に強いが、SVG内の正確な座標制御ができない。  
SVG内の移動だけは PointerEvents で管理し、`toSvg()` でクライアント座標を SVG viewBox 座標に変換している。

```ts
function toSvg(cx: number, cy: number) {
  const svg = svgRef.value
  if (!svg) return { x: cx, y: cy }
  const pt = svg.createSVGPoint()
  pt.x = cx; pt.y = cy
  const ctm = svg.getScreenCTM()
  if (!ctm) return { x: cx, y: cy }
  const r = pt.matrixTransform(ctm.inverse())
  return { x: r.x, y: r.y }
}
```

`getScreenCTM().inverse()` を使うことで、SVGが拡縮・平行移動されていても常に正確な内部座標を取得できる。

---

### 3. クリックとドラッグを同じノードで区別した

```ts
let dragMoved = false

function onPointerMove(e: PointerEvent) {
  const dx = e.clientX - dragOrigin.x
  const dy = e.clientY - dragOrigin.y
  if (dx * dx + dy * dy > 9) dragMoved = true  // 3px 以上動いたらドラッグ
  ...
}

function onPointerUp(id: DragId) {
  const moved = dragMoved
  dragging.value = null
  if (!moved) triggerNode(id)  // 動いていなければクリック扱い → ダイアログを開く
}
```

**なぜ:** 同じノードで「クリック → ダイアログを開く」と「ドラッグ → 位置を変える」の両方を成立させるため。  
`mousedown/mouseup` だけでは区別できないので `dragMoved` フラグで仕分けた。  
閾値を `dx*dx + dy*dy > 9`（半径3px）にしたのは、意図しない微小な手ブレをドラッグと誤判定しないようにするため。

---

### 4. `dragEnterCount` をカウンターにした

```ts
let dragEnterCount = 0

function onDragEnter() {
  dragEnterCount++
  isDragTarget.value = true
}

function onDragLeave() {
  dragEnterCount--
  if (dragEnterCount <= 0) {
    dragEnterCount = 0
    isDragTarget.value = false
  }
}
```

**なぜ:** `dragenter`/`dragleave` はDOM上の **子要素をまたぐ度にも発火する**。  
ブールフラグだと、マウスが子要素上を通過するたびに `isDragTarget` がチカチカしてUIが不安定になる。  
カウンターにすることで「キャンバス全体を出た時だけ `false` にする」という正確な判定ができる。

---

### 5. `isPlaced` でサイドバー側のドラッグを無効化した

```html
<div
  :draggable="!isPlaced(feat.id)"
  @dragstart="onDragStart($event, feat.id)"
>
```

**なぜ:** `placedNodes` がシングルトンなので、サイドバーもキャンバスの状態をリアクティブに把握できる。  
すでに配置済みのノードはドラッグ不可にし、チェックマークアイコンに切り替えることで「このノードは配置済み」をUIで伝える。

---

## アーキテクチャ全体図

```
AppAside（パレット）
    ↓ HTML DnD（featureId を dataTransfer で渡す）
NodeCanvas（SVGキャンバス）
    ↑↓ useTriangleNodes（モジュールシングルトンの ref）
AppAside（isPlaced でリアクティブにUI更新）
```

---

## まとめ

| 判断 | 理由 |
|---|---|
| モジュールスコープシングルトン | Pinia不使用・prop drilling不使用で2コンポーネント間の状態共有 |
| DnD 2系統 | HTML DnD は座標精度が低いため、SVG内はPointerEventsを使用 |
| `dragMoved` フラグ | クリックとドラッグを同一ノードで両立 |
| `dragEnterCount` カウンター | 子要素通過時の誤発火を防ぐ |
| `isPlaced` + `:draggable` | シングルトン状態をそのままドラッグ制御に利用 |
