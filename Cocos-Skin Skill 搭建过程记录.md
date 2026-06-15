# Cocos-Skin Skill 搭建过程记录

> 目标：把小游戏换皮流程封装成 AI Skill，让非技术同事一句话完成换皮

---

## 一、需求分析

### 原始痛点

- 换皮依赖 Cocos 引擎本地安装，门槛高
- 每次换皮需要手动一对一配图，效率低
- 换皮后预览需要起本地服务，有一定技术门槛
- 流程无法复用，每次都要重头来

### 目标

- 不安装 Cocos，也能完成换皮
- 最少操作步骤跑通全流程
- Token 消耗极致压缩，方便团队规模化使用

---

## 二、技术调研阶段

### 2.1 分析 Cocos 3.x Build 产物结构

扫描真实工程（`SQ_LotteryGame-skin_1`），发现关键结论：

**图片全部以 UUID 命名**，原始文件名（如 `bg.jpg`）在构建阶段完全丢失：
```
assets/main/native/06/0614ebdb-a6d3-4d48-9fed-0b51d0122726.jpg  ← 原始是 bg.jpg
assets/resources/native/21/21608e45-7b71-4ffc-bd0f-d9a80e032912.png
```

**关键发现**：UUID ↔ 原始文件名的映射存在源工程的 `.meta` 文件里：
```json
// Art/GameForm/bg.jpg.meta
{ "uuid": "0614ebdb-a6d3-4d48-9fed-0b51d0122726", ... }
```

这意味着：只要有源工程，就能建立「原始文件名 → UUID → build 里的实际文件」的完整映射链。

### 2.2 共扫描到 35 张图片，按尺寸分组

| 类型 | 数量 | 尺寸特征 |
|------|------|---------|
| 主背景 | 1 | 720×1680 |
| 奖品格子图 | 10 | 全部 143×143 |
| 各种按钮 | ~10 | 269×93 为主 |
| 弹窗/面板底图 | ~8 | 各异 |
| 特效/小图标 | ~6 | 各异 |

---

## 三、工具开发阶段（HTML 工具探索）

在做 Skill 之前，先尝试用 HTML 工具实现换皮流程，过程中遇到了几个关键问题和解决方案：

### 3.1 目录选择问题

**问题**：最初用 `showDirectoryPicker` API 选目录，需要 HTTPS 或 localhost，`file://` 协议打不开。

**解决**：改用 `input[webkitdirectory]`，双击 HTML 直接用，Chrome/Safari 全支持。

### 3.2 预览方案的多次尝试

| 方案 | 结果 | 原因 |
|------|------|------|
| iframe + Blob URL | ❌ 黑屏 | Cocos 用 ES Module 动态 import，blob 环境下失效 |
| 注入 fetch/XHR 拦截器 | ❌ 还是跑不起来 | import() 不走 fetch/XHR，拦不住 |
| 下载 ZIP + .command 脚本 | ✅ 可行 | macOS 自带 python3，双击起服务 |
| Netlify 上传 + 二维码 | ✅ 可行 | 需要配置 Token，有网络依赖 |

**最终结论**：浏览器内预览 Cocos 3.x 是死路，本地服务器是唯一可靠方案。

### 3.3 匹配策略的演进

| 版本 | 匹配方式 | 问题 |
|------|---------|------|
| v1 | 按文件名直接匹配 | build 里全是 UUID 文件名，匹配不上 |
| v2 | 按目录路径匹配（需皮肤包和源工程结构一致）| 操作成本高，美术不知道怎么组织目录 |
| **v3（最终）** | **按文件名匹配 UUID 映射，皮肤包任意结构** | 美术只需文件名一致，平铺放置即可 |

---

## 四、Skill 设计阶段

### 4.1 为什么选 Skill 而不是 HTML 工具

| 维度 | HTML 工具 | Skill |
|------|-----------|-------|
| 预览 | 无法在浏览器内跑 Cocos 3.x | AI 直接起本地服务，真实运行 |
| 文件操作 | 受浏览器沙箱限制 | 直接操作本地文件系统 |
| 给同事用 | 需要理解多个步骤 | 一句话触发，无需技术背景 |
| Token 优化 | 无 | 缓存机制，后续复用极省 Token |

### 4.2 核心设计原则

**原则一：最少用户输入**
只问一句话收集路径，其余全自动。路径补全（"桌面 XXX" → `/Users/xxx/Desktop/XXX`）由 AI 完成。

**原则二：UUID 映射缓存**
第一次运行扫描 `.meta` 生成 `.skin_uuid_map.json` 缓存文件，存在源工程根目录。后续所有换皮直接读缓存，不重新扫描，Token 消耗降低约 80%。

**原则三：脚本内联执行**
用 `python3 << 'EOF'` 内联运行，不创建临时文件，减少文件系统操作。

**原则四：产出文件统一放桌面**
需求表 Excel 和换皮后的资源包都自动保存到桌面，文件名带时间戳，用户无需关心路径。

---

## 五、Skill 功能迭代过程

### v1：基础换图

- 扫描 `.meta` 建立 UUID 映射
- 按文件名匹配皮肤包，替换 build 目录图片
- 起本地服务器预览

### v2：加入文案替换

**新增需求**：用 Excel 表（A列原文/B列新文案）批量替换游戏内文案。

**实现**：扫描 build 目录所有 `.js` 文件，全文字符串替换，B列写「不替换」则跳过。

**依赖**：`openpyxl` 读取 xlsx，降级支持 csv。

### v3：加入素材需求表生成

**新增需求**：让美术知道需要出哪些图、叫什么名字、多大尺寸。

**实现**：
- 读取 UUID 映射缓存 + build 目录图片
- 解析图片实际尺寸（PNG/JPG 二进制读取）
- 按模块分组（主游戏界面/结算界面/弹窗等）
- 每行插入缩略图作为参考
- 标注哪些不需要替换（`grid_bg`、`num.png` 等）
- 输出到桌面，文件名带时间戳

### v4：交付环节完善

- 换皮完成后自动 `cp -r` 资源包到桌面
- 告知用户完整路径（`~/Desktop/skin-build-YYYYMMDD-HHMM`）
- 预览从桌面副本起服务，保证预览版本与交付版本一致

### v5：合并为单一 Skill，三模式自动识别

把需求表生成、图片换皮、文案替换合并到同一个 Skill，按用户意图自动识别模式，不需要用户区分触发哪个功能。

三种模式共用 UUID 映射缓存，Token 进一步压缩。

### v6：需求表从「依赖 build 过滤」改为「直接读 assets 源图」

**暴露的问题**：在 CatchMaster 工程实战中，生成的需求表漏掉了玩家.png、退出.png 等 7 张核心素材。用户反馈"核心的东西都没有提取出来，连玩家角色图都没有"。

**根本原因**：需求表生成逻辑是「扫描 .meta 拿 UUID → 去 build 里找对应 UUID 文件 → 只把匹配到的列入表格」。但同一工程下有多个 build 目录（web-mobile、web-mobile-001 等），各 build 包含的资源数量差距悬殊（25/33 vs 32/33）。匹配不上的图就被静默过滤掉了，恰好漏掉的是最核心的角色图、加载图。

**修复**：需求表生成完全基于 `assets/` 源目录遍历，**不做任何 build 过滤**，确保 0 遗漏：
- 尺寸直接从 `.meta` 的 `subMetas.f9941.userData.rawWidth/rawHeight` 读取（元数据里本就有）
- 缩略图直接从 assets 源图生成，不依赖 build 里的 UUID 文件
- build UUID 映射只在「换皮替换」步骤里使用，不参与需求表生成

```
❌ 旧逻辑：assets .meta → uuid → 在 build 里找文件 → 找不到就漏
✅ 新逻辑：assets .meta → 直接读源图 → 100% 收录 → 需求表完整
```

### v7：base build 选择从「选第一个」改为「评分选最优」

**暴露的问题**：换皮替换时，7 张图在选中的 base build 里根本不存在，导致无法替换。

**根本原因**：Skill 里自动定位 build 的逻辑是 `for d in web-*: BUILD_DIR = d; break`，直接选第一个 web- 开头的目录。而工程里多个 build 资源完整度差异极大，`web-mobile`（第一个）只有 25/33，`web-mobile-001` 有 32/33，选错之后核心图就没有替换位置。

**修复**：遍历所有 build 子目录，统计每个 build 中匹配 assets UUID 的图片数量，选匹配率最高的作为 base build：

```python
# ✅ 正确：评分选最优
best_score, best_path = -1, None
for d in os.listdir(build_root):
    score = sum(1 for u in assets_uuid_set if u in build_uuids_of(d))
    if score > best_score:
        best_score, best_path = score, d
BUILD_DIR = best_path
print(f"自动选 base build: {basename} ({best_score}/{total} 匹配)")
```

**顺带发现**：同一工程不同 build 之间资源数量不同是正常现象（各皮肤版本 build 使用了不同的资源集），不是 bug，需要选最完整的那个做基底。

### v8：换皮工具独立化，彻底去掉 Cocos 引擎依赖（已有记录，见下）

### v9：修复 base64 内联图全链路缺陷（2026-06-11）

#### 问题背景

用换皮后的包发现 loading 背景（启动画面）没有替换成功。排查发现这张图不在 `assets/native/` 里，而是以 base64 字符串内联在 `src/settings.json` 的 `splashScreen.background.base64` 字段中。

#### 暴露的三个缺陷

**缺陷 1：Step1 Excel 内联图命名错误（MIME 声明 ≠ 实际格式）**

旧逻辑用 MIME 声明字段决定扩展名：
```python
b64_name = f'inline_img_{idx+1}.{ba["mime"].split("/")[-1]}'
# MIME 声明是 "png"，文件名就写 inline_img_1.png
# 但实际图片是 JPEG（文件头 FFD8），PIL 能识别
```
结果：Excel 告诉美术交 `inline_img_1.png`，Step4 查找 `inline_img_1.jpg` 找不到，替换跳过。

**修复**：用 PIL 解码后读取真实格式决定扩展名：
```python
real_ext = 'jpg' if bimg.format == 'JPEG' else 'png'
```

**缺陷 2：Step1 扫描顺序不稳定导致编号错乱**

`os.walk` 遍历文件顺序在不同平台/系统下不固定，Step1 生成 Excel 里的 `inline_img_1` 和 Step4 替换时的 `inline_img_1` 可能对应不同文件。

**修复**：Step1 和 Step4 的 `os.walk` 内层循环统一加 `sorted(files)`，保证两端编号一致。

**缺陷 3：Step4 内联图替换逻辑以「皮肤包有没有文件」为触发条件，失败静默**

旧逻辑：
```python
inline_files = [f for f in os.listdir(SKIN_DIR) if f.startswith('inline_img_')]
if inline_files:   # ← 皮肤包没有文件就整块跳过，无任何提示
    ...
```
结果：如果皮肤包路径填错、文件名大小写不对、或美术漏交文件，脚本静默跳过，用户看不到任何报错。

**修复**：逻辑反转——以「build 里有没有 base64 内联图」为主判断，有就无论如何都尝试替换，找不到皮肤包文件才打印警告：
```python
# 直接扫 build 里的 base64，逐一去皮肤包取对应编号文件
global_idx = 0
for root, _, files in os.walk(OUT_DIR):
    for f in sorted(files):
        ...
        for m in matches:
            global_idx += 1
            name_base = f'inline_img_{global_idx}'
            # 找不到就打 ⚠️ 警告，不静默跳过
            if not inline_src:
                print(f"⚠️  {name_base}: 皮肤包无对应文件，保留原图")
```

#### 顺带修复：Step4 图片替换 for 循环缩进 Bug

发现 Step4 处理多 UUID 资源（同一文件名对应多个 UUID）时，`dst_ext`、`try/except` 误写在 `for e in entries` 循环外，导致只处理了最后一个 entry，且 `ok_img` 计数不含 `shutil.copy2` 路径。

```python
# ❌ 旧逻辑（缩进错误）
for e in entries:
    dst = build_index.get(e['uuid'].lower())
    if not dst: continue
dst_ext = ...   # ← 在循环外，只拿到最后一个 dst
try:
    shutil.copy2(src, dst)
    ok_img += 1  # ← 只在 else 分支计数，copy2 不计
    except: ...  # ← except 层级不对

# ✅ 新逻辑（全部移入循环）
replaced = 0
for e in entries:
    dst = build_index.get(e['uuid'].lower())
    if not dst: continue
    dst_ext = ...
    try:
        shutil.copy2(src, dst)  # or PIL convert
        replaced += 1
    except Exception as ex:
        print(f"✗ {f}: {ex}")
if replaced:
    ok_img += 1
```

#### 根本教训

> **内联资源不走 UUID 路径，必须单独识别和处理。** Cocos 的 `splashScreen.background.base64` 是引擎直接把图片编码进配置文件的，不经过 assets 资源管道，原有的 UUID 映射机制完全覆盖不到它。任何「以为处理了所有资源」的方案，都要显式验证「有没有不走标准路径的资源」。

**用户需求**："我要的是快速换皮，不用强依赖 Cocos 引擎 build"

**原架构问题**：每次换皮后都要重新用 Cocos Editor 执行 Build，门槛高、耗时长，非技术同事根本无法独立完成。

**新架构**：以资源最完整的 build（如 web-mobile-001，32/33 匹配）作为**永久 base template**，不动它；换皮时只做三件事：
1. 复制 base template 到输出目录
2. 按文件名匹配皮肤包 → 找到对应 UUID 文件路径 → 直接覆盖
3. 起本地服务预览

整个流程纯 Python，完全不依赖 Cocos 引擎，非技术同事可独立运行。

最终落地形态：`skin-tool/skin.py`，三条命令搞定一切：
```
python3 skin.py build          # 首次扫描，建立文件名→UUID→build路径映射（缓存永久有效）
python3 skin.py apply <皮肤包>  # 换皮：复制 base + 覆盖 UUID 文件
python3 skin.py preview        # 本地预览
```

---

## 六、关键技术实现（含更新）

### UUID 映射缓存结构

```json
{
  "bg.jpg": [{"uuid": "0614ebdb-...", "orig": "Art/GameForm/bg.jpg"}],
  "bg.png": [
    {"uuid": "de092161-...", "orig": "Art/GuideForm/bg.png"},
    {"uuid": "9f5f5dd2-...", "orig": "Art/PropForm/bg.png"}
  ]
}
```

同一文件名可对应多个 UUID（多模块复用同名图），换皮时全部替换。

### 图片尺寸读取（无需 Pillow）

直接读二进制头部：
- PNG：第 16-24 字节为宽高（大端序）
- JPG：遍历 SOF 段（marker `0xC0/C1/C2`）

### 文案替换范围

只替换 `.js` 文件（Cocos 文案打包在 bundle.js 里），跳过图片、JSON、wasm 等二进制文件，避免误替换。

### 需求表素材来源（v6 新增）

需求表完全基于 `assets/` 源图生成，关键字段读取方式：

| 字段 | 来源 | 说明 |
|------|------|------|
| 文件名 | `.meta` 对应的原始文件名 | 交付命名 |
| 尺寸 | `.meta → subMetas.f9941.userData.rawWidth/rawHeight` | 无需打开图片 |
| 缩略图 | 直接读 `assets/` 源图 | 不依赖 build |
| 换皮路径 | UUID → base build 里的文件路径 | 仅换皮时使用，不影响需求表完整性 |

### base build 评分选择（v7 新增）

同一工程下可能有多个 build 目录（不同皮肤版本），资源完整度差异可能达到 25/33 vs 32/33。评分逻辑：

```python
score = sum(1 for uuid in assets_uuid_set if uuid in build_uuids_of(this_build))
```

选 score 最高的作为 base。选错 base 会导致核心素材没有替换位置，换皮残缺。

---

## 七、踩坑记录（持续更新）

| 问题 | 原因 | 解决 |
|------|------|------|
| import JSON 里读不到原始文件名 | build 阶段 Cocos 已剥离，只保留 UUID | 改从源工程 .meta 读取 |
| `<script src="jszip">` 后直接跟代码导致失效 | script 标签未关闭 | 改为单独闭合的 `<script>` 块 |
| 服务器命令阻塞工具执行 | `python3 -m http.server` 是前台进程 | 改用 `nohup ... &` 后台启动 |
| Excel 缓存格式不兼容新匹配逻辑 | 早期缓存是路径→UUID，新逻辑需要文件名→UUID | 扫描脚本重建缓存格式 |
| 需求表漏掉核心素材（v6 暴露）| 需求表依赖 build UUID 匹配过滤，匹配不上的被静默丢弃 | 改为直接遍历 assets 源目录，不过滤 |
| 换皮时核心图无替换位置（v7 暴露）| 自动选第一个 web-* build，资源只有 25/33 | 评分选最优 build（遍历所有子目录，选匹配率最高的）|
| 不同 build 资源数量差异大被误以为是 bug | 多个 build 是不同皮肤版本，资源集本就不同 | 理解工程结构后，直接选最完整的做 base |
| loading 背景未替换（v9 暴露）| splashScreen.background.base64 是内联图，不走 UUID 路径，原逻辑完全覆盖不到 | Step1 专项扫描 build 内所有 JSON/JS/HTML 的 base64 字段，列入 Excel；Step4 直接解码替换 |
| Excel 内联图文件名扩展名错误（v9 暴露）| 用 MIME 声明（如 "png"）定扩展名，但实际图片是 JPEG | 用 PIL 解码读真实格式（`bimg.format`），JPEG → `.jpg` |
| 内联图替换静默跳过无提示（v9 暴露）| 以「皮肤包有没有 inline_img_ 文件」为触发条件，文件缺失直接跳过 | 改为以「build 有没有 base64」为主判断，缺文件打 ⚠️ 警告 |
| os.walk 顺序不稳定导致内联图编号错乱（v9 暴露）| Step1 和 Step4 用 os.walk 顺序不固定，同一内联图两端编号可能不一致 | Step1 和 Step4 统一改为 `for f in sorted(files)` |
| Step4 多 UUID 资源只处理最后一个（v9 暴露）| `for e in entries` 循环后，`dst_ext` 和 `try/except` 误写在循环外 | 全部移入循环体，新增 `replaced` 计数变量，计数逻辑统一 |

---

## 八、最终 Skill 结构（v8 更新）

```
cocos-skin/
└── SKILL.md          # 技能全部逻辑，约 700 行
skin-tool/            # 独立换皮工具（不依赖引擎）
├── skin.py           # 三条命令封装：build / apply / preview
└── uuid_map.json     # 文件名→UUID→build路径映射缓存（首次生成后永久复用）
```

**触发词覆盖**：
- 需求表：「生成需求表」「出素材规格」「素材需求」「出素材清单」
- 换皮：「换皮」「替换素材」「皮肤包」「换图」「游戏换皮」「小游戏换皮」
- 换文案：「换文案」「替换文案」「文案替换」

**完整使用流程（v8）**：
```
上新游戏（一次性）
  → 说「生成需求表」+ 提供源工程路径
  → AI 产出 Excel（带参考图，0 遗漏）→ 发给美术

首次初始化换皮工具（一次性）
  → python3 skin.py build
  → 扫描 assets，建立文件名→UUID→build路径映射缓存

美术交图后
  → python3 skin.py apply <皮肤包目录>
  → 自动复制最完整的 base build + 覆盖对应 UUID 文件
  → python3 skin.py preview  确认效果后交付

有文案需求时
  → 追加文案 Excel 路径即可，Skill 内合并一次完成
```

---

## 九、经验总结（持续更新）

1. **先搞清楚技术边界**：Cocos build 产物的 UUID 机制是整个方案的核心，不理解这一点就无法设计正确的匹配逻辑。

2. **预览方案要务实**：花了较多时间尝试浏览器内预览，最终发现是死路。遇到技术限制要果断换方向，不要死磕。

3. **操作成本决定能否落地**：最初要求皮肤包和源工程目录结构一致，被同事指出操作成本高，立即改为按文件名匹配，降低了美术的认知门槛。

4. **缓存是省 Token 的核心**：同一源工程的 UUID 映射不会变，一次扫描永久复用，这是后续换皮 Token 消耗极低的关键。

5. **单 Skill 多模式优于多 Skill**：需求表生成和换皮共用上下文和缓存，合并后用户无需记忆多个触发词，也不需要重复提供路径信息。

6. **产出文件统一放桌面**：减少用户找文件的认知负担，带时间戳避免覆盖历史版本。

7. **数据来源要追到最上游（v6 教训）**：需求表应该从 `assets/` 源目录生成，而不是从 build 产物反推。用 build 做数据源会把「build 是否包含该资源」的不确定性引入需求表，导致核心素材被意外过滤。**原则：展示类数据用最原始的源头，替换类操作才需要 build 路径。**

8. **环境假设要实测验证（v7 教训）**：「选第一个 web-* 目录就是对的」是未经验证的假设。真实工程往往有多个 build 版本共存，自动化选择逻辑必须有评分机制而非写死规则。**原则：凡是「自动选择」的逻辑，都要有验证维度，不能靠命名规律猜测。**

9. **彻底去除引擎依赖是正确方向（v8 教训）**：早期方案仍然依赖 Cocos Editor 做最终 build，这是一个隐形门槛。把「base template + 纯文件覆盖」的思路引入后，整个链路才真正做到零依赖，任何人可独立运行。**原则：能在文件层面解决的事情，不要引入额外运行时依赖。**

10. **「以为处理了所有资源」是最危险的假设（v9 教训）**：Cocos 的 loading 背景图不走 assets 资源管道，直接以 base64 字符串内联在 `settings.json` 配置文件里。原有的 UUID 机制完全覆盖不到，需要专项识别。**原则：任何「全量处理」方案，都要显式列举「不走标准路径的资源类型」，逐一验证，不能靠推断。**

11. **静默失败是 Skill 最难定位的 bug（v9 教训）**：内联图替换原本以「皮肤包有没有对应文件」为触发条件，文件缺失就整块跳过，用户看不到任何提示。排查时只能靠文件时间戳比对才发现 settings.json 没被改动过。**原则：Skill 里所有「条件跳过」的路径，都必须有明确的日志输出，失败要比成功更显眼。**

12. **MIME 声明不可信，要用 PIL 读真实格式（v9 教训）**：settings.json 里 MIME 声明是 `image/png`，实际文件头是 JPEG（`FFD8`）。用 MIME 决定扩展名导致 Excel 里命名为 `.png`，但 Step4 查找 `.jpg` 找不到，替换跳过。**原则：凡是涉及文件格式判断，一律用 PIL/二进制文件头实测，不信任 MIME 字段。**

---

## 十、方法论沉淀：如何搭建一个高质量 Skill

通过 cocos-skin 的完整迭代过程，总结出以下可复用的 Skill 搭建方法论：

### 10.1 技术调研先行，找到「最小可行映射」

搭建 Skill 前，先把核心数据流走通：输入是什么 → 中间状态是什么 → 输出是什么。
cocos-skin 的关键发现是「文件名 → .meta UUID → build 里的文件」这条映射链。找到这条链之前，任何自动化设计都是空谈。

### 10.2 数据来源用最上游，不要依赖中间产物

| 场景 | 反例 | 正例 |
|------|------|------|
| 生成需求表 | 从 build 产物反查 | 直接读 assets 源目录 |
| 读取图片尺寸 | 打开 build 里的 UUID 文件 | 读 .meta 的 rawWidth/rawHeight |
| 生成缩略图 | 用 build 里的图 | 直接用 assets 源图 |

**原则**：展示类操作用最原始的数据源，执行类操作（替换、覆盖）才需要目标路径。

### 10.3 自动选择逻辑要有评分维度

凡是 Skill 里有「自动定位」「自动选择」的地方，都不能靠命名规律猜测，要有可度量的评分维度：
- build 选择：匹配的 UUID 数量
- 模块分类：文件路径关键词命中数
- 文案过滤：技术关键词排除规则

**原则**：自动化的判断必须可解释，出错时能说清楚「为什么选了这个」。

### 10.4 去除所有隐形依赖

每个依赖都是一个落地障碍：
- Cocos Editor → 用文件覆盖替代
- 网络请求 → 本地文件操作替代
- 特定目录结构 → 文件名匹配替代

**原则**：列出所有外部依赖，逐一问「能不能去掉？」。能去掉的一定去掉。

### 10.5 用真实工程做压力测试

Skill 在示例工程上跑通不等于真实可用。每次用新工程测试，都会暴露新假设：
- 示例工程只有一个 build → 真实工程有七八个
- 示例工程文件名都是英文 → 真实工程全是中文
- 示例工程资源整齐 → 真实工程 build 之间资源数量差异达 25% 以上

**原则**：Skill 要在「最脏的真实场景」下测试，而不是在「最干净的示例」上测试。

### 10.6 失败路径比成功路径更重要

一个 Skill 能成功运行不难，难的是失败时能被发现。设计每一个分支时都要问：
- 这条路径失败了，用户能看到提示吗？
- 如果静默跳过，用户如何知道有东西没处理？
- 出错信息能帮用户定位根因，还是只说「跳过」？

v9 的内联图问题之所以难排查，根本原因就是原始逻辑的失败路径是完全静默的。文件时间戳比对是这次找到根因的唯一手段，这本不应该是排查方式。

**原则：每个「跳过」都要说清楚跳过了什么、为什么跳过。成功日志可以简洁，失败日志必须完整。**

### 10.7 两端对称原则：Step1 和 Step4 的处理逻辑必须镜像一致

Step1（生成需求表）和 Step4（执行替换）是一个闭环：Step1 告诉美术交什么文件名，Step4 按这个文件名去找。任何不一致都会导致替换失败：

| 维度 | v9 修复前 | v9 修复后 |
|------|-----------|-----------|
| 内联图扩展名 | Step1 用 MIME 声明，Step4 查 `.jpg` | 两端都用 PIL 真实格式 |
| 内联图编号顺序 | Step1 `os.walk` 随机，Step4 也随机，两端编号可能不一致 | 两端统一 `sorted(files)`，编号稳定 |

**原则：生成需求表和执行替换的文件查找逻辑必须用同一套规则，任何一端改动都要同步另一端。**
