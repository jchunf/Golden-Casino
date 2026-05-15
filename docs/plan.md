# Golden Casino 开发计划

## 一、项目定位

一个 **多人联机** 的网页端 **赌场派对游戏**,灵感来自 Steam《Gamble With Your Friends》:
- 朋友之间开房间一起玩
- 同一桌可以看到其他玩家的余额、下注、动作
- 多种经典赌场玩法集合在一个客户端里

核心体验:**链接发出去就能一起玩,不需要装客户端、不需要注册账号**。

## 二、首期目标(MVP)

打通 **房间 + 一个多人玩法** 的闭环,而不是先做单机老虎机。

### 玩家流程
1. 进入首页,输入昵称 → 创建房间(拿到房号 / 分享链接) 或 加入房间
2. 大厅里看到所有在线玩家、各自的金币余额
3. 房主选择一个游戏开始(MVP 阶段只开放 **多人 21 点 Blackjack**)
4. 一局结束后,大家回到大厅,可以再开下一局或切换游戏

### MVP 玩法:多人 21 点(Blackjack)
- 一桌最多 4 名玩家 + 1 个 AI 庄家，玩家数量不够时补AI玩家。
- 开局每人下注(从自己的金币里扣)
- 庄家发牌,玩家依次决定"要牌 / 停牌 / 加倍"
- 全员结束后庄家亮牌补牌,按 21 点规则结算
- 输赢直接加减各自余额,实时同步给所有人

### 初始金币
- 每个玩家进入房间默认 **1000 金币**
- 输光后可以"乞讨"(其他玩家自愿转账)或退出房间重进重置

### 操作
- 支持键盘操作
- 下注默认下好最小的注，四分之一、二分之一、All In

### 爽感强
- 金币喷射动画
- 粒子特效
- 连胜提示
大赢时或连胜：
- 屏幕短暂金色泛光
- UI 放大弹跳
- 数字滚动增长
- 全桌广播

## 三、参考产品

Steam《Gamble With Your Friends》:
- 多人房间制,链接邀请朋友
- 包含 21 点、骰宝、轮盘、抛硬币、小型老虎机等
- 玩家之间能看到下注金额、表情动作,有"嘲讽"互动
- 卡通画风,节奏快,单局时长短

我们 **不复刻**,只参考它的"房间制 + 多游戏集合 + 朋友互动"这套核心结构。

## 四、技术栈

| 类型 | 选型 | 原因 |
|---|---|---|
| 前端 | **React + Vite + TypeScript** | 状态多、UI 复杂,原生 JS 维护成本高 |
| 实时通信 | **Socket.IO** (WebSocket) | 房间广播、状态同步,生态成熟 |
| 后端 | **Node.js + Express + Socket.IO** | 跟前端同语言,部署简单 |
| 状态存储 | **内存** (Map<roomId, RoomState>) | MVP 阶段不接数据库,服务重启房间就清空 |
| 鉴权 | **昵称 + 房间号**,无密码无注册 | 派对游戏定位,降低进入门槛 |
| 部署 | 前端 GitHub Pages / Vercel,后端 Render / Railway 免费层 | 零成本上线 |

## 五、文件结构

```
Golden-Casino/
├── docs/
│   ├── plan.md
│   └── pull_request_template.md
├── client/                     # 前端
│   ├── index.html
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── pages/
│       │   ├── Home.tsx        # 输入昵称 / 创建 / 加入房间
│       │   ├── Lobby.tsx       # 房间大厅
│       │   └── Blackjack.tsx   # 21 点桌面
│       ├── components/
│       │   ├── PlayerCard.tsx
│       │   ├── Card.tsx        # 扑克牌
│       │   └── ChipStack.tsx
│       └── socket.ts           # Socket.IO 客户端封装
├── server/                     # 后端
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts            # Express + Socket.IO 启动
│       ├── rooms.ts            # 房间管理
│       ├── games/
│       │   └── blackjack.ts    # 21 点状态机
│       └── types.ts            # 客户端共享的类型
├── shared/                     # 前后端共享类型
│   └── protocol.ts
├── LICENSE
└── README.md
```

## 六、Socket.IO 事件协议(初稿)

### Client → Server
| 事件 | 数据 | 说明 |
|---|---|---|
| `room:create` | `{ nickname }` | 创建房间,返回房号 |
| `room:join` | `{ roomId, nickname }` | 加入房间 |
| `room:leave` | - | 离开房间 |
| `game:start` | `{ game: "blackjack" }` | 房主开局 |
| `blackjack:bet` | `{ amount }` | 下注 |
| `blackjack:action` | `{ action: "hit" \| "stand" \| "double" }` | 玩家操作 |
| `chat:send` | `{ text }` | 聊天 |

### Server → Client
| 事件 | 数据 | 说明 |
|---|---|---|
| `room:state` | `RoomState` | 全量房间状态(玩家列表、余额、阶段) |
| `game:state` | `GameState` | 当前游戏的状态机快照 |
| `game:event` | `{ type, payload }` | 增量事件(发牌、玩家行动等),用于触发动画 |
| `chat:message` | `{ from, text, ts }` | 收到聊天 |
| `error` | `{ message }` | 错误提示 |

## 七、实现步骤

1. **脚手架**:`client/` 用 Vite 起 React+TS,`server/` 起 Node+TS+Socket.IO,本地 hello world 跑通
2. **房间系统**:创建房 / 加入房 / 离开房 / 玩家列表广播
3. **大厅 UI**:玩家卡片、余额、"开始游戏"按钮
4. **21 点状态机**(server 端):发牌、要牌、停牌、结算的纯函数
5. **21 点 UI**(client 端):桌面、扑克牌、下注、操作按钮
6. **同步与动画**:`game:event` 触发发牌 / 翻牌动画
7. **聊天 + 表情**:简单文本聊天框
8. **部署**:前端 Vercel,后端 Render,跨域配置
9. **README**:玩法说明 + 在线地址 + 部署指南

## 八、后续可扩展(非 MVP)

- 增加游戏:轮盘、骰宝、抛硬币、多人老虎机
- 旁观模式(房间满了也能看)
- 表情 / 嘲讽动作
- 房间密码、踢人、转让房主
- 数据库持久化金币 + 排行榜
- 移动端适配
- 音效与背景音乐

## 九、风险与权衡

- **后端免费层冷启动**:Render 免费实例闲置 15 分钟会休眠,首次连接慢 → 接受
- **作弊**:所有牌堆和随机数都在 server 端生成,client 不参与决策,避免改前端作弊
- **断线重连**:MVP 阶段断线即离桌,后续再做重连
- **同房同名**:加入时检查昵称冲突,要求换一个

## 十、待确认事项

- [ ] MVP 玩法选 **多人 21 点** 是否 OK?(也可改成 多人骰宝 / 多人轮盘)
- [ ] 技术栈接受 **React + Node + Socket.IO** 吗?
- [ ] 后端部署平台有没有偏好?(Render / Railway / Fly.io / 自己服务器)
- [ ] 是否要加 **聊天功能**?
- [ ] 单房最大人数 **4 人** 够吗?
