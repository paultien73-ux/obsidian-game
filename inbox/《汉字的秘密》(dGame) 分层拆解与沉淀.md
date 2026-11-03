
> **拆解对象**：`apk-src/dgpk-extracted/`（APK 内 web 资源，`MainGame.js` 29504 行 / 253 类，TypeScript→JS 未混淆）  
> **包名**：com.createmaster.dgame.Hanzidemishi.mi（密室逃脱·汉字的秘密）  
> **拆解方法**：按《[密室逃脱分层建模.md](http://xn--kbr47u5cowy2ip1wy71bg1ya.md/)》第十一章 L5→L0 自顶向下逆向映射，所有结论均给出代码证据位置  
> **产出物**：L5/L4 总览（§1-2）、门禁序列模板库（§3）、谜题卡片库（§4）、母题统计表（§5）、交互原语表（§6）、设计模式笔记（§7）、再创作指南（§8）

---

## 1. L5 游戏层：产品结构

### 1.1 技术形态

|维度|事实|证据|
|---|---|---|
|引擎|自研 dEngine（dSprite/dImage/dButton/dSound/CCAction 动作系统，cocos 风格）|MainGame.js 27-2587|
|UI|声明式 JSON `_dscData()` + `dControlLoader.CreateUIByJson`|全部类构造函数|
|数据|JsonExcel 表（`data/configs/talk.json`），TalkEntry.Query(text)→sound|6625-6642|
|存档|SaveData 单例 JSON（`{AppName}_savedata`），1 秒防抖写盘|4149-4238|
|经济|energy=20 体力、keyCount 提示钥匙、gem、vip、edgeRewarded（看广告边角奖励）|4155-4164|
|广告|MyAd.hasMoreGame / MoreGamePanel / EdgeRewardPanel；edgereward*.json 为"获得奖励"字模图集|5014+|

### 1.2 页面流

```
LoginPage → Main(菜单) → SelectLevelPage(选关) → GamePage(GameWorld)
                                    ↑ back            │ hint: HintPanel(GetHintText)
                                                      └ clear: StageClearPanel → 下一关
```

- GameWorld 构造：`LoadLevel(StageData.GetStageByIndex(stageData.stage - 1))`（5990）——**存档存的 stage 是"第几个关卡位"，再经 StageIndex 映射到 Level 类**
- `StageData.StageIndex = [12,1,2,11,10,3,4,9,5,6,7,8,13,14,15,16,17,18]`（6358）  
    → 玩家顺序 ≠ 关卡编号：**先玩 12(山间小路，轻量教学) → 1 → 2 → 11 → 10 → 3/4/9/5/6/7/8 → 13-18 顺序解锁**  
    → 设计意图：把最简单的"山间小路"提为第 1 关做无门槛教学；13-18（密语博物馆…遗忘的车站）是后期高密度连续章节

### 1.3 关卡编号 ↔ 主题 ↔ 难度曲线

|关|主题|关|主题|
|---|---|---|---|
|1|破旧的房间（教学）|10|古老的公寓（汉字笔划密码）|
|2|实验室（数学推导）|11|废弃的公园（拖放方位）|
|3|灰暗的仓库（方向编码）|12|山间小路（呈现顺序第1）|
|4|阴暗地下室（星空望远镜）|13|密语博物馆（高密度双密码）|
|5|古老的山洞（代数密码）|14|神秘的墓穴（8+ 门禁巨关）|
|6|废弃的船（逻辑排序）|15|错乱的钟塔（多阶段 step）|
|7|废弃的医院（生日+体重）|16|迷雾村庄（13 位序列锁）|
|8|破旧的工厂（自定义算术）|17|海上监狱（重复材料收集）|
|9|旧校舍（紫光照试卷）|18|遗忘的车站（涂鸦 step）|

### 1.4 谜题系统的三级构造（本作最核心的架构）

```
Level 类 = 场景图(_dscData) + 门禁编排(Init) + 提示链(GetHintText)     ← L3
Puzzle* 类 = 弹出式解谜面板（lockName 门禁标识）                        ← L1/L2
GameBarBase 家族 = 场景节点/物品/输入原语                               ← L0
```

---

## 2. L4 场景层：叙事与资产

### 2.1 场景即图结构（本作标志性设计）

- 每个场景是 **GameBar 节点图**：`_dscData._child` 声明节点（Text 名称 + Talk 独白 + Icon 图标 + Lock 锁标），`_linkChild = "Bar1,Bar2,Bar1,Bar3,..."` 声明邻接表（Level1: 7722）
- `LevelBase.InitBar` 递归图遍历建立双向 neighbor（7471-7490）；`GameBarBase.m_line`（lineWidth=22，色 0xffad9e8a）**把图的边画成实线**（5112-5115, 5319-5327）
- 点击节点 → `ShowNeighbor()`：邻居从父节点位置**依次弹出**（0.05s 阶梯延迟 + button.mp3）（5281-5313）——导航即"展开子图"
- 拖动松手 → `CollectBar` 碰撞检测按距离排序取最近 → `OnCollectBar` 物品合成判定（7516-7602）
- **场景可拖动平移 + 双指缩放（0.5-2x 带边界钳制）**（GameWorld.OnMultyTouchMove）

### 2.2 叙事数据流

- talk.json：`[{name, sound, text}]`，如 `{"name":"撬棍","sound":"108","text":"一根撬棍，可以用它撬开什么"}`
- 点击 Bar → ShowTalk：顶栏 gametitle.png 弹出文本（FadeIn 0.3→停留 3s→FadeOut），若 enableSpeak 则播 `level2/a{sound}.mp3` 语音（5996-6017）
- 解锁后文案：`SetUnlockNeedBarName(..., unlockSelfText, unlockSelfTalk)` → 解锁瞬间换 Text/Talk（7576-7579）

### 2.3 音效表（data/sound/，17 个，全部显式调用）

|音效|触发点|音效|触发点|
|---|---|---|---|
|button/button2|按钮/Bar 点击|opendoor/openshimen|开门/开石门（SetUnlockNeedBarName 第6参 unlockSound）|
|lock|点锁定的 Bar（伴随 Lock 摇晃 CCShake）|openswitch/2, openlight|机关开关|
|unlock|物品合成/密码通过（PlayUnlock 缩放脉冲）|bolisui|锤子砸玻璃柜（Level13）|
|openpaper|谜题弹窗开/关|clockrun|钟塔|
|right/wrong/failed|对/错/失败|window|Level3 窗|
|game|背景乐|—|—|

### 2.4 资源编号规范

|前缀|内容|规律|
|---|---|---|
|level/|场景底图+物品图标（523 文件）|1-9 关 `N01-N99`（关号×100）；10-18 关 `stageId*1000+item`（如 1019=关10 物品19，1441=关14）|
|level2/|语音配音 a{sound}.mp3（522 文件）|sound 编号与 talk.json 对应|
|level3/|谜题 UI 素材 + 全局线索纸（90 文件）|`90XX` 线索纸跨关共用（9028/9031/9036…）|
|data/|通用 UI/字体 font64/音效/拼图符号 puzzleimg|—|

**状态即换图**：解锁常伴随 `unlockIconFileName` 换 Icon（"蜡烛"→1019.jpg"点燃的蜡烛"）；场景阶段切换用 `SetBackgroudFileName`（6047）。

---

## 3. L3 序列层：门禁编排（序列模板库）

### 3.1 门禁 API 全集（GameBarBase 5079-5530）

|API|签名|语义|
|---|---|---|
|`SetLock(1)`|(isLock, playSound=true)|锁定；Lock 子节点显示；解锁播 unlockSound/默认 unlock.mp3 + 缩放脉冲|
|`SetUnlockNeedBarName`|(names[], unlockSelfText, unlockSelfTalk, isRemoveItem=true, unlockIconFileName, unlockSound)|**物品门禁**：把指定名称的 Bar 拖到本 Bar 上；names.length>1 时生成 GameBarDotPanel 多色收集点；m_bUnlockRemoveItem=false=工具不消耗|
|`SetInputPasswordBars`|(bars[], pass, unlockSelfText, unlockSelfTalk, unlockIconFileName, unlockSound)|**密码门禁**：绑定输入 Bar 列表，GetText() 拼接 == pass|
|`SetClickCallback`|(cb)|点击回调（弹出 Puzzle/线索/状态机）|
|`SetLockCallback`|(cb)|锁状态变化回调（解锁瞬间触发连锁）|
|`m_bUnlockNeedOrder`|SetUnlockNeedOrder|**顺序敏感**：多物品须按 names 顺序投放，否则全部弹回重排（7542-7574）|
|`SetDotText(str, color)`|PasswordBar|自定义密码符号（"■●▲"），L10 出口 5527|

### 3.2 门禁原语分类（18 关归纳）

1. **物品消耗型**：钥匙→出口、锤子→玻璃柜（isRemoveItem=true，消耗）
2. **工具转化型**：剪刀→窗帘（isRemoveItem=false，剪刀保留）+ 换图成"一块抹布"
3. **多材料收集**：钟表←[钟摆,指针,表盘]（DotPanel 点亮）；厕所隔间←[小铁片,小铁片,长柄铁勺]（**同名重复**，靠 m_vecUnlockTriged 实例去重）；两块"一块玉石"→"完整的玉石"（同名互拼）
4. **顺序敏感链**：m_bUnlockNeedOrder（L11 出口符号拖放四角：三角→左上、圆→右上、星→左下、菱→右下）
5. **密码型**：PasswordBar 拼接（"502"/"5527"/"135921"）；ButtonBar 滑窗（m_inputOrder，多退少补）
6. **点击探索型**：SetClickCallback → ShowPuzzle（线索纸/交互谜题）
7. **状态机型（多阶段）**：`SetStep(step)` → `OnClear(step)` 重放关卡状态（L15 沙漏开关→时空轮盘；L16 地下湖水位三次涨落；L18 涂鸦显隐）
8. **中间锁联动**：石棺 Pass9 点击→巨大石门 SetLock(0)（L14）；m_onAddUnlock/m_lockCallback 链式触发

### 3.3 18 关门禁编排速查表

|关|密码锁|物品门禁（→主链）|特色谜题|
|---|---|---|---|
|1 破旧的房间|日记本=502|撬棍→垃圾桶；电源→电脑；钥匙→出口|PuzzleLink 连线（Level1 出口前锁）|
|2 实验室|屏幕=1255(时钟 12:55)；出口=168|指纹膜+ID卡→仪器房；电源线→电脑；培养样本→显微镜|**A=26,B=74,C=68→A+B+C=168**；白笔桌布|
|3 灰暗的仓库|地下室=3121；出口=3285|梯子→货架；撬棍→地板；剪刀→窗帘(323 抹布)→擦墙|**↑3↓2←8→5 方向映射**；364=5 圈数题|
|4 阴暗地下室|抽屉=8734；黑箱=937|电池→手电筒(429)→照过道；望远镜→星空|PuzzleStarrySky 望远镜找星；喷涂油漆显字|
|5 古老的山洞|石门=4515|铲子→苔藓；光线→晒木→打火石→火把→烧蛛网|**A=3,B=7,C=8 → B-A,C-A,B+C=4515**|
|6 废弃的船|瓶子=3218；电子设备=1203|水管→水龙头(605)→灭门火；氧气瓶→淹水走廊|**不等式排序"红>绿,黄>蓝…"**；机关盒←齿轮+发条|
|7 废弃的医院|设备箱=**8068+70=8138**(生日+体重)|温度计+蓝注射器→化学仪器；黄灯→照 X 光|药盒颜色分类；电子门 4 位|
|8 破旧的工厂|电源箱=4648|钥匙→车间；切割机→木材→梯子→通风口|**32+44=26 自定义算术**（铭牌 9043 给规则）|
|9 旧校舍|实验室=135921；暗门=5391|黑卡→光学仪器→紫照试卷；钥匙→图书室|PuzzleFindKey 镜下找钥匙|
|10 古老的公寓|出口=5527(■●▲符号)；茶几=**424(六七五笔划)**|壁炉→蜡烛(1019)→照书籍|画中"六七五"笔划数→数字（主题贴合！）|
|11 废弃的公园|控制室=726；出口=拖符号四角|小水池→洗石→亭子显字；钥匙→暗门|四角方位拖放（提示 9035）|
|12 山间小路|日记本(Puzzle102)=427；箱子=427|钥匙→出口|最小教学关|
|13 密语博物馆|时光之门=423；控制室=3028；电子设备=3102|锤子→玻璃柜(bolisui)；ID卡→面板；照明灯→展厅|**双线并行**；ImageBar 心/圆/星/方|
|14 神秘的墓穴|藏书阁=243；石棺=1459；出口=52853|宝珠→神兽像；玉石+玉石→完整玉石→宝盒；轮盘→祭坛；符牌→密室|8+ 门禁巨关；石棺 Pass9→石门|
|15 错乱的钟塔|密码箱=6735|齿轮→机关(1518)；钥匙→镜柜；宝石→凹槽；钟摆+指针+表盘→钟表|**多阶段**：双沙漏→时空轮盘|
|16 迷雾村庄|图腾=**1111112222211**；石门=022101；密码箱=3642|种子→土壤→花→光影；火种→油灯(1617)→隧道；绳子→古井|13 位 1/2 长序列；地下湖水位 step×3|
|17 海上监狱|库房=635；排水口=3674|磁铁→锈钳(1740)→管道；潜水设备→井盖；腐肉→鱼群|**同名×2 小铁片**收集|
|18 遗忘的车站|—（以物品链为主）|ID卡→驾驶室；按钮→电控(1803)；钥匙→商务车厢|涂鸦 SetLock step 显隐|

### 3.4 序列模板库（可直接复用的 4 种编排）

```
模板A_线性主链:            # 大多数关卡的骨架
  结构: 出口门禁 ← 物品A ← 工具B ← 线索C ← 探索节点
  例: L5 出口4515 ← 石门规则 ← 符号纸 ← 火把照亮
模板B_并行汇流:            # L13/L14：两三条独立支线汇合到出口
  结构: (支线1: 密码A) + (支线2: 物品B) + (支线3: 换图区C) → 出口
模板C_顺序敏感:            # L11 出口：同 4 物品但顺序即答案
  结构: SetUnlockNeedOrder + DotPanel 颜色反馈 + 弹回重排
模板D_多阶段step:          # L15/L16/L18：一关多幕
  结构: SetStep(n) → OnClear(n) 重放 Init 分支 → 场景状态机
```

### 3.5 提示系统（分级 + 资源引用语法）

- `GetHintText()` 按"当前第一个锁着的门禁"逐级返回（LevelBase 子类覆写）：
    
    ```
    if (FindBarByTextSafe("垃圾桶").isLock()) return "用撬棍打开垃圾桶盖子";
    if (FindBarByTextSafe("电路板").isLock()) return "[9028";   // [ 前缀 = 提示图引用
    return "使用钥匙打开出口的大门";
    ```
    
- `"[90XX"` → HintPanel 加载 `level3/90XX.png` 线索图；文字提示直达答案（424 直接写明）
- 提示消耗 keyCount（SaveData），广告可补充

---

## 4. L2 谜题卡片库（17 个 Puzzle 类）

> 基类 `PuzzleBase(lockName, onClear)`（6643-6731）：lockName=门禁标识（GameWorld.m_arrPuzzleBuffer 按名缓存，重开不重置），OnPassWordRight() 统一出口 → ShowClearPuzzle(lockName) → unlock.mp3 → Level.OnUnlock(lockName)。弹窗动画：从起点按钮位置缩放 0→1 飞入（6037-6044）。

```
- id: PuzzleShowPaper        # 使用频率最高（每关 2-4 次）
  类: 线索纸展示 | 行: 7318-7331
  输入: [查看] ; 判定: 无（纯信息）
  实现: paper.png 底 + Image.LoadFromFile("level3/90XX.png") + "线索"标题
  母题: observe 观察 ; puzzleName 按文件名缓存

- id: Puzzle102              # 多行文字线索
  类: 文字线索纸 | 行: 6744-6755
  实现: paper.png + dBitmapFont(autoWrap) 显示任意文本（数学题/规则）
  例: "A组实验结果26\nB组实验结果74\nC组实验结果68"
  母题: observe + encode 编码规则传递

- id: Puzzle101
  类: 密码纸 | 行: 6732-6743
  实现: paper.png + 大号单行文本（直接给密码）
  母题: observe

- id: PuzzleLink             # L1 出口锁
  类: 连线匹配 | 行: 6874-7015
  输入: [拖拽画线] ; 判定: 上排4图案 ↔ 下排4答案(Image8,7,5,6)，线色=起点色
  机制: dLine 手绘线 + isImageUp 上下行约束 + ClearLine 重连
  预置: AddLine(Image2, Image5) 部分已连（教学/降难）
  母题: associate 关联匹配

- id: PuzzleNumCalculator    # 计算器键盘
  类: 数字键盘输入 | 行: 7016-7075
  输入: [数字键+退格+OK] ; 判定: m_inputText==rightPass，错则 CCShake
  参数: num 自定义键位映射 / maxLength 滑窗
  母题: input 输入

- id: PuzzleNumPassword      # 密码转盘
  类: 逐位+1 转盘 | 行: 7076-7121
  输入: [每位点击] ; 判定: 拼接==rightPass（可选 OK 键+shake）
  母题: input（CodeLock 标准实现）

- id: PuzzleAniPassword
  类: 动画帧密码 | 行: 6756-6787
  输入: [按钮循环] ; 判定: 各 spriteFrame 序号拼接==rightPass
  母题: input + manipulate（转图形版 CodeLock）

- id: PuzzleCheckBox
  类: 勾选组合 | 行: 6788-6813
  判定: isCheck()→"1/0" 位串==rightPass
  母题: encode（二进制思维，LogicGrid 简化版）

- id: PuzzleColorSelect
  类: 颜色循环 | 行: 6814-6852
  判定: m_selectData 颜色索引拼接==rightPass（颜色表 colors 可配）
  母题: input（颜色 CodeLock）

- id: PuzzleRotation
  类: 旋转对位 | 行: 7191-7228
  判定: m_rotData[i]%dirNum 拼接==rightPass（dirNum=4 缺省，add 步长可负）
  母题: manipulate（转盘图形对齐）

- id: PuzzleSequence         # 顺序记忆（听音/看图按序）
  类: 序列输入 | 行: 7229-7265
  判定: m_inputData 滑窗 join==rightPass（可选 Enter 确认）
  母题: deduce（Simon Says 型）

- id: PuzzleSequenceLight    # 点亮式顺序
  类: 一次性顺序选择 | 行: 7266-7317
  判定: 每钮限选一次（indexOf 去重）+ Light 亮起；错则 ClearInput 全重置
  母题: deduce（比 Sequence 多了"重置"张力）

- id: PuzzleSlider / PuzzleSliderH
  类: 滑块对位（竖/横） | 行: 7332-7441
  判定: positionY/X 吸附最近档位 == poses[rightPass[i]]（rightPass 是档位下标数组）
  母题: manipulate（推杆机关）

- id: PUzzlePinTu            # 拼图
  类: 块交换拼图 | 行: 7122-7190
  机制: CanDrag 拖卡→吸附最近 block（isCollectSprite）→与占位卡交换
  判定: card.userData["Num"]==blockIndex 全对
  母题: jigsaw 拼图（交换制，非滑块制）

- id: PuzzleFindKey          # L9
  类: 放大镜搜索 | 行: 6853-6873
  机制: CanDrag 拖镜体，len<width/2 时 KeyButton 显形，点钥匙过关
  母题: search 探索（隐藏物搜索）

- id: PuzzleStarrySky        # L4
  类: 望远镜观星 | 行: 7442-7454
  机制: ClipLayer 裁剪窗口 + CanDrag 平移星空大图（无判定，纯观察）
  母题: observe（配合场外密码记录）
```

---

## 5. L1 母题统计表

|参考文档母题|dGame 实现|出现频率|备注|
|---|---|---|---|
|CodeLock 数字密码锁|PasswordBar×n + SetInputPasswordBars|**18/18 关**|每关 1-3 组，最高 6 位|
|CipherBreak 密码破译|Puzzle102 文字规则 + 计算推导|12+ 处|本作特色：**汉字笔划(424)/生日+体重(8138)/代数(4515)/不等式排序**|
|ItemMatch 物品匹配合成|SetUnlockNeedBarName|18/18 关|含同名重复、顺序敏感变体|
|MechanismDoor 机关门|SwitchBar/开关+换图|10+ 处|unlockSound 定制开门音|
|Jigsaw 拼图|PUzzlePinTu|1-2 处|交换式|
|LogicGrid 逻辑格子|PuzzleCheckBox/ColorSelect|少量|简化为位串/色串|
|MusicPuzzle 音序|PuzzleSequence/SequenceLight|少量|视觉序（无音高绑定）|
|ShadowPuzzle/Blacklight|紫外灯照试卷/白笔桌布/光照显字|5+ 处|全部用"换图"实现，无真实光影|
|MirrorPuzzle 镜像|PuzzleStarrySky/FindKey|2 处|观察探索型|
|Maze 导航|场景图 _linkChild|每关|导航本身非谜题，图展开即移动|
|NPCDialogue 对话|ShowTalk + talk.json|每关|无分支，纯氛围+提示|
|Timeline 时间线|时钟 12:55→1255|2 处|归入编码破译|
|WeighPuzzle 称重|水量 32+18 毫升|1 处|归入数字推导|

**统计结论**：母题集中度极高——CodeLock + ItemMatch + CipherBreak 三类覆盖 90% 谜题量；物理操作型母题（拼图/滑块/旋转）仅点缀。开发成本主要在**每关的插画与线索纸内容**，而非谜题机制。

---

## 6. L0 原语层：交互原语表

### 6.1 参考文档 12 原语 → dGame 实现映射

|原语|dGame 实现|证据|
|---|---|---|
|search 探索|点击 GameBar 展开邻居 / PuzzleFindKey 镜中寻物|ShowNeighbor 5281|
|observe 观察|PuzzleShowPaper/102/StarrySky / 光照换图|7318|
|combine 合成|拖 A 到 B → OnCollectBar|7536|
|encode 编码|SetDotText 符号 / talk 文本规则|6445|
|decode 解码|GetHintText/Puzzle102 规则→密码|7704|
|associate 关联|PuzzleLink 连线 / 图腾图案序|6874|
|deduce 推理|数学推导谜题链|L2/L5/L7|
|manipulate 操控|CanDrag/CanRotation/Slider/Rotation|5610-5828|
|input 输入|9 种输入 Bar + NumPassword/Calculator|§6.2|
|trigger 触发|SetClickCallback / SetLockCallback|5118|
|navigate 导航|图遍历 + 邻居展开 + 缩放平移|7471|
|communicate 传达|ShowTalk/配音/StageClearEff|5996|

### 6.2 输入 Bar 家族（SetInputPasswordBars 可绑定的 9 种，OnInputPassword instanceof 清单 7631）

|Bar|状态空间|GetText|特色|
|---|---|---|---|
|PasswordBar|0-9 循环|数字|SetDotText 自定义符号+色|
|PasswordBar2|九宫格键盘|滑窗字符串|m_maxLength|
|ButtonBar|序号|m_inputOrder 滑窗|SetButton 绑定外部按钮|
|ColorBar|4 色|索引|红橙黄绿 dot.png 着色|
|ArrowBar|8 向 ↑↗→↘↓↙←↖|索引|SetContents 自定义符号(≤4)|
|StarBar|4 符号 ◆●★▲|索引|SetContents|
|ImageBar|任意图片≤4|索引|SetContents(files)——L13 心/圆/星/方|
|SwitchBar|开/关（初开=1）|0/1|文本"开/关"|
|SwitchBar2|开/关（初关=0）|0/1|SetContant(close,open) 双帧动画，isOpen/isClose——**机关灯谓词**|

### 6.3 通用组件

|组件|能力|备注|
|---|---|---|
|CanDrag|H/V 拖拽、dragLimitLength 行程限制、屏幕边界钳制、长按 500ms、双击、回弹、onDrag 回调|5610-5752|
|CanRotation|触摸角度跟随旋转、m_angleStep 档位吸附、0-360 归一|5753-5828|
|GameBarDotPanel|最多 4 色收集点（AddUnlock(color) 点亮）|5880-5928|

---

## 7. 设计模式笔记

1. **场景即图（Graph-as-Scene）**：`_linkChild` 邻接表 + 递归 InitBar + dLine 画边。导航、谜题位置关系、拖放可达性全部由图决定。复用时只需换邻接表。
2. **门禁 API 显式化**：所有进度门禁收敛到 3 个 Set 方法（SetLock/SetUnlockNeedBarName/SetInputPasswordBars），关卡逻辑可被静态扫描——本拆解的 grep 清单即证明（18 关编排 130 行可完整提取）。
3. **lockName 统一标识 + 实例缓存**：Puzzle 的 m_lockName 既是弹窗缓存键（m_arrPuzzleBuffer）又是门禁路由键（OnUnlock(lockName)），一个字符串串起 UI 状态与进度状态。
4. **状态即换图**：不解锁新场景，只换 Icon/背景（m_unlockIconFileName/SetBackgroudFileName）。零动画成本表达"蜡烛点燃/石头洗净"。
5. **声明式 UI + 命名约定**：`_dscData` JSON 树中 Text/Talk/Icon/Lock/Dot/Light 子节点名被引擎按约定消费（FindChild），策划数据与代码解耦。
6. **汉字主题化编码**：密码不是孤立数字，而是中文知识编码（"六七五"笔划→424）。主题与谜题耦合是本作差异点，成本低（只是线索纸内容）。
7. **呈现顺序与编号解耦（StageIndex）**：内容制作顺序 ≠ 难度顺序 ≠ 呈现顺序，运营期可无损重排章节。
8. **分级提示链**：GetHintText 按当前第一个锁定的门禁顺序生成，`[90XX` 语法把文字提示升级为图片提示，提示消耗货币形成变现点。
9. **反馈语言学**：成功=缩放脉冲+unlock.mp3；失败=CCShake+wrong.mp3；不可开=水平抖动+lock.mp3+锁标摇晃。三档反馈全由动作系统组合，无定制动画。
10. **同一谜题的难度旋钮**：PuzzleLink 预置一条连线；Sequence→SequenceLight 加重置惩罚；Order 门禁加顺序维度；密码长度 3→6 位。同一套代码覆盖新手到硬核。
11. **Puzzle 弹窗位置语义**：弹窗从触发它的 Bar 位置缩放飞入（puzzleStartButton→m_puzzleStartPos），保持空间因果感。
12. **数据表驱动叙事**：talk.json 文本↔配音编号映射，OutputTalk 工具方法可 dump 全关文案（7685）——本地化/改写只需处理一张表。

---

## 8. 再创作指南

用这套引擎/模式做新游戏，需要产出的内容清单：

1. **每关 4 件套**（以 Level1 为模板，约 50 行）：
    - `_dscData()`：底图 + GameBar 节点表 + `_linkChild` 邻接表 + 输入 Bar
    - `Init()`：门禁编排（SetUnlockNeedBarName/SetInputPasswordBars/SetClickCallback）
    - `GetHintText()`：分级提示链
    - 素材：底图 N01.jpg + 物品 Icon NXX.jpg + 线索纸 90XX.png
2. **谜题机制按需挑 17 个现成类**，新母题只需继承 PuzzleBase 实现 Init2 + CheckClear2 + OnPassWordRight 三点。
3. **主题化编码库**（可移植的密码题型）：笔划数/拼音首字母/生肖序/天干地支序/成语缺字——把目标答案数字伪装成中文知识。
4. **难度曲线配方**：教学关（1 密码+3 门禁，如 L12）→ 标准关（双密码+5 门禁+1 弹窗谜题，L1-L9）→ 巨关（8+ 门禁并行汇流，L13/L14）→ 多阶段关（step 状态机，L15/L16/L18）。
5. **本地化**：talk.json 一张表 + GetHintText 字符串 + 关卡名，即可整包换语言；配音按 sound 编号并行替换。

### 质量检查清单（对照参考文档 11.5.4）

- [x]  每个谜题可追溯到母题（§5 表）
- [x]  每个门禁有唯一解锁路径（§3.3 表，代码验证）
- [x]  提示链覆盖全部门禁（GetHintText 逐关比对，[90XX 图提示兜底）
- [x]  失败反馈存在（shake/wrong 三档）
- [ ]  谜题公平性需实机验证（部分算术谜题规则依赖线索纸可读性，如 L8 32+44=26 的规则藏在铭牌图中）