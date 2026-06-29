# エアホッケーゲーム 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 単一HTMLファイルで2台のiPad対戦エアホッケーゲームを実装する（PeerJS P2P通信、Canvas描画、バニラJS）

**Architecture:** ホスト権威モデル。ホストのみが物理演算を実行し、毎フレームゲスト側へ状態を送信。ゲストはマレット座標のみを送信し、受信した状態を描画。すべてのUI状態は接続→対戦→結果の3画面をDOMコンテナの表示切替で管理。

**Tech Stack:** HTML5 + CSS3 + Canvas API + Vanilla JS + PeerJS 1.5.4 (CDN) + Google Fonts "M PLUS Rounded 1c"

## Global Constraints

- 単一HTMLファイル（`index.html`）に HTML + CSS + JS をすべて収める
- 外部依存は PeerJS のみ（`<script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>`）
- フレームワーク不使用（バニラJS）、ビルド工程なし
- 対象ブラウザ: iPad Safari 最新2バージョン
- 横持ち（landscape）前提。縦持ちなら「横にしてね」オーバーレイを表示
- 台・パック・マレットは Canvas に描画。スコア・ボタン・ID表示は Canvas 外の DOM
- 座標は 0〜1 に正規化して送受信
- `touch-action: none` でブラウザスクロール・ズームを抑止
- チューニング用定数は JS の先頭に `const CONFIG = {...}` でまとめる
- GitHub Pages（https）配信前提

---

## ファイル構成

```
index.html   ← 唯一のファイル。すべてインライン。
```

### index.html 内部構成
```
<head>
  Google Fonts import
  <style>
    CSS変数・リセット・アニメーション定義
    画面コンテナのレイアウト
    接続画面・結果画面のコンポーネントスタイル
  </style>
</head>
<body>
  #screen-orient   ← 縦持ちオーバーレイ
  #screen-connect  ← 接続画面（4ステップ）
  #screen-game     ← 対戦画面（Canvas + スコアバー）
  #screen-result   ← 結果画面

  <script src="PeerJS CDN">
  <script>
    // 1. CONFIG定数
    // 2. State管理
    // 3. DOM参照
    // 4. 画面切替
    // 5. Canvas描画
    // 6. 物理エンジン（ホストのみ実行）
    // 7. タッチ操作
    // 8. PeerJS通信
    // 9. ゲームループ
    // 10. アニメーション制御
  </script>
</body>
```

---

### Task 1: HTML骨格 + CSSファウンデーション

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: `#screen-orient`, `#screen-connect`, `#screen-game`, `#screen-result` の4コンテナ。それぞれ `display:none` でデフォルト非表示、`.active` クラスで表示

- [ ] **Step 1: index.html を作成してHTML骨格を書く**

```html
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black">
<title>エアホッケー</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c:wght@500;700;800;900&display=swap" rel="stylesheet">
<style>
/* === RESET & BASE === */
*, *::before, *::after { box-sizing: border-box; }
html, body {
  margin: 0; padding: 0;
  width: 100%; height: 100%;
  overflow: hidden;
  font-family: 'M PLUS Rounded 1c', system-ui, sans-serif;
  -webkit-font-smoothing: antialiased;
  background: #1a2238;
  touch-action: none;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}

/* === CSS DESIGN TOKENS === */
:root {
  --c-bg: #fef9f0;
  --c-bg-success: #f0faf6;
  --c-ink: #1a2238;
  --c-blue: #4ea8de;
  --c-blue-light: #bfe0f5;
  --c-red: #ff6b5e;
  --c-red-light: #ffd4cd;
  --c-mint: #3fc7a3;
  --c-yellow: #ffc233;
  --c-goal-blue: #bfe0f5;
  --c-goal-red: #ffc9c2;
  --c-goal-flash: #ffe39a;
  --c-wood-top: #cda473;
  --c-wood-bot: #bb874f;
}

/* === ANIMATIONS === */
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes starspin { to { transform: rotate(360deg); } }
@keyframes hop {
  0%, 100% { transform: translateY(0); }
  45% { transform: translateY(-16px); }
}
@keyframes fall {
  0% { transform: translateY(0) rotate(0deg); }
  100% { transform: translateY(110vh) rotate(540deg); }
}
@keyframes pop {
  0% { transform: scale(0); }
  60% { transform: scale(1.25); }
  100% { transform: scale(1); }
}
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-8px); }
}
@keyframes popbadge {
  0% { transform: scale(0.9) rotate(-10deg); }
  100% { transform: scale(1.12) rotate(6deg); }
}
@keyframes ping {
  0% { transform: scale(0.4); opacity: 0.9; }
  100% { transform: scale(1.8); opacity: 0; }
}

/* === SCREEN CONTAINERS === */
.screen {
  position: fixed;
  inset: 0;
  display: none;
  align-items: center;
  justify-content: center;
}
.screen.active { display: flex; }

/* 縦持ちオーバーレイ */
#screen-orient {
  z-index: 9999;
  background: var(--c-ink);
  flex-direction: column;
  gap: 20px;
  color: #fff;
  font-size: 22px;
  font-weight: 800;
  text-align: center;
  padding: 40px;
}
#screen-orient .rotate-icon {
  font-size: 64px;
  animation: spin 2s linear infinite;
  display: inline-block;
}

/* === CONNECT SCREEN === */
#screen-connect {
  background: #1a2238;
}
.connect-card {
  width: 500px;
  max-width: 96vw;
  max-height: 96vh;
  background: var(--c-bg);
  border: 4px solid var(--c-ink);
  border-radius: 34px;
  box-shadow: 0 14px 34px rgba(26,34,56,.18);
  overflow: hidden;
  position: relative;
  display: flex;
  flex-direction: column;
}

/* connect steps */
.connect-step { display: none; flex-direction: column; flex: 1; }
.connect-step.active { display: flex; }

/* 2a logo area */
.connect-logo {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  padding: 22px 0 6px;
}
.connect-logo-icons {
  display: flex;
  align-items: center;
  gap: 9px;
}
.connect-logo-title {
  font: 800 16px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  opacity: 0.7;
  letter-spacing: 2px;
}
/* 2a buttons */
.connect-btns {
  flex: 1;
  display: flex;
  gap: 18px;
  padding: 6px 26px 26px;
  align-items: stretch;
}
.btn-role {
  flex: 1;
  border: 4px solid var(--c-ink);
  border-radius: 28px;
  box-shadow: 0 7px 0 var(--c-ink);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 14px;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
  transition: transform 0.1s;
}
.btn-role:active { transform: translateY(4px); box-shadow: 0 3px 0 var(--c-ink); }
.btn-role-host { background: var(--c-mint); }
.btn-role-guest { background: var(--c-yellow); }
.btn-role-label {
  font: 900 24px/1.15 'M PLUS Rounded 1c';
  color: var(--c-ink);
  text-align: center;
}

/* 2b host waiting - inner layout */
.connect-host-inner {
  flex: 1;
  display: flex;
  flex-direction: row;
  align-items: center;
}
.connect-code-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  padding: 0 10px;
}
.connect-code-label {
  font: 800 16px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  opacity: 0.6;
  letter-spacing: 2px;
}
.connect-code-tiles {
  display: flex;
  gap: 9px;
}
.code-tile {
  width: 48px; height: 64px;
  background: #fff;
  border: 4px solid var(--c-ink);
  border-radius: 14px;
  box-shadow: 0 4px 0 var(--c-ink);
  display: flex;
  align-items: center;
  justify-content: center;
  font: 900 38px 'M PLUS Rounded 1c';
}
.code-tile:nth-child(1) { color: var(--c-blue); }
.code-tile:nth-child(2) { color: var(--c-red); }
.code-tile:nth-child(3) { color: var(--c-mint); }
.code-tile:nth-child(4) { color: var(--c-yellow); }

.connect-spinner-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  padding: 0 10px;
}
.spinner-wrap {
  position: relative;
  width: 118px; height: 118px;
}
.spinner-ring {
  position: absolute;
  inset: 0;
  border: 11px solid rgba(26,34,56,.14);
  border-top: 11px solid var(--c-mint);
  border-radius: 50%;
  animation: spin 1.1s linear infinite;
}
.spinner-char {
  position: absolute;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  width: 60px; height: 60px;
}
.connect-wait-label {
  font: 800 18px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  opacity: 0.6;
}

/* 2c guest input */
.connect-guest-inner {
  flex: 1;
  display: flex;
  flex-direction: row;
}
.connect-slots-panel {
  flex: 0.9;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 0 8px;
}
.connect-slots {
  display: flex;
  gap: 8px;
}
.input-slot {
  width: 42px; height: 56px;
  background: #fff;
  border: 4px solid var(--c-ink);
  border-radius: 13px;
  box-shadow: 0 4px 0 var(--c-ink);
  display: flex;
  align-items: center;
  justify-content: center;
  font: 900 32px 'M PLUS Rounded 1c';
  color: var(--c-ink);
}
.input-slot-empty::after {
  content: '';
  display: block;
  width: 13px; height: 13px;
  border-radius: 50%;
  background: var(--c-ink);
  opacity: 0.18;
}
.connect-keypad {
  flex: 1.1;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 9px;
  padding: 18px 24px 18px 0;
  align-content: center;
}
.key {
  background: #fff;
  border: 4px solid var(--c-ink);
  border-radius: 16px;
  box-shadow: 0 4px 0 var(--c-ink);
  display: flex;
  align-items: center;
  justify-content: center;
  font: 900 28px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  height: 52px;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
  transition: transform 0.08s;
}
.key:active { transform: translateY(3px); box-shadow: 0 1px 0 var(--c-ink); }
.key-del { background: var(--c-red-light); }
.key-ok { background: var(--c-mint); }

/* 2d connected */
.connect-success-inner {
  flex: 1;
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
  gap: 30px;
  position: relative;
  background: var(--c-bg-success);
}
.connect-success-check {
  position: relative;
  width: 140px; height: 140px;
  animation: hop 1.3s ease-in-out infinite;
}
.connect-success-text {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 18px;
}
.connect-success-title {
  font: 900 28px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  letter-spacing: 2px;
}
.connect-success-chars {
  display: flex;
  gap: 16px;
}
.sparkle {
  position: absolute;
  clip-path: polygon(50% 0%,61% 35%,98% 35%,68% 57%,79% 91%,50% 70%,21% 91%,32% 57%,2% 35%,39% 35%);
}

/* === GAME SCREEN === */
#screen-game {
  background: var(--c-ink);
  flex-direction: column;
  align-items: stretch;
  padding: 0;
}
#game-score-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 20px 4px;
  flex-shrink: 0;
}
.score-left, .score-right {
  display: flex;
  align-items: center;
  gap: 12px;
}
.score-num {
  font: 900 48px/1 'M PLUS Rounded 1c';
}
.score-num-blue { color: var(--c-blue); }
.score-num-red { color: var(--c-red); }
.score-num.scoring { animation: popbadge 0.6s ease-in-out infinite alternate; font-size: 58px; }
.score-num.dimmed { opacity: 0.55; }

#btn-pause {
  width: 52px; height: 52px;
  border-radius: 16px;
  background: #fff;
  border: 3px solid var(--c-ink);
  box-shadow: 0 4px 0 var(--c-ink);
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
}
.pause-bar {
  width: 8px; height: 21px;
  border-radius: 4px;
  background: var(--c-ink);
}

#game-canvas-wrap {
  flex: 1;
  position: relative;
  min-height: 0;
  padding: 4px 12px 12px;
}
#game-canvas {
  display: block;
  width: 100%;
  height: 100%;
  border-radius: 24px;
}

/* ゴール演出オーバーレイ（Canvas上） */
#goal-overlay {
  position: absolute;
  inset: 4px 12px 12px;
  pointer-events: none;
  z-index: 5;
  display: none;
}
#goal-overlay.active { display: block; }

/* ping ring */
.ping-ring {
  position: absolute;
  border-radius: 50%;
  border: 5px solid var(--c-yellow);
  animation: ping 1s ease-out infinite;
}
/* +1 badge */
.plus-one {
  position: absolute;
  background: var(--c-yellow);
  border: 3px solid var(--c-ink);
  border-radius: 50%;
  width: 58px; height: 58px;
  display: flex;
  align-items: center;
  justify-content: center;
  font: 900 26px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  box-shadow: 0 4px 0 var(--c-ink);
  animation: popbadge 0.7s ease-in-out infinite alternate;
}

/* 紙吹雪レイヤー */
#confetti-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;
  z-index: 10;
  overflow: hidden;
  display: none;
}
#confetti-layer.active { display: block; }

/* 切断メッセージ */
#disconnect-msg {
  position: fixed;
  inset: 0;
  z-index: 100;
  background: rgba(26,34,56,0.85);
  display: none;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: 24px;
  color: #fff;
  font-size: 24px;
  font-weight: 800;
  text-align: center;
  padding: 40px;
}
#disconnect-msg.active { display: flex; }
#btn-reconnect {
  background: var(--c-mint);
  border: 4px solid var(--c-ink);
  border-radius: 28px;
  box-shadow: 0 6px 0 var(--c-ink);
  padding: 16px 40px;
  font: 900 24px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  cursor: pointer;
}

/* === RESULT SCREEN === */
#screen-result {
  background: #1a2238;
}
.result-card {
  width: 760px;
  max-width: 96vw;
  height: 570px;
  max-height: 96vh;
  background: var(--c-bg);
  border: 4px solid var(--c-ink);
  border-radius: 38px;
  box-shadow: 0 14px 34px rgba(26,34,56,.18);
  overflow: hidden;
  position: relative;
  display: flex;
  flex-direction: row;
}
.result-left {
  flex: 1.15;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  padding: 24px 12px;
  z-index: 2;
}
.result-star-burst {
  position: absolute;
  top: 90px;
  width: 250px; height: 250px;
  background: #ffe39a;
  clip-path: polygon(50% 0%,61% 35%,98% 35%,68% 57%,79% 91%,50% 70%,21% 91%,32% 57%,2% 35%,39% 35%);
  animation: starspin 18s linear infinite;
  opacity: 0.6;
}
.result-crown {
  position: relative;
  width: 88px; height: 58px;
  z-index: 2;
  margin-bottom: -8px;
}
.result-crown-back {
  position: absolute;
  inset: 0;
  background: var(--c-ink);
  clip-path: polygon(0% 100%,0% 32%,26% 56%,50% 4%,74% 56%,100% 32%,100% 100%);
}
.result-crown-front {
  position: absolute;
  left: 4px; right: 4px; top: 5px; bottom: 0;
  background: var(--c-yellow);
  clip-path: polygon(0% 100%,0% 32%,26% 56%,50% 4%,74% 56%,100% 32%,100% 100%);
}
.result-winner-char {
  position: relative;
  z-index: 2;
  width: 150px; height: 150px;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: hop 1.1s ease-in-out infinite;
}
.result-kachi {
  font: 900 32px 'M PLUS Rounded 1c';
  color: var(--c-red);
  margin-top: 16px;
  letter-spacing: 3px;
  z-index: 2;
}
.result-kachi.blue { color: var(--c-blue); }
.result-right {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 14px;
  padding: 26px 30px 26px 6px;
  z-index: 2;
}
.result-scores {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
}
.result-score-card {
  flex: 1;
  background: #fff;
  border: 4px solid var(--c-ink);
  border-radius: 24px;
  box-shadow: 0 6px 0 var(--c-ink);
  padding: 14px 8px 12px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}
.result-score-card.winner { outline: 5px solid var(--c-yellow); outline-offset: 3px; }
.result-score-vs {
  font: 900 24px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  opacity: 0.4;
}
.result-score-num {
  font: 900 48px/1 'M PLUS Rounded 1c';
}
.result-score-num-red { color: var(--c-red); }
.result-score-num-blue { color: var(--c-blue); }
.result-score-num.loser { font-size: 40px; }
.result-msg {
  text-align: center;
  font: 800 15px 'M PLUS Rounded 1c';
  color: var(--c-ink);
  opacity: 0.6;
}
#btn-replay {
  width: 100%;
  background: var(--c-mint);
  border: 4px solid var(--c-ink);
  border-radius: 28px;
  box-shadow: 0 8px 0 var(--c-ink);
  padding: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 14px;
  margin-top: 4px;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
  transition: transform 0.1s;
}
#btn-replay:active { transform: translateY(4px); box-shadow: 0 4px 0 var(--c-ink); }
.replay-icon {
  position: relative;
  width: 34px; height: 34px;
}
.replay-label {
  font: 900 34px 'M PLUS Rounded 1c';
  color: var(--c-ink);
}
</style>
</head>
<body>

<!-- 縦持ちオーバーレイ -->
<div id="screen-orient" class="screen">
  <span class="rotate-icon">📱</span>
  <div>iPadを よこにしてね</div>
</div>

<!-- 接続画面 -->
<div id="screen-connect" class="screen active">
  <div class="connect-card" style="height:392px;">

    <!-- 2a: えらぶ -->
    <div id="step-choose" class="connect-step active">
      <div class="connect-logo">
        <div class="connect-logo-icons">
          <!-- 青マレット小 -->
          <div style="width:40px;height:40px;border-radius:50%;background:var(--c-blue);border:3px solid var(--c-ink);box-shadow:0 4px 0 var(--c-ink);display:flex;align-items:center;justify-content:center;">
            <div style="width:18px;height:18px;border-radius:50%;background:var(--c-blue-light);border:2px solid var(--c-ink);"></div>
          </div>
          <!-- パック小 -->
          <div style="width:24px;height:24px;border-radius:50%;background:var(--c-ink);display:flex;align-items:center;justify-content:center;">
            <div style="width:9px;height:9px;border-radius:50%;background:var(--c-yellow);"></div>
          </div>
          <!-- 赤マレット小 -->
          <div style="width:40px;height:40px;border-radius:50%;background:var(--c-red);border:3px solid var(--c-ink);box-shadow:0 4px 0 var(--c-ink);display:flex;align-items:center;justify-content:center;">
            <div style="width:18px;height:18px;border-radius:50%;background:var(--c-red-light);border:2px solid var(--c-ink);"></div>
          </div>
        </div>
        <div class="connect-logo-title">エアホッケー</div>
      </div>
      <div class="connect-btns">
        <div id="btn-host" class="btn-role btn-role-host">
          <div style="width:72px;height:72px;border-radius:50%;background:#fff;border:4px solid var(--c-ink);display:flex;align-items:center;justify-content:center;position:relative;">
            <div style="position:absolute;width:34px;height:9px;border-radius:5px;background:var(--c-ink);"></div>
            <div style="position:absolute;width:9px;height:34px;border-radius:5px;background:var(--c-ink);"></div>
          </div>
          <div class="btn-role-label">へやを<br>つくる</div>
        </div>
        <div id="btn-guest" class="btn-role btn-role-guest">
          <div style="width:72px;height:72px;border-radius:18px;background:#fff;border:4px solid var(--c-ink);position:relative;">
            <div style="position:absolute;right:8px;top:10px;bottom:10px;width:22px;border:4px solid var(--c-ink);border-radius:6px;background:#fff3cf;"></div>
            <div style="position:absolute;left:9px;top:50%;transform:translateY(-50%);display:flex;align-items:center;">
              <div style="width:20px;height:8px;border-radius:4px;background:var(--c-ink);"></div>
              <div style="width:0;height:0;border-top:9px solid transparent;border-bottom:9px solid transparent;border-left:13px solid var(--c-ink);margin-left:-2px;"></div>
            </div>
          </div>
          <div class="btn-role-label">へやに<br>はいる</div>
        </div>
      </div>
    </div>

    <!-- 2b: ホスト待機 -->
    <div id="step-host-wait" class="connect-step">
      <div class="connect-host-inner">
        <div class="connect-code-panel">
          <div class="connect-code-label">あいことば</div>
          <div class="connect-code-tiles">
            <div class="code-tile" id="code-d0">-</div>
            <div class="code-tile" id="code-d1">-</div>
            <div class="code-tile" id="code-d2">-</div>
            <div class="code-tile" id="code-d3">-</div>
          </div>
        </div>
        <div class="connect-spinner-panel">
          <div class="spinner-wrap">
            <div class="spinner-ring"></div>
            <div class="spinner-char"><!-- 青キャラ --></div>
          </div>
          <div class="connect-wait-label">まってるよ…</div>
        </div>
      </div>
    </div>

    <!-- 2c: ゲスト入力 -->
    <div id="step-guest-input" class="connect-step">
      <div class="connect-guest-inner">
        <div class="connect-slots-panel">
          <div class="connect-slots" id="input-slots"></div>
        </div>
        <div class="connect-keypad" id="keypad"></div>
      </div>
    </div>

    <!-- 2d: つながった！ -->
    <div id="step-connected" class="connect-step">
      <div class="connect-success-inner">
        <div style="position:absolute;top:50px;left:50px;width:24px;height:24px;background:var(--c-yellow);clip-path:polygon(50% 0%,61% 35%,98% 35%,68% 57%,79% 91%,50% 70%,21% 91%,32% 57%,2% 35%,39% 35%);animation:float 2.4s ease-in-out infinite;"></div>
        <div style="position:absolute;top:70px;right:70px;width:18px;height:18px;background:var(--c-red);clip-path:polygon(50% 0%,61% 35%,98% 35%,68% 57%,79% 91%,50% 70%,21% 91%,32% 57%,2% 35%,39% 35%);animation:float 2.8s ease-in-out infinite;"></div>
        <div style="position:absolute;bottom:50px;left:90px;width:16px;height:16px;background:var(--c-blue);clip-path:polygon(50% 0%,61% 35%,98% 35%,68% 57%,79% 91%,50% 70%,21% 91%,32% 57%,2% 35%,39% 35%);animation:float 2.2s ease-in-out infinite;"></div>
        <div class="connect-success-check">
          <div style="position:absolute;inset:0;background:var(--c-mint);border:4px solid var(--c-ink);border-radius:50%;box-shadow:0 8px 0 var(--c-ink);"></div>
          <div style="position:absolute;left:40px;top:74px;width:28px;height:12px;background:#fff;border-radius:6px;transform:rotate(45deg);transform-origin:left;"></div>
          <div style="position:absolute;left:57px;top:85px;width:52px;height:12px;background:#fff;border-radius:6px;transform:rotate(-52deg);transform-origin:left;"></div>
        </div>
        <div class="connect-success-text">
          <div class="connect-success-title">つながった！</div>
          <div class="connect-success-chars">
            <div id="char-blue-hop" style="position:relative;width:56px;height:56px;animation:hop 1.1s ease-in-out infinite;"></div>
            <div id="char-red-hop" style="position:relative;width:56px;height:56px;animation:hop 1.1s 0.15s ease-in-out infinite;"></div>
          </div>
        </div>
      </div>
    </div>

  </div>
</div>

<!-- 対戦画面 -->
<div id="screen-game" class="screen">
  <div id="game-score-bar">
    <div class="score-left">
      <div id="char-blue" style="position:relative;width:52px;height:52px;flex:none;"></div>
      <div id="score-blue" class="score-num score-num-blue">0</div>
    </div>
    <div id="btn-pause">
      <div class="pause-bar"></div>
      <div class="pause-bar"></div>
    </div>
    <div class="score-right">
      <div id="score-red" class="score-num score-num-red">0</div>
      <div id="char-red" style="position:relative;width:52px;height:52px;flex:none;"></div>
    </div>
  </div>
  <div id="game-canvas-wrap">
    <canvas id="game-canvas"></canvas>
    <div id="goal-overlay">
      <div class="ping-ring" id="ping-ring" style="width:90px;height:90px;"></div>
      <div class="plus-one" id="plus-one">+1</div>
    </div>
    <div id="confetti-layer"></div>
  </div>
</div>

<!-- 結果画面 -->
<div id="screen-result" class="screen">
  <div class="result-card">
    <div id="result-confetti" style="position:absolute;inset:0;overflow:hidden;pointer-events:none;z-index:9;"></div>
    <div class="result-left">
      <div class="result-star-burst"></div>
      <div class="result-crown">
        <div class="result-crown-back"></div>
        <div class="result-crown-front"></div>
      </div>
      <div class="result-winner-char" id="result-winner-char"></div>
      <div class="result-kachi" id="result-kachi">かち！</div>
    </div>
    <div class="result-right">
      <div class="result-scores">
        <div class="result-score-card" id="result-card-left">
          <div id="result-char-left" style="position:relative;width:52px;height:52px;"></div>
          <div class="result-score-num result-score-num-red" id="result-num-left">0</div>
        </div>
        <div class="result-score-vs">VS</div>
        <div class="result-score-card" id="result-card-right">
          <div id="result-char-right" style="position:relative;width:48px;height:48px;"></div>
          <div class="result-score-num result-score-num-blue" id="result-num-right">0</div>
        </div>
      </div>
      <div class="result-msg" id="result-msg">つぎは がんばろう！</div>
      <div id="btn-replay">
        <div class="replay-icon">
          <div style="position:absolute;inset:0;border:5px solid var(--c-ink);border-radius:50%;border-top-color:transparent;transform:rotate(-40deg);"></div>
          <div style="position:absolute;top:-3px;right:0;width:0;height:0;border-left:8px solid transparent;border-right:8px solid transparent;border-bottom:12px solid var(--c-ink);transform:rotate(55deg);"></div>
        </div>
        <div class="replay-label">もう1回</div>
      </div>
    </div>
  </div>
</div>

<!-- 切断メッセージ -->
<div id="disconnect-msg">
  <div>😢 あいてが きれました</div>
  <div id="btn-reconnect">もどる</div>
</div>

<script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
<script>
// ========== PLACEHOLDER: ゲームロジック（Task 2〜10 で追記） ==========
</script>
</body>
</html>
```

- [ ] **Step 2: ブラウザで開いて表示を確認する**

```bash
# index.html をブラウザで開く（またはローカルサーバー）
python -m http.server 8080
# → http://localhost:8080/index.html を開く
```
期待結果: 接続画面（えらぶ）が表示される。「へやを つくる」「へやに はいる」ボタンが見える。

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: HTML skeleton and CSS foundation"
```

---

### Task 2: CONFIG定数 + 状態管理 + 画面切替

**Files:**
- Modify: `index.html` — `<script>` 内に追記

**Interfaces:**
- Produces:
  - `CONFIG` オブジェクト（全チューニング定数）
  - `state` オブジェクト（ゲーム状態）
  - `showScreen(name)` 関数 — `'connect'|'game'|'result'` で画面切替
  - `showConnectStep(step)` 関数 — `'choose'|'hostWait'|'guestInput'|'connected'` でステップ切替

- [ ] **Step 1: ゲームロジックのscriptタグ内にCONFIG定数を書く**

```js
// ========== CONFIG: チューニング定数 ==========
const CONFIG = {
  WIN_SCORE: 5,           // 先取点数

  // パック
  PUCK_RADIUS: 0.038,     // 正規化半径（Canvas幅に対する比率）
  PUCK_INIT_SPEED: 0.004, // 初速（正規化単位/フレーム）
  PUCK_MAX_SPEED: 0.018,  // 最高速
  PUCK_MIN_SPEED: 0.002,  // 最低速（これ未満なら加速）
  PUCK_FRICTION: 0.9995,  // 毎フレームの摩擦係数（1=なし）

  // マレット（年上/年下）
  MALLET_RADIUS_ELDER: 0.055,  // 年上マレット正規化半径
  MALLET_RADIUS_YOUNGER: 0.070, // 年下マレット正規化半径（大きい）

  // ゴール幅（正規化。台の高さに対する比率）
  GOAL_WIDTH_ELDER: 0.45,   // 年上側ゴール（狭い＝守りにくい）
  GOAL_WIDTH_YOUNGER: 0.55, // 年下側ゴール（広い＝守りやすい）

  // 物理
  RESTITUTION: 1.2,       // マレット衝突時パックへ伝える速度倍率

  // 通信
  SYNC_HZ: 30,            // ホスト→ゲスト状態同期Hz
  LERP_ALPHA: 0.25,       // ゲスト側補間係数（0〜1、小さいほど滑らか）

  // 演出
  GOAL_ANIM_MS: 1800,     // 得点演出フェーズ時間（ミリ秒）
  CONNECTED_SHOW_MS: 1500, // 「つながった！」表示時間

  // ハンデ（デフォルトオフ）
  HANDICAP_ENABLED: false,
  YOUNGER_SIDE: 'left',   // 'left' or 'right'
};

// ========== STATE ==========
const state = {
  // 画面
  screen: 'connect',            // 'connect' | 'game' | 'result'
  connectStep: 'choose',        // 'choose' | 'hostWait' | 'guestInput' | 'connected'

  // 役割
  role: null,                   // 'host' | 'guest'
  roomCode: '',                 // 4桁文字列

  // ゲスト入力
  inputDigits: [],              // 入力中の数字配列（0〜4桁）

  // ゲーム
  phase: 'waiting',             // 'waiting' | 'playing' | 'goal' | 'result'
  score: { host: 0, guest: 0 },
  winner: null,                 // 'host' | 'guest' | null

  // ゲームオブジェクト（正規化座標）
  puck: { x: 0.5, y: 0.5, vx: 0, vy: 0 },
  malletHost: { x: 0.2, y: 0.5 },
  malletGuest: { x: 0.8, y: 0.5 },

  // ゲスト側の補間用
  puckTarget: { x: 0.5, y: 0.5 },

  // 演出フラグ
  goalSide: null,               // 'host' | 'guest'（最後に得点した側）
  isPaused: false,

  // ハンデ設定（接続時に決定）
  handicap: {
    enabled: false,
    youngerSide: 'left',        // 'left' | 'right'
  },
};
```

- [ ] **Step 2: 画面切替関数を書く**

```js
// ========== SCREEN MANAGEMENT ==========
function showScreen(name) {
  state.screen = name;
  document.getElementById('screen-connect').classList.toggle('active', name === 'connect');
  document.getElementById('screen-game').classList.toggle('active', name === 'game');
  document.getElementById('screen-result').classList.toggle('active', name === 'result');
}

function showConnectStep(step) {
  state.connectStep = step;
  const steps = ['choose', 'hostWait', 'guestInput', 'connected'];
  const ids = {
    choose: 'step-choose',
    hostWait: 'step-host-wait',
    guestInput: 'step-guest-input',
    connected: 'step-connected',
  };
  steps.forEach(s => {
    document.getElementById(ids[s]).classList.toggle('active', s === step);
  });
}
```

- [ ] **Step 3: 縦持ち検知を追加**

```js
// ========== ORIENTATION CHECK ==========
function checkOrientation() {
  const isPortrait = window.matchMedia('(orientation: portrait)').matches;
  document.getElementById('screen-orient').classList.toggle('active', isPortrait);
}
window.matchMedia('(orientation: portrait)').addEventListener('change', checkOrientation);
checkOrientation();
```

- [ ] **Step 4: ブラウザで確認**

コンソールで `showScreen('game')` 、`showScreen('result')` を実行して画面が切り替わることを確認。

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat: config constants, state management, screen switching"
```

---

### Task 3: Canvas描画エンジン

**Files:**
- Modify: `index.html` — script 内に追記

**Interfaces:**
- Consumes: `state` (puck, malletHost, malletGuest, phase, score, goalSide), `CONFIG`
- Produces:
  - `initCanvas()` — Canvasのサイズ設定
  - `render()` — 1フレーム描画（requestAnimationFrame から呼ばれる）
  - `getCanvasCoords(normX, normY)` — 正規化→Canvas pixel座標変換
  - `getNormCoords(canvasX, canvasY)` — Canvas pixel→正規化座標変換

- [ ] **Step 1: Canvas変数とリサイズ処理**

```js
// ========== CANVAS ==========
const canvas = document.getElementById('game-canvas');
const ctx = canvas.getContext('2d');
let canvasW = 0, canvasH = 0;

function initCanvas() {
  const wrap = document.getElementById('game-canvas-wrap');
  const rect = wrap.getBoundingClientRect();
  canvas.width = rect.width;
  canvas.height = rect.height;
  canvasW = canvas.width;
  canvasH = canvas.height;
}

window.addEventListener('resize', () => {
  if (state.screen === 'game') initCanvas();
});

function getCanvasCoords(nx, ny) {
  return { x: nx * canvasW, y: ny * canvasH };
}

function getNormCoords(cx, cy) {
  return { x: cx / canvasW, y: cy / canvasH };
}
```

- [ ] **Step 2: 台・ゴール・中央線の描画関数**

```js
// ゴール設定を取得（ハンデ考慮）
function getGoalConfig() {
  const h = state.handicap;
  // hostは左側プレイヤー、guestは右側プレイヤー
  const leftGoalWidth = h.enabled && h.youngerSide === 'left'
    ? CONFIG.GOAL_WIDTH_YOUNGER : CONFIG.GOAL_WIDTH_ELDER;
  const rightGoalWidth = h.enabled && h.youngerSide === 'right'
    ? CONFIG.GOAL_WIDTH_YOUNGER : CONFIG.GOAL_WIDTH_ELDER;
  return { leftGoalWidth, rightGoalWidth };
}

function drawTable() {
  const { leftGoalWidth, rightGoalWidth } = getGoalConfig();
  const goalDepth = 50; // Canvas pixel単位（固定感がある方が見やすい）

  // 木枠背景
  const woodGrad = ctx.createLinearGradient(0, 0, 0, canvasH);
  woodGrad.addColorStop(0, '#cda473');
  woodGrad.addColorStop(1, '#bb874f');
  const pad = 10;
  ctx.fillStyle = woodGrad;
  ctx.beginPath();
  ctx.roundRect(0, 0, canvasW, canvasH, 20);
  ctx.fill();

  // プレイ面（左右グラデーション）
  const playGrad = ctx.createLinearGradient(pad, 0, canvasW - pad, 0);
  playGrad.addColorStop(0, '#e7f3fb');
  playGrad.addColorStop(0.5, '#e7f3fb');
  playGrad.addColorStop(0.5, '#ffeeeb');
  playGrad.addColorStop(1, '#ffeeeb');
  ctx.fillStyle = playGrad;
  ctx.beginPath();
  ctx.roundRect(pad, pad, canvasW - pad * 2, canvasH - pad * 2, 14);
  ctx.fill();
  ctx.strokeStyle = '#1a2238';
  ctx.lineWidth = 3;
  ctx.stroke();

  // 中央縦破線
  ctx.save();
  ctx.strokeStyle = 'rgba(26,34,56,0.45)';
  ctx.lineWidth = 5;
  ctx.setLineDash([12, 10]);
  ctx.beginPath();
  ctx.moveTo(canvasW / 2, pad);
  ctx.lineTo(canvasW / 2, canvasH - pad);
  ctx.stroke();
  ctx.setLineDash([]);
  ctx.restore();

  // 中央破線円
  ctx.save();
  ctx.strokeStyle = 'rgba(26,34,56,0.4)';
  ctx.lineWidth = 4;
  ctx.setLineDash([10, 8]);
  ctx.beginPath();
  ctx.arc(canvasW / 2, canvasH / 2, 48, 0, Math.PI * 2);
  ctx.stroke();
  ctx.setLineDash([]);
  ctx.restore();

  // 左ゴール（青のネット面）
  const leftGoalH = canvasH * leftGoalWidth;
  const leftGoalTop = (canvasH - leftGoalH) / 2;
  ctx.fillStyle = state.goalSide === 'host' ? CONFIG._goalFlashActive ? '#ffe39a' : '#bfe0f5' : '#bfe0f5';
  ctx.beginPath();
  ctx.roundRect(0, leftGoalTop, goalDepth, leftGoalH, [0, 40, 40, 0]);
  ctx.fill();
  ctx.strokeStyle = '#1a2238';
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.moveTo(0, leftGoalTop);
  ctx.arcTo(goalDepth, leftGoalTop, goalDepth, leftGoalTop + leftGoalH, 40);
  ctx.arcTo(goalDepth, leftGoalTop + leftGoalH, 0, leftGoalTop + leftGoalH, 40);
  ctx.lineTo(0, leftGoalTop + leftGoalH);
  ctx.stroke();
  // 左ネット縞
  drawNetLines(0, leftGoalTop + leftGoalH * 0.18, goalDepth * 0.62, leftGoalH * 0.64, false);

  // 右ゴール（赤のネット面）
  const rightGoalH = canvasH * rightGoalWidth;
  const rightGoalTop = (canvasH - rightGoalH) / 2;
  ctx.fillStyle = state.goalSide === 'guest' ? CONFIG._goalFlashActive ? '#ffe39a' : '#ffc9c2' : '#ffc9c2';
  ctx.beginPath();
  ctx.roundRect(canvasW - goalDepth, rightGoalTop, goalDepth, rightGoalH, [40, 0, 0, 40]);
  ctx.fill();
  ctx.strokeStyle = '#1a2238';
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.moveTo(canvasW, rightGoalTop);
  ctx.arcTo(canvasW - goalDepth, rightGoalTop, canvasW - goalDepth, rightGoalTop + rightGoalH, 40);
  ctx.arcTo(canvasW - goalDepth, rightGoalTop + rightGoalH, canvasW, rightGoalTop + rightGoalH, 40);
  ctx.lineTo(canvasW, rightGoalTop + rightGoalH);
  ctx.stroke();
  // 右ネット縞
  drawNetLines(canvasW - goalDepth * 0.62, rightGoalTop + rightGoalH * 0.18, goalDepth * 0.62, rightGoalH * 0.64, false);
}

function drawNetLines(x, y, w, h, _unused) {
  ctx.save();
  ctx.globalAlpha = 0.45;
  ctx.strokeStyle = '#1a2238';
  ctx.lineWidth = 2;
  const spacing = 11;
  for (let ly = y; ly < y + h; ly += spacing) {
    ctx.beginPath();
    ctx.moveTo(x, ly);
    ctx.lineTo(x + w, ly);
    ctx.stroke();
  }
  ctx.restore();
}
```

- [ ] **Step 3: マレット・パックの描画関数**

```js
function getMalletRadius(side) {
  const h = state.handicap;
  if (!h.enabled) return CONFIG.MALLET_RADIUS_ELDER;
  const isYoungerSide = (side === 'left' && h.youngerSide === 'left') ||
                        (side === 'right' && h.youngerSide === 'right');
  return isYoungerSide ? CONFIG.MALLET_RADIUS_YOUNGER : CONFIG.MALLET_RADIUS_ELDER;
}

function drawMallet(nx, ny, color, lightColor, side) {
  const { x, y } = getCanvasCoords(nx, ny);
  const r = getMalletRadius(side) * canvasW;
  const innerR = r * 0.52;

  // 影
  ctx.save();
  ctx.shadowColor = 'rgba(26,34,56,0.5)';
  ctx.shadowOffsetY = 5;
  ctx.shadowBlur = 0;

  // 本体
  ctx.fillStyle = color;
  ctx.strokeStyle = '#1a2238';
  ctx.lineWidth = 4;
  ctx.beginPath();
  ctx.arc(x, y, r, 0, Math.PI * 2);
  ctx.fill();
  ctx.stroke();

  ctx.restore();

  // 内側ディスク
  ctx.fillStyle = lightColor;
  ctx.strokeStyle = '#1a2238';
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.arc(x, y, innerR, 0, Math.PI * 2);
  ctx.fill();
  ctx.stroke();
}

function drawPuck(nx, ny) {
  const { x, y } = getCanvasCoords(nx, ny);
  const r = CONFIG.PUCK_RADIUS * canvasW;

  ctx.save();
  ctx.shadowColor = 'rgba(26,34,56,0.35)';
  ctx.shadowOffsetY = 4;
  ctx.shadowBlur = 0;

  ctx.fillStyle = '#1a2238';
  ctx.beginPath();
  ctx.arc(x, y, r, 0, Math.PI * 2);
  ctx.fill();

  ctx.restore();

  // イエロードット
  ctx.fillStyle = '#ffc233';
  ctx.beginPath();
  ctx.arc(x, y, r * 0.39, 0, Math.PI * 2);
  ctx.fill();
}
```

- [ ] **Step 4: render()関数（メインのフレーム描画）**

```js
function render() {
  if (state.screen !== 'game') return;

  ctx.clearRect(0, 0, canvasW, canvasH);
  drawTable();

  // ホスト（左）のマレット: role=host は自分のマレット、guest は受信したホストマレット
  const hostPos = state.malletHost;
  const guestPos = state.malletGuest;

  drawMallet(hostPos.x, hostPos.y, '#4ea8de', '#bfe0f5', 'left');
  drawMallet(guestPos.x, guestPos.y, '#ff6b5e', '#ffd4cd', 'right');

  // パック（ゲスト側はlerp後の座標）
  const px = state.role === 'guest' ? state.puck.x : state.puck.x;
  const py = state.role === 'guest' ? state.puck.y : state.puck.y;
  drawPuck(px, py);
}
```

- [ ] **Step 5: ゲーム画面テスト**

コンソールで以下を実行して描画確認：
```js
showScreen('game');
initCanvas();
state.puck = {x:0.5, y:0.5, vx:0, vy:0};
state.malletHost = {x:0.2, y:0.5};
state.malletGuest = {x:0.8, y:0.5};
render();
```
期待結果: 台（木枠・左青/右赤のプレイ面・中央線・ゴール）＋青マレット＋赤マレット＋パックが表示される。

- [ ] **Step 6: コミット**

```bash
git add index.html
git commit -m "feat: canvas rendering - table, mallets, puck"
```

---

### Task 4: キャラクター描画ヘルパー + DOM UI組み立て

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces:
  - `drawCharacter(el, color, cheekColor, size)` — DOM要素にキャラを生成
  - `buildKeypad()` — ゲストキーパッドHTML生成
  - `updateScoreBar()` — スコアバー更新
  - `updateInputSlots()` — 入力スロット表示更新

- [ ] **Step 1: キャラクター描画ヘルパー（DOM要素へインライン構築）**

```js
// ========== CHARACTER DOM BUILDER ==========
function buildChar(color, cheekColor, size = 58) {
  const s = size;
  return `
    <div style="position:absolute;inset:0;background:${color};border:${Math.round(s*0.069)}px solid #1a2238;border-radius:50%;box-shadow:0 ${Math.round(s*0.086)}px 0 #1a2238;"></div>
    <div style="position:absolute;top:${Math.round(s*0.362)}px;left:${Math.round(s*0.241)}px;width:${Math.round(s*0.138)}px;height:${Math.round(s*0.19)}px;background:#1a2238;border-radius:50%;"></div>
    <div style="position:absolute;top:${Math.round(s*0.362)}px;right:${Math.round(s*0.241)}px;width:${Math.round(s*0.138)}px;height:${Math.round(s*0.19)}px;background:#1a2238;border-radius:50%;"></div>
    <div style="position:absolute;top:${Math.round(s*0.534)}px;left:${Math.round(s*0.12)}px;width:${Math.round(s*0.172)}px;height:${Math.round(s*0.103)}px;background:${cheekColor};border-radius:50%;"></div>
    <div style="position:absolute;top:${Math.round(s*0.534)}px;right:${Math.round(s*0.12)}px;width:${Math.round(s*0.172)}px;height:${Math.round(s*0.103)}px;background:${cheekColor};border-radius:50%;"></div>
    <div style="position:absolute;top:${Math.round(s*0.534)}px;left:50%;transform:translateX(-50%);width:${Math.round(s*0.31)}px;height:${Math.round(s*0.155)}px;border:3px solid #1a2238;border-top:none;border-radius:0 0 ${Math.round(s*0.31)}px ${Math.round(s*0.31)}px;"></div>
  `;
}

function initCharElements() {
  // スコアバーのキャラ
  document.getElementById('char-blue').innerHTML = buildChar('#4ea8de', '#ff9d90', 52);
  document.getElementById('char-red').innerHTML = buildChar('#ff6b5e', '#ffce6b', 52);
  // 接続成功画面のキャラ
  document.getElementById('char-blue-hop').innerHTML = buildChar('#4ea8de', '#ff9d90', 56);
  document.getElementById('char-red-hop').innerHTML = buildChar('#ff6b5e', '#ffce6b', 56);
  // スピナー中のキャラ
  document.querySelector('.spinner-char').innerHTML = buildChar('#4ea8de', '#ff9d90', 60);
}
```

- [ ] **Step 2: キーパッド生成**

```js
function buildKeypad() {
  const keypad = document.getElementById('keypad');
  keypad.innerHTML = '';
  const keys = ['1','2','3','4','5','6','7','8','9','del','0','ok'];
  keys.forEach(k => {
    const div = document.createElement('div');
    div.className = 'key';
    if (k === 'del') {
      div.className += ' key-del';
      div.innerHTML = `<div style="position:relative;width:24px;height:24px;">
        <div style="position:absolute;top:9px;left:1px;width:22px;height:6px;border-radius:3px;background:#1a2238;transform:rotate(45deg);"></div>
        <div style="position:absolute;top:9px;left:1px;width:22px;height:6px;border-radius:3px;background:#1a2238;transform:rotate(-45deg);"></div>
      </div>`;
      div.addEventListener('click', onKeyDel);
    } else if (k === 'ok') {
      div.className += ' key-ok';
      div.innerHTML = `<div style="position:relative;width:30px;height:30px;">
        <div style="position:absolute;left:2px;top:15px;width:12px;height:6px;background:#1a2238;border-radius:3px;transform:rotate(45deg);transform-origin:left;"></div>
        <div style="position:absolute;left:10px;top:20px;width:23px;height:6px;background:#1a2238;border-radius:3px;transform:rotate(-52deg);transform-origin:left;"></div>
      </div>`;
      div.addEventListener('click', onKeyOk);
    } else {
      div.textContent = k;
      div.dataset.digit = k;
      div.addEventListener('click', () => onKeyDigit(k));
    }
    keypad.appendChild(div);
  });
}

function updateInputSlots() {
  const slots = document.getElementById('input-slots');
  slots.innerHTML = '';
  for (let i = 0; i < 4; i++) {
    const div = document.createElement('div');
    div.className = 'input-slot';
    if (i < state.inputDigits.length) {
      div.textContent = state.inputDigits[i];
    } else {
      div.classList.add('input-slot-empty');
    }
    slots.appendChild(div);
  }
}

function onKeyDigit(d) {
  if (state.inputDigits.length >= 4) return;
  state.inputDigits.push(d);
  updateInputSlots();
}

function onKeyDel() {
  state.inputDigits.pop();
  updateInputSlots();
}

function onKeyOk() {
  if (state.inputDigits.length < 4) return;
  const code = state.inputDigits.join('');
  connectAsGuest(code);
}
```

- [ ] **Step 3: スコアバー更新**

```js
function updateScoreBar() {
  const blueEl = document.getElementById('score-blue');
  const redEl = document.getElementById('score-red');

  // hostは左（青）、guestは右（赤）
  blueEl.textContent = state.score.host;
  redEl.textContent = state.score.guest;

  blueEl.classList.toggle('scoring', state.goalSide === 'host' && state.phase === 'goal');
  redEl.classList.toggle('scoring', state.goalSide === 'guest' && state.phase === 'goal');
  blueEl.classList.toggle('dimmed', state.goalSide === 'guest' && state.phase === 'goal');
  redEl.classList.toggle('dimmed', state.goalSide === 'host' && state.phase === 'goal');
}
```

- [ ] **Step 4: 合言葉タイル表示（ホスト）**

```js
function displayRoomCode(code) {
  state.roomCode = code;
  ['d0','d1','d2','d3'].forEach((id, i) => {
    document.getElementById('code-' + id).textContent = code[i] || '-';
  });
}
```

- [ ] **Step 5: 初期化呼び出しを追加**

DOMContentLoaded または script末尾に:
```js
function init() {
  initCharElements();
  buildKeypad();
  updateInputSlots();
  checkOrientation();
}
document.addEventListener('DOMContentLoaded', init);
```

- [ ] **Step 6: ブラウザで確認**

ブラウザで開き、キャラクターが表示されること、キーパッドが表示されることを確認。

- [ ] **Step 7: コミット**

```bash
git add index.html
git commit -m "feat: character DOM builder, keypad, score bar UI"
```

---

### Task 5: PeerJS通信レイヤー

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `state.role`, `state.roomCode`, PeerJS CDN（`window.Peer`）
- Produces:
  - `startHost()` — ホストとして部屋を作成
  - `connectAsGuest(code)` — ゲストとして接続
  - `sendToGuest(data)` — ホスト→ゲスト送信
  - `sendToHost(data)` — ゲスト→ホスト送信
  - コールバック: `onGuestMalletUpdate(x, y)`, `onStateReceived(data)`

- [ ] **Step 1: PeerJS変数と送信関数**

```js
// ========== PEERJS ==========
let peer = null;
let conn = null;

function sendToGuest(data) {
  if (conn && conn.open) {
    try { conn.send(data); } catch(e) {}
  }
}

function sendToHost(data) {
  if (conn && conn.open) {
    try { conn.send(data); } catch(e) {}
  }
}

function onConnectionEstablished() {
  showConnectStep('connected');
  setTimeout(() => {
    showScreen('game');
    initCanvas();
    if (state.role === 'host') {
      startGame();
    }
    startGameLoop();
  }, CONFIG.CONNECTED_SHOW_MS);
}

function showDisconnect() {
  document.getElementById('disconnect-msg').classList.add('active');
}

function hideDisconnect() {
  document.getElementById('disconnect-msg').classList.remove('active');
}
```

- [ ] **Step 2: ホスト開始**

```js
function startHost() {
  state.role = 'host';
  const code = String(Math.floor(1000 + Math.random() * 9000));
  const peerId = 'air-' + code;

  peer = new Peer(peerId);

  peer.on('open', () => {
    displayRoomCode(code);
    showConnectStep('hostWait');
  });

  peer.on('connection', (connection) => {
    conn = connection;
    conn.on('open', () => {
      showConnectStep('connected');
      onConnectionEstablished();
    });
    conn.on('data', (data) => {
      if (data.t === 'mallet') {
        state.malletGuest.x = data.x;
        state.malletGuest.y = data.y;
      }
    });
    conn.on('close', showDisconnect);
    conn.on('error', showDisconnect);
  });

  peer.on('error', (err) => {
    if (err.type === 'unavailable-id') {
      // 衝突したら別のコードで再試行
      peer.destroy();
      setTimeout(startHost, 200);
    } else {
      console.error('PeerJS error:', err);
    }
  });

  peer.on('disconnected', () => {
    if (state.screen === 'game') showDisconnect();
  });
}
```

- [ ] **Step 3: ゲスト接続**

```js
function connectAsGuest(code) {
  state.role = 'guest';
  const peerId = 'air-' + code;

  peer = new Peer();

  peer.on('open', () => {
    conn = peer.connect(peerId, {
      reliable: false,
      serialization: 'json',
    });

    conn.on('open', () => {
      onConnectionEstablished();
    });

    conn.on('data', (data) => {
      if (data.t === 'state') {
        onStateReceived(data);
      }
    });

    conn.on('close', showDisconnect);
    conn.on('error', showDisconnect);
  });

  peer.on('error', (err) => {
    if (err.type === 'peer-unavailable') {
      alert('合言葉が ちがうかも…もう一度 確認してね');
      state.inputDigits = [];
      updateInputSlots();
    } else {
      console.error('PeerJS error:', err);
    }
  });

  peer.on('disconnected', () => {
    if (state.screen === 'game') showDisconnect();
  });
}

function onStateReceived(data) {
  // ゲスト側: サーバーから受信した状態を適用
  state.puckTarget.x = data.puck.x;
  state.puckTarget.y = data.puck.y;
  state.malletHost.x = data.mallet.x;
  state.malletHost.y = data.mallet.y;
  state.score.host = data.score.host;
  state.score.guest = data.score.guest;

  if (data.phase !== state.phase) {
    state.phase = data.phase;
    if (data.phase === 'goal') {
      state.goalSide = data.goalSide;
      startGoalAnimation();
    } else if (data.phase === 'result') {
      state.winner = data.winner;
      showResult();
    }
  }

  updateScoreBar();
}
```

- [ ] **Step 4: ホスト→ゲスト状態ブロードキャスト**

```js
let lastSyncTime = 0;
const SYNC_INTERVAL = 1000 / CONFIG.SYNC_HZ;

function broadcastState() {
  if (state.role !== 'host') return;
  const now = performance.now();
  if (now - lastSyncTime < SYNC_INTERVAL) return;
  lastSyncTime = now;

  sendToGuest({
    t: 'state',
    puck: { x: state.puck.x, y: state.puck.y },
    mallet: { x: state.malletHost.x, y: state.malletHost.y },
    score: state.score,
    phase: state.phase,
    goalSide: state.goalSide,
    winner: state.winner,
  });
}
```

- [ ] **Step 5: 再接続ボタン**

```js
document.getElementById('btn-reconnect').addEventListener('click', () => {
  hideDisconnect();
  if (peer) { try { peer.destroy(); } catch(e) {} }
  peer = null; conn = null;
  state.phase = 'waiting';
  state.screen = 'connect';
  state.connectStep = 'choose';
  state.role = null;
  showScreen('connect');
  showConnectStep('choose');
});
```

- [ ] **Step 6: 接続ボタンイベント**

```js
document.getElementById('btn-host').addEventListener('click', startHost);
document.getElementById('btn-guest').addEventListener('click', () => {
  state.inputDigits = [];
  updateInputSlots();
  showConnectStep('guestInput');
});
```

- [ ] **Step 7: 2タブで接続テスト**

1. タブ1を開く → 「へやを つくる」クリック → 合言葉4桁が表示される
2. タブ2を開く → 「へやに はいる」→ 合言葉入力 → 「つながった！」が両タブに表示される

期待結果: 約1.5秒後にゲーム画面へ遷移する。

- [ ] **Step 8: コミット**

```bash
git add index.html
git commit -m "feat: PeerJS connection, host/guest roles, state broadcast"
```

---

### Task 6: 物理エンジン（ホスト側）

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `state.puck`, `state.malletHost`, `state.malletGuest`, `CONFIG`, `getGoalConfig()`
- Produces:
  - `startGame()` — ゲーム開始（パック中央配置）
  - `updatePhysics()` — 1フレーム分の物理演算（ホスト専用）
  - `checkGoal()` — ゴール判定。得点時 `state.phase = 'goal'` に

- [ ] **Step 1: ゲーム開始とパックリセット**

```js
// ========== PHYSICS (HOST ONLY) ==========
function startGame() {
  state.score = { host: 0, guest: 0 };
  state.phase = 'playing';
  state.winner = null;
  state.goalSide = null;
  resetPuck();
  updateScoreBar();
}

function resetPuck() {
  state.puck.x = 0.5;
  state.puck.y = 0.5;
  // ランダムな角度で発射（45°±30°の範囲、左右どちらかへ）
  const angle = (Math.random() * 0.52 - 0.26) + (Math.random() > 0.5 ? 0 : Math.PI);
  state.puck.vx = Math.cos(angle) * CONFIG.PUCK_INIT_SPEED;
  state.puck.vy = Math.sin(angle) * CONFIG.PUCK_INIT_SPEED;
}
```

- [ ] **Step 2: 物理演算メインループ**

```js
function updatePhysics() {
  if (state.role !== 'host') return;
  if (state.phase !== 'playing') return;

  const p = state.puck;
  const { leftGoalWidth, rightGoalWidth } = getGoalConfig();
  const GOAL_DEPTH_NORM = 50 / canvasW; // ゴール奥行き（正規化）

  // 摩擦
  p.vx *= CONFIG.PUCK_FRICTION;
  p.vy *= CONFIG.PUCK_FRICTION;

  // 最低速度保証
  const speed = Math.sqrt(p.vx * p.vx + p.vy * p.vy);
  if (speed > 0 && speed < CONFIG.PUCK_MIN_SPEED) {
    const scale = CONFIG.PUCK_MIN_SPEED / speed;
    p.vx *= scale;
    p.vy *= scale;
  }

  // 移動
  p.x += p.vx;
  p.y += p.vy;

  // 上下の壁反射（プレイ面境界。Canvas上のpad=10pxを正規化）
  const padY = 10 / canvasH;
  const padX = 10 / canvasW;
  const r = CONFIG.PUCK_RADIUS;

  if (p.y - r < padY) { p.y = padY + r; p.vy = Math.abs(p.vy); }
  if (p.y + r > 1 - padY) { p.y = 1 - padY - r; p.vy = -Math.abs(p.vy); }

  // 左ゴール判定
  const leftGoalTop = (1 - leftGoalWidth) / 2;
  const leftGoalBot = leftGoalTop + leftGoalWidth;
  if (p.x - r < padX + GOAL_DEPTH_NORM) {
    if (p.y > leftGoalTop && p.y < leftGoalBot) {
      // ゴール（ゲスト得点）
      onGoal('guest');
      return;
    } else {
      // 壁反射
      if (p.x - r < padX) {
        p.x = padX + r;
        p.vx = Math.abs(p.vx);
      }
    }
  }

  // 右ゴール判定
  const rightGoalTop = (1 - rightGoalWidth) / 2;
  const rightGoalBot = rightGoalTop + rightGoalWidth;
  if (p.x + r > 1 - padX - GOAL_DEPTH_NORM) {
    if (p.y > rightGoalTop && p.y < rightGoalBot) {
      // ゴール（ホスト得点）
      onGoal('host');
      return;
    } else {
      // 壁反射
      if (p.x + r > 1 - padX) {
        p.x = 1 - padX - r;
        p.vx = -Math.abs(p.vx);
      }
    }
  }

  // マレット衝突
  collideMalletPuck(state.malletHost, 'left');
  collideMalletPuck(state.malletGuest, 'right');
}
```

- [ ] **Step 3: マレット衝突処理**

```js
let prevMalletHost = { x: 0.2, y: 0.5 };
let prevMalletGuest = { x: 0.8, y: 0.5 };

function collideMalletPuck(mallet, side) {
  const p = state.puck;
  const mr = getMalletRadius(side) * (canvasW / canvasW); // 正規化半径 = getMalletRadius で返る値
  const pr = CONFIG.PUCK_RADIUS;
  const malletR = side === 'left'
    ? getMalletRadius('left')
    : getMalletRadius('right');
  const dist = Math.sqrt((p.x - mallet.x) ** 2 + (p.y - mallet.y) ** 2);
  const minDist = malletR + pr;

  if (dist < minDist && dist > 0) {
    // 法線方向に押し出す
    const nx = (p.x - mallet.x) / dist;
    const ny = (p.y - mallet.y) / dist;

    // めり込み解消
    p.x = mallet.x + nx * minDist;
    p.y = mallet.y + ny * minDist;

    // マレットの移動速度を取得
    const prevM = side === 'left' ? prevMalletHost : prevMalletGuest;
    const mvx = mallet.x - prevM.x;
    const mvy = mallet.y - prevM.y;

    // 衝突後速度 = 法線方向の速度反射 + マレット速度 * 反発係数
    const dot = p.vx * nx + p.vy * ny;
    p.vx = p.vx - 2 * dot * nx + mvx * CONFIG.RESTITUTION;
    p.vy = p.vy - 2 * dot * ny + mvy * CONFIG.RESTITUTION;

    // 速度上限
    const speed = Math.sqrt(p.vx * p.vx + p.vy * p.vy);
    if (speed > CONFIG.PUCK_MAX_SPEED) {
      p.vx = (p.vx / speed) * CONFIG.PUCK_MAX_SPEED;
      p.vy = (p.vy / speed) * CONFIG.PUCK_MAX_SPEED;
    }
  }

  // 前フレーム位置を記録
  if (side === 'left') {
    prevMalletHost.x = mallet.x;
    prevMalletHost.y = mallet.y;
  } else {
    prevMalletGuest.x = mallet.x;
    prevMalletGuest.y = mallet.y;
  }
}
```

- [ ] **Step 4: ゴール処理**

```js
function onGoal(scoringSide) {
  state.phase = 'goal';
  state.goalSide = scoringSide;
  state.score[scoringSide]++;
  updateScoreBar();
  startGoalAnimation();
  broadcastState(); // 即座に状態を同期

  setTimeout(() => {
    if (state.score[scoringSide] >= CONFIG.WIN_SCORE) {
      state.phase = 'result';
      state.winner = scoringSide;
      broadcastState();
      showResult();
    } else {
      state.phase = 'playing';
      state.goalSide = null;
      resetPuck();
      stopGoalAnimation();
      broadcastState();
    }
  }, CONFIG.GOAL_ANIM_MS);
}
```

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat: physics engine - puck movement, wall reflection, mallet collision, goal detection"
```

---

### Task 7: タッチ操作

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `state.role`, `canvasW`, `canvasH`, `getNormCoords()`
- Produces: `initTouchHandlers()` — タッチイベント登録

- [ ] **Step 1: タッチハンドラー**

```js
// ========== TOUCH HANDLING ==========
let activeTouchId = null;

function clampMallet(nx, ny, side) {
  const padX = 10 / canvasW;
  const padY = 10 / canvasH;
  const mr = side === 'left' ? getMalletRadius('left') : getMalletRadius('right');

  // 自陣制限
  const minX = side === 'left' ? padX + mr : 0.5 + mr;
  const maxX = side === 'left' ? 0.5 - mr : 1 - padX - mr;

  return {
    x: Math.max(minX, Math.min(maxX, nx)),
    y: Math.max(padY + mr, Math.min(1 - padY - mr, ny)),
  };
}

function initTouchHandlers() {
  canvas.addEventListener('touchstart', onTouchStart, { passive: false });
  canvas.addEventListener('touchmove', onTouchMove, { passive: false });
  canvas.addEventListener('touchend', onTouchEnd, { passive: false });
  canvas.addEventListener('touchcancel', onTouchEnd, { passive: false });
}

function onTouchStart(e) {
  e.preventDefault();
  if (activeTouchId !== null) return;
  const t = e.changedTouches[0];
  activeTouchId = t.identifier;
  moveMalletFromTouch(t);
}

function onTouchMove(e) {
  e.preventDefault();
  for (const t of e.changedTouches) {
    if (t.identifier === activeTouchId) {
      moveMalletFromTouch(t);
      break;
    }
  }
}

function onTouchEnd(e) {
  for (const t of e.changedTouches) {
    if (t.identifier === activeTouchId) {
      activeTouchId = null;
      break;
    }
  }
}

function moveMalletFromTouch(touch) {
  const rect = canvas.getBoundingClientRect();
  const cx = touch.clientX - rect.left;
  const cy = touch.clientY - rect.top;
  const { x: nx, y: ny } = getNormCoords(cx * (canvas.width / rect.width), cy * (canvas.height / rect.height));

  const side = state.role === 'host' ? 'left' : 'right';
  const clamped = clampMallet(nx, ny, side);

  if (state.role === 'host') {
    state.malletHost.x = clamped.x;
    state.malletHost.y = clamped.y;
  } else {
    state.malletGuest.x = clamped.x;
    state.malletGuest.y = clamped.y;
    // ゲスト→ホストへ送信
    sendToHost({ t: 'mallet', x: clamped.x, y: clamped.y });
  }
}
```

- [ ] **Step 2: ゲームループへ組み込み**

initCanvas()呼び出し後に initTouchHandlers() を呼ぶ。`onConnectionEstablished` 内:
```js
function onConnectionEstablished() {
  showConnectStep('connected');
  setTimeout(() => {
    showScreen('game');
    initCanvas();
    initTouchHandlers();
    if (state.role === 'host') startGame();
    startGameLoop();
  }, CONFIG.CONNECTED_SHOW_MS);
}
```

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: touch handling for mallet control with half-court constraint"
```

---

### Task 8: ゲームループ + 補間

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `updatePhysics()`, `render()`, `broadcastState()`, `state`
- Produces: `startGameLoop()`, `stopGameLoop()`

- [ ] **Step 1: ゲームループ**

```js
// ========== GAME LOOP ==========
let animFrameId = null;

function lerp(a, b, t) {
  return a + (b - a) * t;
}

function startGameLoop() {
  if (animFrameId) cancelAnimationFrame(animFrameId);

  function loop() {
    animFrameId = requestAnimationFrame(loop);

    if (state.screen !== 'game') return;
    if (state.isPaused) return;

    if (state.role === 'host') {
      updatePhysics();
      broadcastState();
    } else {
      // ゲスト: パックをlerp補間
      state.puck.x = lerp(state.puck.x, state.puckTarget.x, CONFIG.LERP_ALPHA);
      state.puck.y = lerp(state.puck.y, state.puckTarget.y, CONFIG.LERP_ALPHA);
    }

    render();
  }

  loop();
}

function stopGameLoop() {
  if (animFrameId) {
    cancelAnimationFrame(animFrameId);
    animFrameId = null;
  }
}
```

- [ ] **Step 2: 一時停止ボタン**

```js
document.getElementById('btn-pause').addEventListener('click', () => {
  state.isPaused = !state.isPaused;
});
```

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: game loop with physics update, lerp interpolation, pause"
```

---

### Task 9: 得点演出 + 紙吹雪

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces:
  - `startGoalAnimation()` — 得点演出開始（pingリング、+1バッジ、紙吹雪）
  - `stopGoalAnimation()` — 演出終了

- [ ] **Step 1: ゴールオーバーレイ演出**

```js
// ========== GOAL ANIMATION ==========
CONFIG._goalFlashActive = false;
let confettiPieces = [];

function startGoalAnimation() {
  CONFIG._goalFlashActive = true;
  const overlay = document.getElementById('goal-overlay');
  overlay.classList.add('active');

  // ping ringとバッジの位置（得点されたゴールの近く）
  const ring = document.getElementById('ping-ring');
  const badge = document.getElementById('plus-one');
  if (state.goalSide === 'guest') {
    // ゲストのゴール = 右端ゴール
    ring.style.right = '5%';
    ring.style.left = '';
    ring.style.top = '50%';
    ring.style.transform = 'translate(50%, -50%)';
    badge.style.right = '18%';
    badge.style.left = '';
    badge.style.top = '24%';
  } else {
    // ホストのゴール = 左端ゴール
    ring.style.left = '5%';
    ring.style.right = '';
    ring.style.top = '50%';
    ring.style.transform = 'translate(-50%, -50%)';
    badge.style.left = '18%';
    badge.style.right = '';
    badge.style.top = '24%';
  }

  startConfetti(document.getElementById('confetti-layer'), 18);

  // キャラのhopアニメーション
  const scoringChar = state.goalSide === 'host'
    ? document.getElementById('char-blue')
    : document.getElementById('char-red');
  scoringChar.style.animation = 'hop 1s ease-in-out infinite';

  updateScoreBar();
}

function stopGoalAnimation() {
  CONFIG._goalFlashActive = false;
  document.getElementById('goal-overlay').classList.remove('active');
  stopConfetti(document.getElementById('confetti-layer'));

  document.getElementById('char-blue').style.animation = '';
  document.getElementById('char-red').style.animation = '';

  updateScoreBar();
}
```

- [ ] **Step 2: 紙吹雪生成**

```js
function startConfetti(container, n) {
  container.innerHTML = '';
  container.classList.add('active');
  const colors = ['#ff6b5e', '#3fc7a3', '#4ea8de', '#ffc233'];

  for (let i = 0; i < n; i++) {
    const div = document.createElement('div');
    const left = (i * 89) % 100;
    const size = 11 + ((i * 7) % 9);
    const delay = ((i * 0.41) % 3.2).toFixed(2);
    const dur = (2.4 + ((i * 0.33) % 2.0)).toFixed(2);
    const color = colors[i % colors.length];
    const radius = (i % 3 === 0) ? '50%' : '3px';
    const rot = (i * 37) % 360;

    div.style.cssText = [
      'position:absolute',
      `left:${left}%`,
      'top:-24px',
      `width:${size}px`,
      `height:${Math.round(size * 0.72)}px`,
      `background:${color}`,
      'border:2px solid #1a2238',
      `border-radius:${radius}`,
      `transform:rotate(${rot}deg)`,
      `animation:fall ${dur}s ${delay}s linear infinite`,
    ].join(';');

    container.appendChild(div);
  }
}

function stopConfetti(container) {
  container.classList.remove('active');
  container.innerHTML = '';
}
```

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: goal animation - ping ring, +1 badge, confetti"
```

---

### Task 10: 結果画面

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `state.winner`, `state.score`
- Produces: `showResult()` — 結果画面を組み立てて表示

- [ ] **Step 1: 結果画面表示関数**

```js
// ========== RESULT SCREEN ==========
function showResult() {
  stopGameLoop();

  const isHostWinner = state.winner === 'host';
  const winnerColor = isHostWinner ? '#4ea8de' : '#ff6b5e';
  const winnerCheek = isHostWinner ? '#ff9d90' : '#ffce6b';

  // 勝者キャラ（拡大）
  const charEl = document.getElementById('result-winner-char');
  charEl.innerHTML = `<div style="transform:scale(1.78);">${buildChar(winnerColor, winnerCheek, 72)}</div>`;

  // かち！の色
  const kachiEl = document.getElementById('result-kachi');
  kachiEl.style.color = winnerColor;
  kachiEl.classList.toggle('blue', isHostWinner);

  // スコアカード
  const leftCard = document.getElementById('result-card-left');
  const rightCard = document.getElementById('result-card-right');
  const numLeft = document.getElementById('result-num-left');
  const numRight = document.getElementById('result-num-right');
  const charLeft = document.getElementById('result-char-left');
  const charRight = document.getElementById('result-char-right');

  // 左=host(青), 右=guest(赤) 固定
  charLeft.innerHTML = buildChar('#4ea8de', '#ff9d90', 52);
  charRight.innerHTML = buildChar('#ff6b5e', '#ffce6b', 48);
  numLeft.textContent = state.score.host;
  numRight.textContent = state.score.guest;
  numLeft.style.color = '#4ea8de';
  numRight.style.color = '#ff6b5e';

  leftCard.classList.toggle('winner', isHostWinner);
  rightCard.classList.toggle('winner', !isHostWinner);
  numLeft.classList.toggle('loser', !isHostWinner);
  numRight.classList.toggle('loser', isHostWinner);

  // やさしいメッセージ
  const loserTeam = isHostWinner ? 'あかチーム' : 'あおチーム';
  document.getElementById('result-msg').textContent = `${loserTeam} つぎは がんばろう！`;

  // 紙吹雪
  startConfetti(document.getElementById('result-confetti'), 30);

  showScreen('result');
}

// もう1回ボタン
document.getElementById('btn-replay').addEventListener('click', () => {
  stopConfetti(document.getElementById('result-confetti'));
  showScreen('game');
  if (state.role === 'host') {
    startGame();
    startGameLoop();
    broadcastState();
  } else {
    startGameLoop();
  }
});
```

- [ ] **Step 2: 動作確認**

2タブで接続 → ゲーム → コンソールで `state.score.host = 5; onGoal('host')` を実行 → 結果画面が表示されることを確認。「もう1回」で再戦できることを確認。

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: result screen with winner character, scores, replay button"
```

---

### Task 11: ハンデシステム

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces: 接続画面の「えらぶ」ステップに「ハンデ設定」UIを追加

仕様: 接続確立後、ホスト側でハンデ設定（年下はどっち側か）を選べる。設定はゲスト側にも送信。

- [ ] **Step 1: ハンデUI追加（接続画面2aの下部）**

`step-choose` の `.connect-btns` の後に追加:
```html
<!-- ハンデ設定（2a画面の下部） -->
<div id="handicap-ui" style="padding:0 20px 16px;display:flex;align-items:center;gap:12px;justify-content:center;">
  <label style="font:800 14px 'M PLUS Rounded 1c';color:#1a2238;opacity:0.7;display:flex;align-items:center;gap:8px;cursor:pointer;">
    <input type="checkbox" id="handicap-toggle" style="width:20px;height:20px;">
    ハンデあり（年下は
  </label>
  <select id="handicap-side" style="font:800 14px 'M PLUS Rounded 1c';border:3px solid #1a2238;border-radius:10px;padding:4px 8px;background:#fff;">
    <option value="left">左（青）</option>
    <option value="right">右（赤）</option>
  </select>
  <span style="font:800 14px 'M PLUS Rounded 1c';color:#1a2238;opacity:0.7;">）</span>
</div>
```

- [ ] **Step 2: ハンデ設定の読み取りと送信**

```js
function getHandicapSettings() {
  const enabled = document.getElementById('handicap-toggle').checked;
  const side = document.getElementById('handicap-side').value;
  return { enabled, youngerSide: side };
}

// startGame() 内で呼び出す
function startGame() {
  state.handicap = getHandicapSettings();
  state.score = { host: 0, guest: 0 };
  state.phase = 'playing';
  state.winner = null;
  state.goalSide = null;
  resetPuck();
  updateScoreBar();
  // ハンデ設定をゲストへ送信
  sendToGuest({ t: 'handicap', ...state.handicap });
}

// ゲスト側: ハンデ受信
// conn.on('data') に追加:
// if (data.t === 'handicap') {
//   state.handicap = { enabled: data.enabled, youngerSide: data.youngerSide };
// }
```

- [ ] **Step 3: conn.on('data') のハンデ処理を追加**

`startHost()` の `conn.on('data')` コールバック内（ゲスト→ホスト受信）は変更不要。
`connectAsGuest()` の `conn.on('data')` に以下を追加:
```js
conn.on('data', (data) => {
  if (data.t === 'state') {
    onStateReceived(data);
  } else if (data.t === 'handicap') {
    state.handicap = { enabled: data.enabled, youngerSide: data.youngerSide };
  }
});
```

- [ ] **Step 4: コミット**

```bash
git add index.html
git commit -m "feat: handicap system - goal width and mallet size adjustment"
```

---

### Task 12: 縦持ち検知 + 最終ポリッシュ

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Safari固有の操作抑止（html/body に追加済みを確認）**

以下が `<html>` に含まれているか確認（Task 1で追加済み）:
```css
html, body {
  touch-action: none;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
  overflow: hidden;
}
```
また `<meta name="apple-mobile-web-app-capable">` も確認。

- [ ] **Step 2: ゴール演出のCanvasフラッシュ処理をdrawTable()に適用確認**

`drawTable()` 内でゴールフラッシュ色を使っているか確認:
```js
// 左ゴール色の決定
const leftFlashing = state.phase === 'goal' && state.goalSide === 'host';
ctx.fillStyle = leftFlashing ? '#ffe39a' : '#bfe0f5';
// 右ゴール色の決定
const rightFlashing = state.phase === 'goal' && state.goalSide === 'guest';
ctx.fillStyle = rightFlashing ? '#ffe39a' : '#ffc9c2';
```
これを `drawTable()` に反映（Task 3で `CONFIG._goalFlashActive` を使っているなら `leftFlashing`/`rightFlashing` に変更）。

- [ ] **Step 3: 縦持ちオーバーレイ改善**

絵文字を使わずCSSのみで回転アイコンを表示する（Safari互換）:
```html
<!-- screen-orientの内容を更新 -->
<div id="screen-orient" class="screen">
  <div style="width:60px;height:90px;border:6px solid #fff;border-radius:10px;animation:spin 2s linear infinite;display:flex;align-items:center;justify-content:center;">
    <div style="width:20px;height:6px;border-radius:3px;background:#fff;"></div>
  </div>
  <div>iPadを よこにしてね</div>
</div>
```

- [ ] **Step 4: エラーハンドリング完結確認**

以下のエラーケースが処理されているか確認:
- `peer-unavailable`: ゲスト接続時に「合言葉がちがうかも」表示済み（Task 5）
- `unavailable-id`: ホスト側の再試行実装済み（Task 5）
- `network`: `peer.on('error')` のフォールバックがあるか確認。なければ追加:
  ```js
  peer.on('error', (err) => {
    if (err.type === 'unavailable-id') { /* 再試行 */ }
    else if (err.type === 'peer-unavailable') { alert('合言葉が ちがうかも…'); }
    else { alert('ネットワークエラーが発生しました'); }
  });
  ```

- [ ] **Step 5: 受け入れ基準チェックリスト**

```
□ 2台のiPadで接続できる（合言葉方式）
□ 両画面で同じ位置にパックが見える
□ 各自のマレットを動かせ、相手画面にも反映される
□ パックを打ち合えてゴールでスコアが入る
□ 5点先取で結果画面。「もう1回」で再戦
□ 縦持ちで「横にしてね」表示
□ 切断で「あいてがきれました」表示
□ ハンデON/OFFでゴール幅・マレットサイズが変わる
```

- [ ] **Step 6: 最終コミット**

```bash
git add index.html
git commit -m "feat: complete airhockey game with PeerJS, canvas rendering, handicap system"
```

---

## スペック vs デザイン 整合性チェック結果

| 項目 | spec.md | design.md | 対応 |
|---|---|---|---|
| 勝利点 | 5点（定数化） | デモは7-5表示 | 定数5点で実装、デモは静的 |
| ゴール幅 | 左右別定数 | 50%×50px | 50%を正規化ベースに変換して実装 |
| 台描画 | Canvas | CSS | specに従いCanvas使用 |
| スコア表示 | DOM外 | DOM | 両方同じ、DOM実装 |
| 接続方式 | room-XXXX形式 | 4桁あいことば | air-XXXX形式で実装 |
| キャラクター | 指定なし | CSS多図形 | CSS多図形で実装 |
| アニメーション | 指定なし | @keyframes詳細あり | design.mdに従う |
