# ThreeBody — Voice First オンボーディングとダイアログバグ修正

> 対象ファイル: `src/components/NodeCanvas.vue`

---

## 1. Voice First オンボーディング（案Cの実装）

### 何をしたか

`NodeCanvas.vue` に `onboarded` / `slotsSummoned` の2つの状態を追加し、  
初回ロード時にユーザーがVoiceノードをクリックするまでの「誘導演出」を実装した。

```ts
const onboarded = ref(false)      // Voice を一度でも押したか
const slotsSummoned = ref(false)  // 召喚アニメーション済みか

function triggerNode(id: DragId) {
  if (id === 'center') {
    if (!onboarded.value) {
      onboarded.value = true
      setTimeout(() => { slotsSummoned.value = true }, 150)
    }
    recording.value ? stop() : start()
  }
  // ...
}
```

### なぜこの設計にしたか

#### 背景：ゼロ状態の問題

改善前のキャンバスは「白い点が一つあるだけの暗い画面」だった。  
ヒントテキスト（`opacity: 0.18`）は実用的な視認性に欠け、ユーザーが何をすべきか分からない状態だった。

#### 採用した哲学：「全ての会話はVoiceを起点にする」

ThreeBodyの設計思想として **Voice（音声）がすべての会話の重心** である。  
Chat・MCP・Context の3機能は、Voiceが呼び起こす「可能性」として存在する。  
この哲学をインタラクションそのもので体験させるために、案Cを選んだ。

> 案A（脈動する重心）は視覚的だが、Voiceを押さなくても進める  
> 案B（三体軌道）は斬新だが「漂うノードを固定する」操作が伝わりにくい  
> 案C（Voice First）は「声を出すことで世界が始まる」という哲学をインタラクションで表現できる

#### パルスリング（3重の波紋）

```css
@keyframes pulse-ring {
  0%   { transform: scale(1);   opacity: 0.5; }
  100% { transform: scale(4.5); opacity: 0;   }
}
.pulse-ring-0 { animation: pulse-ring 2.4s ease-out infinite; animation-delay: 0s;   }
.pulse-ring-1 { animation: pulse-ring 2.4s ease-out infinite; animation-delay: 0.8s; }
.pulse-ring-2 { animation: pulse-ring 2.4s ease-out infinite; animation-delay: 1.6s; }
```

波紋を3本・0.8秒ずらしたのは「途切れなく広がり続ける」視覚効果のため。  
1本だと点滅に見え、2本だと間隔が空く。3本・等間隔が最も連続的に見える。

`transform-box: fill-box` + `transform-origin: center` を使ったのは、  
SVGの `transform` がデフォルトで原点(0,0)基準になるため、円の中心を起点にスケールするために必要。

#### 召喚アニメーション（スロットの出現）

```css
@keyframes slot-in {
  from { opacity: 0; transform: scale(0.3); }
  to   { opacity: 1; transform: scale(1);   }
}
.slot-summon   { animation: slot-in 0.65s cubic-bezier(0.34, 1.56, 0.64, 1) both; }
.slot-summon-0 { animation-delay: 0ms;   }
.slot-summon-1 { animation-delay: 130ms; }
.slot-summon-2 { animation-delay: 260ms; }
```

`cubic-bezier(0.34, 1.56, 0.64, 1)` はバネ感のあるイージング（オーバーシュートあり）。  
「召喚された」という演出に機械的なリニアではなく生命感のある動きを与えるため。  
時差（0ms / 130ms / 260ms）をつけたのは、3つが同時に出ると「ポップアップ」に見えるため。

#### スキップボタンの配置

```html
<button
  class="pointer-events-auto text-white/20 hover:text-white/45 ..."
  @click="skipOnboard"
>スキップ</button>
```

親div に `pointer-events-none` を設定しSVGのクリックを素通しにしつつ、  
スキップボタンだけ `pointer-events-auto` で復活させた。  
「声を出せない環境（電車・会議室）でも詰まらない」というUX上の配慮。

---

## 2. ダイアログが意図せず開くバグの修正

### 何が起きていたか

ノードを**削除した時**と**DnDでキャンバスに追加した時**に、  
そのノードのダイアログが開いてしまうバグがあった。

### 原因の分析

#### 削除ボタン（×）のケース

```html
<g
  @click.stop="removeNode(node.id)"
  @pointerdown.stop          <!-- ← pointerdown は止めている -->
                             <!-- ← pointerup は止めていない！ -->
>
```

イベントの発火順序：
1. `pointerdown` → `.stop` で伝播阻止 → `onPointerDown(node.id)` は呼ばれない → `dragging = null`
2. `pointerup` → `.stop` なし → 親 `<g>` にバブリング → `onPointerUp(node.id)` が呼ばれる
3. `click` → `.stop` → `removeNode` でノード削除

`onPointerUp` の時点では `dragging = null`、`dragMoved = false` なので  
「クリックした」と判定されて `triggerNode(id)` → ダイアログが開く。

#### DnDドロップのケース

HTML DnD は pointer event とは別の系統だが、ドロップ後にブラウザが  
synthetic `pointerup` を発火させることがある。  
新しく配置されたノードの上に発火すると、`dragging = null`・`dragMoved = false` のまま  
`onPointerUp` → `triggerNode` → ダイアログが開く。

### 修正内容

```ts
// Before
function onPointerUp(id: DragId) {
  const moved = dragMoved
  dragging.value = null
  if (!moved) triggerNode(id)  // dragging の状態を確認していない
}

// After
function onPointerUp(id: DragId) {
  const wasTracked = dragging.value === id  // このノードのpointerdownを受けたか確認
  const moved = dragMoved
  dragging.value = null
  if (wasTracked && !moved) triggerNode(id)  // 両方trueの時だけ発火
}
```

### なぜ `wasTracked` チェックで両方解決できるか

| 状況 | `dragging.value` | `wasTracked` | 結果 |
|---|---|---|---|
| 普通のクリック | `=== id`（`onPointerDown`が呼ばれた） | `true` | `triggerNode` → ダイアログ/録音 ✓ |
| 削除ボタン（×） | `null`（`pointerdown.stop`で阻止） | `false` | 何もしない ✓ |
| DnDドロップ後 | `null`（DnD中はpointerdownなし） | `false` | 何もしない ✓ |

`dragging.value` は `onPointerDown` の中でのみ設定される。  
つまり「`dragging.value === id` が真 ＝ このノードのpointerdownを正規に受けた」という保証になる。  
`@pointerup.stop` を削除ボタンに追加する方法もあるが、それでは**DnDケースは解決しない**。  
`wasTracked` による一元管理が最小変更で両方を解決する最適解。
