<div align="center">

<img src="docs/readme/cover.svg" alt="Neural-Janitor project overview" width="100%" />

# Neural-Janitor · Chrome / Edge 标签页管理

**从你的真实关闭行为中学习，先预览，再让清理真正执行。**

[![macOS](https://img.shields.io/badge/macOS-13%2B-5C84C6)](companion/NeuralJanitorCompanion/Package.swift) [![Browser](https://img.shields.io/badge/Browser-Chrome%20%2F%20Edge-5375B5)](extension/manifest.json) [![License](https://img.shields.io/badge/License-MIT-3A9F82)](LICENSE)

[English](README.md) · **简体中文**

[安装](#安装指南) · [第一次使用](#第一次使用) · [决策机制](#清理决策怎样形成) · [代码导航](#代码导航)

</div>

如果你平时在 Mac 上经常开着六七十个甚至上百个标签页，大概率也试过（然后卸掉过）各种“闲置 X 小时自动关标签页”的浏览器扩展。固定倒计时最大的问题是不分青红皂白：它常常把你还没审完的 PR、正在查的 API 文档悄悄关掉，却留着十五六个早就看完的 Google 搜索结果页占内存。

Neural-Janitor 由 Chrome / Edge 扩展与本地 Swift 伴随程序（`The Chronos Engine`）组成。它不靠死板的定时器，而是观察**你平时到底在什么时候按 `Cmd+W` 手动关标签页**（`manual_browser_close`），再结合根域名历史、页面类别、前台停留时长和交互次数，学出属于你自己的保留习惯。Core ML 空闲窗口预测与 Apple `NaturalLanguage` 本地文本分类全部在设备端运行，作为辅助信号配合这套策略。

运行需要 **macOS 13+、Chrome / Edge 与 Swift 工具链**。新安装永远默认处于 **Test**（仅预览）模式——在系统学够你的真实习惯、且你主动切到 **Deploy** 之前，不会在后台偷关任何一个页面。

## 为什么它用起来不招人烦

| 核心机制 | 代码里具体是怎么做的 |
| --- | --- |
| **先预览，攒够信任再动手** | 新安装默认 `Test`（只给过期候选打标，绝不自动关闭）。`Test → Armed → Deploy` 的状态跃迁由 [deployment-readiness.js](extension/js/deployment-readiness.js) 根据真实学习进度把关。 |
| **算“切到后台多久”，不算“打开多久”** | 清理比较的是 `now - lastBackgroundedAt` 以及前台停留、交互次数——哪怕一个标签页是三天前打开的，只要十分钟前刚切过去看过，就绝不会当成过期页。 |
| **绝不用自动清理去自我强化** | 在 [closure-learner.js](extension/js/closure-learner.js) 中，手动关闭权重为 `1.0`（自动过滤 `<15s` 的误触），而自动清理样本权重被压到 `0.2` 仅作上下文；一旦你在恢复记录里点恢复，**对应的自动关闭样本会被立刻删除**，当场纠正偏差。 |
| **搜索结果页单独隔离建桶** | 搜索页被单独分流进 `Search Results` / `search:<engine>` 学习桶（[search-results.js](extension/js/search-results.js)）。你养成 30 秒关掉 Google 搜索页的习惯，不会连累同根域名下的 Google Docs 或 Gmail 被提前清理。 |
| **Chrome 与 Edge 共享同一份本地记忆** | 两个浏览器通过 Native Messaging stdio 连接同一个本地 Swift 伴随程序（位于 `~/Library/Application Support/Neural-Janitor/`），共用同一套学习档案和 Core ML 模型。 |
| **真实空闲门控 + 节假日感知** | 定时清理必须等待浏览器进入真实的 `idle` / `locked` 状态，并且会向伴随程序发送未来 7 天的节假日等级，避免把周一法定假日当成普通工作日来预测作息。 |

## 清理决策怎样形成

![预览、就绪度与恢复](docs/readme/flow-cn.svg)

| 模式 | 看到什么 | 可以发生什么 |
| --- | --- | --- |
| `Test` | 过期候选标签与实时学习统计 | 只打标预览，AI Clean 与定时清理全部锁定；新安装默认状态。 |
| `Armed` | 你请求了 `Deploy`，且已积累至少 **3 次手动关闭 + 1 个已学习桶** | 进入待命状态，继续按预览模式运行，攒够就绪门槛后自动接管。 |
| `Deploy` | 已达到 **5 次手动关闭 + 2 个已学习桶**（满 **10 次 + 3 个桶**标记为 `safer`） | 满足空闲条件的定时清理与手动 AI Clean 可以关闭标签页，并写入一键恢复记录。 |

[deployment-readiness.js](extension/js/deployment-readiness.js) 同时负责就绪升级与降级保护：如果重置了模型或样本不足以支撑当前模式，会自动从 `Deploy` 退回 `Armed` 或 `Test`。

手动关闭时间学习始终是主决策源。就算 Core ML 模型预测当前空闲概率很高，只要你还在操作浏览器，定时任务就**不会**动手关页。弹窗顶部的 **AI Clean** 则是你主动点击的即时减压通道，想立刻腾出内存时可以小批量清掉明显低价值的后台页。

## 安装指南

Native Messaging 需要在 macOS 上安装一个 native host manifest。Chrome / Edge 不能静默安装这个 host，所以仍然需要一次性的安装步骤，除非把伴随程序打包成签名安装器。

### 1. 克隆或打开仓库

```bash
git clone https://github.com/dubianche123/neural-janitor-tab-manager.git
cd neural-janitor-tab-manager
```

### 2. 加载扩展

打开 `chrome://extensions` 或 `edge://extensions`，启用 Developer Mode，选择 **Load unpacked**，然后选中：

```text
extension/
```

复制浏览器显示的 Extension ID。

### 3. 构建并链接伴随程序

```bash
chmod +x scripts/install.sh
./scripts/install.sh YOUR_CHROME_EXTENSION_ID [YOUR_EDGE_EXTENSION_ID]
```

之后重新加载扩展。扩展发起 Native Messaging 连接时，伴随程序会自动启动。
如果 Chrome 和 Edge 的扩展 ID 不一样，就把两个 ID 都传给脚本，这样 native host manifest 会同时允许两边。
安装脚本会把伴随程序放在 `~/Library/Application Support/Neural-Janitor/NeuralJanitorCompanion`。
Native Messaging 连接是短连接：浏览器每次请求启动 host，拿到一次响应后主动断开，避免拖住浏览器退出。

### 4. 伴随程序变化后

如果 Swift 伴随程序或 native host 元数据变化了，再运行一次：

```bash
./scripts/install.sh YOUR_CHROME_EXTENSION_ID [YOUR_EDGE_EXTENSION_ID]
```

## 第一次使用

1. 打开弹窗，确认处于 **Test**，并检查伴随程序连接状态。
2. 正常浏览，手动关闭已经看完的页面。这些真实决定才是有用的训练信号。
3. 使用 **Preview** 检查过期候选项，并查看域名/分类学习与就绪状态。
4. 希望启用清理时请求 Deploy。如果仅达到可待命门槛，会进入 Armed，继续等待学习。
5. 发生一次允许的关闭后，打开关闭记录，尝试恢复其中一项。

例如，经常快速关闭搜索结果，应当主要教会独立搜索桶；不应因此直接缩短同一大型网站下邮件、工作页面的寿命。有域名建议时，域名历史优先于宽泛分类。

恢复时优先尝试浏览器会话恢复；没有可用会话时，回退为重新打开记录的 URL。它不是未保存页面状态的备份。恢复与样本删除逻辑见 [background.js](extension/js/background.js)。

## 架构

### 1. 标签页交互追踪器

追踪器记录标签页什么时候进入前台、什么时候离开前台、在前台停留了多久，以及交互了多少次。清理时比较的是 `now - lastBackgroundedAt`，而不是单纯看“打开多久了”。

### 2. 手动关闭学习器

手动关闭是主信号。学习器会保存分类、根域名、前台停留、后台年龄和交互次数，再从有意义的手动样本中推荐关闭时间。根域名模式会先适应；类别级模式需要更多、跨多个域名的证据。自动清理样本只作为上下文记录，不会拿来反过来训练系统自己。

### 3. 本地页面分类器

浏览器先用域名映射和页面信号进行分类。置信度不高时，Swift 伴随程序会用 Apple `NaturalLanguage` 对标题、描述和正文打分。浏览器侧还会保留一个很小的根域名分类记忆，让重复站点不要一直掉回 `Other`。

### 4. 辅助闲置预测器

伴随程序会从本地活动历史训练一个 9 特征的 `TrainingSample` 模型。这个模型是辅助角色：它影响闲置情境倍率和 ML 控制台，而真正的主学习器仍然是关闭时间学习。自动 stale 关闭等待 Chrome 真实报告 `idle` / `locked`；单独一个很高的模型先验不会在你仍然 active 时关闭标签页。

### 5. 节假日感知闲置窗口

浏览器会把未来七个自然日的节假日等级发送给伴随程序。这样即使今天不是假日，周一如果是日本或中国节假日，周一的预测也会单独变化。工作日和周末/假期的睡眠窗口只是先验，不是硬性清理规则。

### 6. 内存压力清理

AI Cleanup 会按照学到的关闭压力、参与度、交互次数和类别做弱权重排序。低价值、低交互、长时间闲置的标签页优先被清理。Test 模式只做预览，不会真正关闭。Deploy 生效后，手动触发 AI Clean 时，即使当前还没到压力阈值，也可以主动清掉一小批明显不重要的后台标签页。

## 分类关闭时间规则

标签页的关闭时间由四部分决定：类别默认值、手动关闭学习、根域名历史和单个标签页的重要度。只要某个根域名已经学到自己的关闭时间，它会优先于宽泛的大类经验，避免一个网站把整个类别带偏。搜索结果页会被隔离进 `Search Results` 和 `search:<engine>` 学习桶。设置里的滑块只是上限，不是替换；模型可以更早关闭，但不会超过你设定的最大值。

| 分类 | 最大闲置时间 | 说明 |
|----------|--------------|------|
| **NSFW** | **12 小时** | 打开一次就走，不等闲置窗口。 |
| Search Results | 12 小时 | 默认就是偏一次性的页面，之后会按你的 SERP 关闭习惯适应。 |
| Social Media | 3 天 | 价值衰减很快。 |
| Entertainment | 5 天 | 常回看，但通常不是工作核心。 |
| News | 5 天 | 新鲜度很重要。 |
| Shopping | 7 天 | 有用，但不该无限期保留。 |
| Other | 7 天 | 未分类页面的保守默认值。 |
| Reference | 10 天 | 文档和文章往往还能继续参考。 |
| Work & Productivity | 14 天 | PR、工单和草稿需要时间。 |
| Email & Communication | 14 天 | 会话连续性有时很重要。 |
| **Finance & Banking** | **30 天** | 高价值会话，但不是永久的。 |
| **AI Tools** | **30 天** | 长时间的研究和对话窗口通常是有意保留的。 |

## 在多台 Mac 之间迁移本地模型

Neural-Janitor 的学习成果保存在：

```text
~/Library/Application Support/Neural-Janitor/
```

不要在伴随程序运行时用 iCloud 直接同步这个目录。请改用导出 / 导入脚本。

### 在源 Mac 上导出

```bash
./scripts/export_model_bundle.sh --output ~/Desktop
```

如果你还想连原始活动历史一起迁移：

```bash
./scripts/export_model_bundle.sh --with-events --output ~/Desktop
```

### 在目标 Mac 上导入

先正常安装 Neural-Janitor，再执行：

```bash
./scripts/import_model_bundle.sh ~/Desktop/neural-janitor-model-bundle-YYYYMMDD-HHMMSS.tar.gz
```

如果你故意想恢复 `activity_events.json`：

```bash
./scripts/import_model_bundle.sh --with-events ~/Desktop/neural-janitor-model-bundle-YYYYMMDD-HHMMSS.tar.gz
```

导入脚本会校验 checksum，并把旧文件备份到：

```text
~/Library/Application Support/Neural-Janitor/backups/
```

导入后重新加载浏览器扩展，让伴随程序重新载入模型。

## 弹窗怎么用

- **Preview**：立刻给 stale 标签页打标，不会关闭任何页面。
- **AI Clean**：顶部按钮用于把标签页数量或内存压力压回目标；AI Suggestions 里的 `Clean safest` / `Clean more` 是单独限量的小批量清理。
- **MEM / CPU**：显示当前内存压力、CPU 占用，以及简短的 CPU 型号 / 线程数。
- **ML Insights**：显示未来七天的闲置窗口，包含工作日、周末或节假日标签。
- **Settings**：控制是否使用伴随程序、日历选择、关闭时间上限、白名单、定时黑名单和 AI Cleanup 目标。
- **AI Suggestions**：把标签页数量/内存压力、stale 预览、低重要度清理拆成独立卡片；Deploy/Test 就绪状态会单独显示。
- **Reset Model State**：清空关闭学习、域名记忆、idle 预测和本地 companion 学习文件。

如果你更习惯命令行，直接运行：

```bash
scripts/reset_model_state.sh
```

## 代码导航

| 职责 | 源码 |
| --- | --- |
| 浏览器事件、定时清理与恢复 | [background.js](extension/js/background.js) |
| 手动关闭建议与样本处理 | [closure-learner.js](extension/js/closure-learner.js) |
| Test / Armed / Deploy 状态转换 | [deployment-readiness.js](extension/js/deployment-readiness.js) |
| 清理排序 | [cleanup-ranking.js](extension/js/cleanup-ranking.js) |
| 页面分类与独立搜索桶 | [categorizer.js](extension/js/categorizer.js)、[search-results.js](extension/js/search-results.js) |
| 弹窗控制与说明 | [popup.js](extension/js/popup.js)、[popup.html](extension/popup.html) |
| 原生宿主、训练、预测与本地存储 | [Swift 伴随程序](companion/NeuralJanitorCompanion/Sources/main.swift) |
| 宿主注册与模型迁移 | [scripts/](scripts/) |

浏览器负责标签页权限与动作，伴随程序通过 **Native Messaging stdio 上带长度前缀的 JSON** 提供本地学习与预测。安装脚本注册允许连接宿主的扩展 ID。[详细 C4 源码](docs/readme/runtime-detail-cn.mmd) 保留了组件视角。

## 安全与隐私

- **没有云端分析**：活动日志、模型和标签页注册表都保留在本机和扩展本地存储中。
- **没有远程追踪脚本**：扩展不会注入远程代码或分析像素。
- **纯本地模型**：Core ML 训练和推理都在设备上完成。
- **Native Messaging 边界**：浏览器 JS 和 Swift 通过长度前缀的本地 JSON over stdio 通信。

## 开发检查

```bash
node --check extension/js/background.js
node --check extension/js/content.js
node --check extension/js/constants.js
node --check extension/js/categorizer.js
node --check extension/js/holidays.js
node --check extension/js/idle-detector.js
node --check extension/js/popup.js
node --check extension/js/storage.js
python3 -B -m py_compile scripts/train_model.py
bash -n scripts/install.sh
bash -n scripts/uninstall.sh
bash -n scripts/export_model_bundle.sh
bash -n scripts/import_model_bundle.sh
swift build -c release --package-path companion/NeuralJanitorCompanion
```

## 许可证

采用 [MIT License](LICENSE) 开源。由于 macOS 上的 Native Messaging 需要本地注册宿主清单，请在浏览器开发者模式下加载 `extension/` 并执行一次 `./scripts/install.sh` 完成伴随程序构建与注册。

<p align="center"><sub>Neural-Janitor：边缘加速的标签页卫生管理 —— Chronos 引擎</sub></p>
