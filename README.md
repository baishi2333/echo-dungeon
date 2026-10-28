# 回声地牢：命运之喉

> 一款用 **声音** 施法的中世纪暗黑 Roguelike 卡牌游戏。
> 对着麦克风念出卡牌的名字，火球就会飞出去——念错，火球不会。

Godot 4.7.2 · GDScript · 单机离线 · Windows

---

## 这是什么

30 天，3 章，8 个职业，200 张技能卡，102 件装备，19 种敌人。

核心机制有三层：

1. **语音施法** —— 念卡名代替点击。念错不惩罚（不扣能量、不进冷却），只是打不出去。
2. **变声回放** —— 你念的咒语会按职业音色变调后放回来。战士低沉、法师清亮、恶魔带混响。
3. **心绪轴** —— 麦克风的音量和音高方差驱动"冷静 ↔ 狂暴"，狂暴时伤害高但受伤也高。

不是语音助手式的"说'攻击'就攻击"，是**要你真的念出那张牌的名字**。

---

## 快速开始（玩）

1. 到 [Releases](../../releases) 下载最新 `回声地牢verX.X.zip`
2. **解压整个文件夹**（不要只拖 exe 出来，`asr/` 必须和 exe 放一起）
3. 双击 `EchoDungeon.exe`

需要 Windows 10/11 + 麦克风。没有麦克风也能玩，念咒会降级成按音量判定。

**不需要安装 Python** —— 语音识别已打包成独立 exe。

---

## 快速开始（开发）

### 环境

| 项     | 版本                            |
| ----- | ----------------------------- |
| Godot | **4.7.2**（低版本跑不了，用了 4.4+ 的语法） |
| 系统    | Windows 10/11                 |
| 可选    | Python 3.10+（只在改语音识别脚本时需要）    |

### 跑起来

```bash
git clone https://github.com/<你的用户名>/echo-dungeon.git
cd echo-dungeon

# 资源不在仓库里，先按下方「资源怎么来」补上

# 用 Godot 打开
Godot_v4.7.2-stable_win64.exe --path . --editor
```

### 调试入口

```bash
EchoDungeon.exe -- --dev-test              # 40 项自检
EchoDungeon.exe -- --dev-mictest           # 麦克风 3 秒自检
EchoDungeon.exe -- --dev-sim               # 8 职业平衡模拟（约 3 分钟）
EchoDungeon.exe -- --dev-battle warrior    # 直接进战斗
EchoDungeon.exe -- --dev-class             # 直接进选人
EchoDungeon.exe -- --dev-codex             # 直接进图鉴
```

---

## 资源怎么来

仓库里**不含** `assets/art/`（832MB 图片）和 `tools/models/`（66MB 语音模型），因为 Git 扛不住。

**办法 A：下载资源包（推荐）**

到 [Releases](../../releases) 找 `assets-art.part*.rar`，解压到项目根目录，得到 `assets/art/`。

**办法 B：自己生成**

```bash
# 图片：按 data/*.json 里的 portrait 字段描述，用任意 AI 生图工具生成
# 语音模型：Vosk 中文小模型
pip install vosk
python -c "import vosk, os; os.makedirs('tools/models', exist_ok=True); vosk.Model(lang='cn')"
```

模型放 `tools/models/vosk/`，目录结构要和 `tools/asr_vosk.py` 里读的一致。

---

## 项目结构

```
echo-dungeon/
├── project.godot
├── scripts/
│   ├── main.gd                屏幕路由 + 调试入口
│   ├── game_data.gd           数据加载 / 存档 / 卡组 / 背包
│   ├── ui.gd                  设计系统（颜色 / 字号 / 组件工厂）
│   ├── gothic_button.gd       自绘切角按钮
│   ├── mic_chant.gd           麦克风采集 + 心绪轴 + ASR 调度
│   ├── settings_panel.gd
│   ├── battle/
│   │   ├── engine.gd          战斗引擎（纯逻辑，无 UI）
│   │   └── ...
│   ├── screen_title.gd
│   ├── screen_class_select.gd
│   ├── screen_battle.gd
│   ├── screen_map.gd
│   ├── screen_event.gd
│   ├── screen_bag.gd
│   ├── screen_codex.gd
│   └── selftest.gd            40 项自检
├── data/                      JSON 数据（技能 / 装备 / 敌人 / 事件 …）
├── tools/
│   ├── asr_vosk.py            语音识别脚本
│   └── models/vosk/           ← 需自行下载
└── assets/                    ← 大部分需自行下载
```

**UI 全部代码生成**，`.tscn` 只留空壳——改布局不用开编辑器，也方便 diff。

---

## 几个技术决定

### 语音识别不惩罚失败

Web Speech API 必须联网，这游戏是单机的，用不了。改用 **Vosk 离线识别**。

识别不准怎么办？**不设置信度阈值、不加冷却**。策略是：

- 关键词包含匹配（念"火球"能匹配"火球术"）
- 念错就念错了，卡牌不出，但**不扣任何资源**
- 玩家可以随时点击施法，语音是加成不是枷锁

### 变声器

`AudioEffectPitchShift` + `AudioEffectCapture`，纯本地、延迟可控。

每个职业一套音色参数（基频倍率、混响、失真），8 职业就是 8 种声音。

### 心绪轴

麦克风的 RMS 音量 + 音高方差 → 两个浮点数 → 冷静 / 狂暴。

**关键**：采集时必须关掉 `echoCancellation` / `noiseSuppression` / `autoGainControl`，
否则音频驱动会把音量动态抹平，心绪轴就死了。

### 数据驱动

技能 / 装备 / 敌人 / 事件 / 关卡全在 `data/*.json`，改数值不用碰代码。

`data/enemies.json` 里带了完整的平衡基准：

- `player_curve`：7 个节点的玩家 HP / DPS 成长
- `ttk_target`：杂兵 2-3.5 回合、精英 4.5-6、Boss 9-12.5 的目标击杀回合

---

## 已知限制

- 部分敌人技能（召唤、全体伤害、额外行动）尚未实现
- 9 类装备 / 敌人被动只接了数值部分
- 语音识别有约 3 秒延迟（Vosk 整句识别的固有代价）
- 中文没有真正的 Blackletter 字体，标题是做成图的

---

## 授权

代码 **MIT**，美术资源 **CC BY-NC 4.0**。

代码你随便用、改、商用都行；图不能拿去卖。

详见 [LICENSE](LICENSE)。

---

## 致谢

- [Godot Engine](https://godotengine.org/)
- [Vosk](https://alphacephei.com/vosk/) 离线语音识别
- 字体：思源宋体 / 思源黑体（SIL OFL）
