# ThreeBody — 開発ロードマップ（2026-05-09 時点）

## 現状サマリー

### 実装済み

| 領域 | 状態 | 備考 |
|------|------|------|
| Vue 3 + Vite フロントエンド | ✅ 稼働中 | TypeScript, Tailwind CSS |
| 三角形ノードUI（NodeCanvas） | ✅ 完成 | ドラッグ&ドロップ、アニメーション |
| 音声入力（VoiceInput） | ✅ 完成 | Web Speech API |
| チャットダイアログ（ChatDialog） | ⚠️ UIのみ | バックエンド未接続、モック応答 |
| MCPダイアログ | ✅ UIのみ | 実機能未実装 |
| コンテキストダイアログ | ✅ UIのみ | 実機能未実装 |
| 設定ダイアログ（思考レベル1〜5 / 言語 / システムプロンプト） | ✅ 完成 | ローカル状態のみ、永続化なし |
| Supabaseユーザー認証 | ✅ 完成 | ログイン / サインアップ / OAuth コールバック |
| Expressバックエンド（server.ts） | ✅ 稼働中 | SSEストリーミング対応 |
| AIプロバイダー統合 | ✅ 完成 | Anthropic / OpenAI / DeepSeek / Ollama |
| 思考レベルモデルマッピング（Lv.1〜5） | ✅ 完成 | トークン数・拡張思考バジェット制御 |
| Ollamaローカルモデル（qwen2.5） | ✅ 動作確認済 | localhost:11434 |

### 未実装（コアギャップ）

- **useChat.ts** が `simulateStream()` モック関数を使用 → バックエンド `/api/chat` に未接続
- 設定変更（プロバイダー選択、思考レベル）がチャットAPIに反映されない
- 設定のlocalStorage永続化なし
- マルチモーダル出力（画像・地図・ゲーム）なし
- MCP実機能なし

---

## フェーズ別ロードマップ

### Phase 1 — バックエンド接続（最優先）
**目標:** useChat.ts がモックを脱却し、実際のAIと会話できる状態にする

- [ ] **1-1** `useChat.ts` の `simulateStream` を削除し、`/api/chat` SSEエンドポイントに接続
  - `EventSource` または `fetch` + ReadableStream で SSE 受信
  - `messages` をAnthropicフォーマット `{role, content}[]` に変換して送信
  - ストリーミングデルタをリアルタイムで `MessageBubble` に反映
- [ ] **1-2** `useSettings.ts` のプロバイダー・思考レベル・システムプロンプトをリクエストボディに含める
- [ ] **1-3** エラーハンドリング（ネットワーク断、API制限）をUIに表示
- [ ] **1-4** ローカルストレージへの設定永続化（`useSettings.ts` に `localStorage` ラッパー追加）

**完了条件:** ChatDialogで入力したメッセージがOllamaモデルにストリーミング送信され、応答がリアルタイム表示される

---

### Phase 2 — 音声→AI会話の完全連結
**目標:** マイクボタンを押して話すだけで、AIが音声テキストに応答する体験の完成

- [ ] **2-1** `useVoiceInput` の `finalText` → `useChat.sendMessage` への自動転送を確認・修正
- [ ] **2-2** AI応答のText-to-Speech（Web Speech Synthesis API または外部TTS）
- [ ] **2-3** 音声認識中の視覚フィードバック改善（バー高さと音量の連動精度向上）
- [ ] **2-4** 多言語対応（設定の言語切り替え → `SpeechRecognition.lang` に反映）

**完了条件:** 話す → AIが返答する（音声オプション付き）の1サイクルが完結

---

### Phase 3 — マルチモーダル出現（三体パラダイムの核心）
**目標:** 会話の文脈からリッチメディアが「具現化」する体験

#### 3-1 画像生成
- [ ] バックエンドに `/api/image` エンドポイント追加（DALL-E 3 / Stable Diffusion）
- [ ] AIの応答テキストから画像生成トリガーを検出（キーワード / ツールコール）
- [ ] `NodeCanvas` に画像ノードを追加（SVG内にforeignObjectで表示）
- [ ] 生成画像をドラッグ可能な浮動パネルとして配置

#### 3-2 地図表示
- [ ] Mapbox GL JS または Leaflet を統合
- [ ] 会話中の地名・座標を抽出してマップノードを召喚
- [ ] チャットとマップの双方向インタラクション（地名クリック→チャット詳細聞き直し）

#### 3-3 インタラクティブゲーム
- [ ] 会話からゲームルール・テーマを抽出
- [ ] Canvas/WebGL を用いたミニゲームコンポーネント（クイズ、シミュレーションなど）
- [ ] ゲーム結果を会話に返す閉ループ設計

**完了条件:** 「〇〇の画像を見せて」「東京の地図を出して」と言うと、ノードキャンバスにメディアが出現する

---

### Phase 4 — MCP（Model Context Protocol）実装
**目標:** 外部ツール・データソースとAIを繋ぐ拡張レイヤー

- [ ] MCPサーバー接続フレームワーク（`McpDialog` の実機能化）
- [ ] ファイルシステムMCP（ローカルファイル読み込み・検索）
- [ ] Web検索MCP（Brave Search / SerpAPI）
- [ ] データベースMCP（Supabase テーブル参照）
- [ ] ツール呼び出し結果をチャット会話に透過的に組み込む

---

### Phase 5 — UX/パフォーマンス磨き込み
**目標:** Apple製品水準の体験品質

- [ ] **5-1** メッセージ履歴の永続化（Supabase `messages` テーブル）
- [ ] **5-2** セッション管理（会話履歴を複数セッションで保持・切り替え）
- [ ] **5-3** `NodeCanvas` のノード位置をセッション間で保存
- [ ] **5-4** レスポンシブデザイン（モバイル対応）
- [ ] **5-5** PWA化（オフライン対応、ホーム画面追加）
- [ ] **5-6** キーボードショートカット全面実装
- [ ] **5-7** アニメーションの60fps安定化（SVGレンダリング最適化）

---

### Phase 6 — 本番デプロイ
**目標:** インターネット公開可能な状態

- [ ] **6-1** フロントエンド: Vercel / Cloudflare Pages へデプロイ
- [ ] **6-2** バックエンド: Railway / Fly.io / Render でコンテナ化
- [ ] **6-3** 環境変数管理（APIキーをサーバーサイドに完全隔離）
- [ ] **6-4** レート制限・認証ミドルウェアの追加
- [ ] **6-5** Supabase RLS（Row Level Security）でユーザーデータ分離
- [ ] **6-6** ドメイン取得・SSL設定

---

## 技術負債リスト

| 項目 | 優先度 | 内容 |
|------|--------|------|
| useChat.ts モック削除 | 🔴 最高 | Phase 1で対応 |
| APIキーのフロント露出リスク | 🔴 最高 | 現在バックエンド経由だが `.env` 管理の徹底 |
| SettingsDialog の MCP セクションがコメントアウト | 🟡 中 | Phase 4で復活 |
| `useSettings.ts` 永続化なし | 🟡 中 | Phase 1-4で対応 |
| `McpPanel.vue` / `McpDialog.vue` の中身が空 | 🟠 高 | Phase 4で対応 |
| チャット履歴がページリロードで消滅 | 🟠 高 | Phase 5-1で対応 |
| Ollamaモデルがqwen2.5:0.5b固定（全レベル） | 🟡 中 | より高性能なローカルモデルに更新 |

---

## 今すぐ着手すべき最小アクション（Next Step）

```
Phase 1-1: useChat.ts のバックエンド接続
```

```typescript
// useChat.ts 変更の骨格
async function sendMessage(text: string) {
  // ユーザーメッセージを追加
  // assistantメッセージを空で追加（streaming: true）
  
  const response = await fetch('http://localhost:3000/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      messages: toAnthropicFormat(messages.value),
      provider: settings.provider,
      thinkingLevel: settings.thinkingLevel,
      systemPrompt: settings.systemPrompt,
    }),
  })
  
  // SSE読み取りループ
  const reader = response.body!.getReader()
  // ...デルタをassistantメッセージに追記
}
```

このステップが完了すると「話す/タイプする → AIが実際に応答する」という三体の最小動作ループが成立し、以降の全フェーズの土台になる。
