# NEON SNAKE 🐍 — 霓虹貪吃蛇

A modern reimagination of the classic Snake game, built with Vue 3 + TypeScript + Canvas.

## Design Concept

When I was a child, I played many Snake games with different modes—versus battles, stage challenges, boss fights. Those memories inspired me to build a Snake game that brings all those experiences together in one place. My core idea was simple: **what if Snake could be more than just eating food?**

I wanted players to be able to compete head-to-head with friends in a split-screen duel, invade each other's territory through portals, and set traps to outsmart their opponent. I wanted a boss fight where a giant AI snake hunts you down and you have to fight back with missiles. I wanted stage progression where each level gets harder, faster, and more treacherous. And when there's no one around to play with, an AI opponent should be there to keep the challenge alive.

Every mode is designed to offer a different flavor of Snake—whether you want the classic chill experience, an intense PvP showdown, or a PvE boss battle. The game uses a neon cyberpunk aesthetic with glowing visuals, particle effects, and responsive canvas rendering to create an immersive arcade feel. My goal was to make a Snake game that I would want to play myself, with enough variety that you never get bored.

## How My Version Is Different

傳統的貪吃蛇很單純——控制方向、吃食物、不要撞牆。我的版本保留了這個核心玩法，但在上面疊加了**四種完全不同的模式**：

| 模式 | 差異點 |
|------|--------|
| **經典模式** | 加入 6 種能力道具（護盾、幽靈、雙倍分數、縮短、緩速、陷阱），還有 Combo 連擊系統，得分方式更有策略性 |
| **雙人對戰** | 左右分屏 1v1 對戰！每人 3 條命，穿越傳送門入侵對手場地，放陷阱陷害對方，擊殺對手可偷取生命。支援 AI 電腦對手 |
| **Boss 戰** | 分數達到 15 分時，一條 12 節的巨型 Boss 蛇會現身追殺你！你可以發射飛彈反擊，擊敗 Boss 可獲得大量加分 |
| **關卡遊戲** | 連續挑戰 20 關，每關需達到目標分數，完成後會出現黑洞傳送門，鑽進去晉級。後面關卡出現陷阱阻擋，速度越來越快 |

此外還有**商店系統**可以用金幣購買開局道具，和**回放功能**可以重播每一局的過程。

## How To Play

### 一般操作
| 按鍵 | 功能 |
|------|------|
| ⬆ ⬇ ⬅ ➡ | 控制蛇的方向 |
| ESC / P | 暫停 / 繼續 |
| SPACE | Boss 模式發射飛彈 |

### 能力道具（經典模式）
道具會隨機出現在場地上，蛇頭經過即可獲得：

| 圖示 | 名稱 | 效果 |
|------|------|------|
| 🛡 | SHIELD | 抵擋一次碰撞（牆壁或自身） |
| 👻 | GHOST | 穿越牆壁，從另一側出現（32 步） |
| 💎 | 2×SCORE | 所有得分翻倍（40 步） |
| ✂️ | SHRINK | 蛇身立即縮短一半 |
| ⏱ | SLOW-MO | 移動速度減慢（38 步） |

### 雙人對戰操作

**P1（左側）**：⬆ ⬇ ⬅ ➡ 移動，**空白鍵** 發動技能

**P2（右側）**：**W A S D** 移動，**0 鍵** 發動技能

雙人技能道具：
| 圖示 | 名稱 | 效果 |
|------|------|------|
| 🌀 | 傳送門 | 穿越到對手場地，入侵吃對方的食物 |
| 🗑 | TRAP | 在對手場地放置永久陷阱，踩到即死 |
| 🧹 | CLEANSE | 清除自己場地上所有陷阱 |

其餘能力道具（護盾、幽靈、雙倍分數等）在雙人模式中也可使用。

### 雙人對戰規則
- 每人 3 條命
- 擊殺對手可偷取對方一條命
- 入侵對方場地可搶吃食物
- 率先耗盡對方生命者獲勝
- 可選擇與 AI 電腦對戰

### 商店
在選單下方點擊 🏪 商店，使用闖關賺來的金幣購買開局道具（護盾、緩速、雙倍分數、額外生命）。

---

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```

### Run Unit Tests

```sh
npm run test:unit
```

### Lint

```sh
npm run lint
```
