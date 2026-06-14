# Rime 输入法重装与学习记录

> 日期：2026-06-14
> 环境：Windows 11，小狼毫（Weasel）0.17.4

---

## 1. 操作流程：删除并重新部署 Rime

### 1.1 发现的文件结构

| 路径 | 用途 |
|------|------|
| `C:\Program Files\Rime\weasel-0.17.4\` | 小狼毫程序本体 |
| `C:\Users\User\AppData\Roaming\Rime\` | 用户配置、输入方案、词库 |
| `C:\Users\User\AppData\Roaming\plum\` | 东风破（plum）包管理工具 |

### 1.2 删除旧配置

1. 终止 WeaselServer.exe 进程
   ```
   taskkill /PID <PID> /F
   ```
2. 删除用户配置目录
   ```
   rm -rf %APPDATA%\Rime
   ```

### 1.3 下载并安装输入方案

由于 plum 通过 git clone 下载极慢（GitHub 网络问题），改用手动下载 zip 方式：

**下载的方案包（从 GitHub）：**

| 包名 | 仓库地址 | 说明 |
|------|---------|------|
| rime-prelude | rime/rime-prelude | 基础配置（default.yaml, key_bindings 等） |
| rime-essay | rime/rime-essay | 八股文词频数据 |
| rime-luna-pinyin | rime/rime-luna-pinyin | 朙月拼音 |
| rime-cangjie | rime/rime-cangjie | 仓颉5 |
| rime-stroke | rime/rime-stroke | 笔画输入 |
| rime-jyutping | rime/rime-jyutping | 粤拼 |
| rime-easy-en | BlindingDark/rime-easy-en | Easy English |
| rime-japanese | gkovacs/rime-japanese | 日语 |
| rime-vietnamese | gkovacs/rime-vietnamese | 越南语 |
| rime-spanish | gkovacs/rime-spanish | 西班牙语（也支持法语、德语） |
| rime-hangul | einverne/rime-hangul | 韩语 |

**下载命令：**
```bash
curl -fsSL "https://github.com/rime/rime-prelude/archive/master.zip" -o rime-prelude.zip
curl -fsSL "https://github.com/rime/rime-essay/archive/master.zip" -o rime-essay.zip
# ... 其他包同理
```

### 1.4 解压安装

用 Python 脚本批量解压，将 `.yaml`、`.txt`、`.lua`、`opencc/` 文件复制到 `%APPDATA%\Rime\`：

```python
import zipfile, os, shutil, glob

workdir = r'C:\Users\User\WorkBuddy\2026-06-14-17-47-36'
rime_dir = r'C:\Users\User\AppData\Roaming\Rime'

os.makedirs(rime_dir, exist_ok=True)

zip_files = glob.glob(os.path.join(workdir, '*.zip'))
for zf in sorted(zip_files):
    with zipfile.ZipFile(zf, 'r') as z:
        names = z.namelist()
        if not names:
            continue
        top_dir = names[0].split('/')[0]
        z.extractall(workdir)
        extracted = os.path.join(workdir, top_dir)
        for root, dirs, files in os.walk(extracted):
            for f in files:
                src = os.path.join(root, f)
                rel = os.path.relpath(src, extracted)
                if rel.startswith('opencc') or f.endswith(('.yaml', '.txt', '.lua')):
                    dst = os.path.join(rime_dir, rel)
                    os.makedirs(os.path.dirname(dst), exist_ok=True)
                    shutil.copy2(src, dst)
        shutil.rmtree(extracted)
```

### 1.5 注册输入方案

创建 `default.custom.yaml`：

```yaml
patch:
  schema_list:
    - schema: luna_pinyin_simp
    - schema: luna_pinyin
    - schema: luna_quanpin
    - schema: luna_pinyin_fluency
    - schema: luna_pinyin_tw
    - schema: jyutping
    - schema: cangjie5
    - schema: cangjie5_express
    - schema: stroke
    - schema: easy_en
    - schema: japanese
    - schema: vietnamese
    - schema: vietnamese_hannom
    - schema: spanish
    - schema: hangeul
    - schema: hangeul2set
```

### 1.6 部署

```bash
# 先启动 WeaselServer
"/c/Program Files/Rime/weasel-0.17.4/WeaselServer.exe" &

# 然后部署
"/c/Program Files/Rime/weasel-0.17.4/WeaselDeployer.exe" --deploy &
```

部署耗时约 2-3 分钟（日语 mozc 词库较大），最终在 `build/` 目录生成 57 个编译文件。

---

## 2. Rime 架构理解

### 2.1 四层架构

```
Layer 1: Frontend（前端）
    → Weasel (Windows) / Squirrel (macOS) / Fcitx5 (Linux)
    → 负责接收按键、显示候选词

Layer 2: librime（引擎）
    → 按键处理 / 编码翻译 / 候选词过滤
    → 核心 C++ 库

Layer 3: Schema + Dictionary（方案 + 词库）
    → .schema.yaml 定义输入规则和引擎流水线
    → .dict.yaml 定义词典

Layer 4: Build + User data（编译产物 + 用户数据）
    → .prism.bin / .table.bin / .reverse.bin
    → .userdb/ 用户词频记忆
```

### 2.2 按键处理流水线

```
用户按键 → Processor → Segmentor → Translator → Filter → 候选词列表
```

| 组件 | 职责 | 例子 |
|------|------|------|
| Processor | 处理原始按键事件 | 判断快捷键、切换中英文 |
| Segmentor | 把输入码分段 | `nihao` → `ni` + `hao` |
| Translator | 把编码翻译成候选词 | `ni` → 「你/尼/泥」 |
| Filter | 过滤/排序候选词 | 去重、简繁转换、词频排序 |

这些组件在 `.schema.yaml` 的 `engine:` 部分定义，可以自由组合。

### 2.3 文件格式

| 文件 | 作用 | 是否可改 |
|------|------|---------|
| `default.yaml` | 全局默认配置 | ❌ 改 `.custom.yaml` |
| `default.custom.yaml` | 全局自定义补丁 | ✅ |
| `xxx.schema.yaml` | 输入方案定义 | ❌ 改 `.custom.yaml` |
| `xxx.custom.yaml` | 方案自定义补丁 | ✅ |
| `xxx.dict.yaml` | 词典源文件 | ✅ 加词改词 |
| `essay.txt` | 八股文（词频数据） | 一般不改 |
| `weasel.yaml` | 小狼毫界面样式 | ❌ 改 `.custom.yaml` |

> **核心原则**：永远改 `.custom.yaml`，不改原始 `.yaml`。部署时 Rime 会自动把 custom 合并进去，更新方案时定制不会丢失。

---

## 3. 迁移指南

### 迁移到新电脑

1. **在新电脑上安装小狼毫**（程序本体必须安装）
2. **关闭小狼毫**（右键托盘图标 → 退出）
3. **复制整个 `%APPDATA%\Rime\` 到新电脑的 `%APPDATA%\Rime\`**
4. **启动小狼毫 → 右键 → 重新部署**

可选：同时复制 `%APPDATA%\plum\`（东风破工具，方便以后管理方案）。

### 不用 plum 也能增删方案

**添加方案：**
1. 去 GitHub 下载方案的 `.yaml`、`.txt`、`.lua` 文件
2. 复制到 `%APPDATA%\Rime\`
3. 在 `default.custom.yaml` 的 `schema_list` 里加上方案名
4. 重新部署

**删除方案：**
1. 从 `default.custom.yaml` 的 `schema_list` 里去掉方案名
2. 可选：删掉该方案的源文件
3. 重新部署

### plum 的好处

| 功能 | 有 plum | 无 plum |
|------|---------|---------|
| 安装方案 | `rime-install rime/rime-jyutping` | 手动下载复制 |
| 更新方案 | `rime-install --update` | 手动检查更新 |
| 批量安装 | `rime-install preset extra` | 一个个下 |
| 依赖处理 | 自动拉取 | 容易漏 |

---

## 4. 研究路线

| 级别 | 内容 | 耗时 |
|------|------|------|
| Level 1 | 读 `default.custom.yaml` 和 `luna_pinyin.schema.yaml`，理解结构 | 5 分钟 |
| Level 2 | 读官方文档：定制指南、RimeWithSchemata | 1 小时 |
| Level 3 | 读源码：librime（C++）、weasel（C++） | 深入 |
| Level 4 | 实战：调方案顺序、改皮肤、加自定义短语、写最简方案 | 持续 |

### 官方资源

- 官网：https://rime.im
- 定制指南：https://github.com/rime/home/wiki/CustomizationGuide
- Schema 详解：https://github.com/rime/home/wiki/RimeWithSchemata
- librime 源码：https://github.com/rime/librime
- weasel 源码：https://github.com/rime/weasel

---

## 5. 本次安装的方案清单

| 方案 | schema_id | 说明 |
|------|-----------|------|
| 朙月拼音·简化字 | luna_pinyin_simp | 默认方案，简体中文 |
| 朙月拼音 | luna_pinyin | 繁体中文 |
| 朙月全拼 | luna_quanpin | 全拼模式 |
| 朙月拼音·语句流 | luna_pinyin_fluency | 流畅模式 |
| 朙月拼音·台湾正体 | luna_pinyin_tw | 台湾繁体 |
| 粤拼 | jyutping | 粤语拼音 |
| 仓颉5 | cangjie5 | 仓颉输入 |
| 仓颉5·快序 | cangjie5_express | 仓颉快速 |
| 笔画 | stroke | 五笔画 |
| Easy English | easy_en | 英文输入 |
| 日语 | japanese | 日语输入 |
| 越南语 | vietnamese | 越南语 |
| 越南语·汉字 | vietnamese_hannom | 越汉混输 |
| 西班牙语 | spanish | 西/法/德语 |
| 韩语 | hangeul | 韩语 |
| 韩语2集 | hangeul2set | 韩语扩展 |

> 马来语：目前没有找到 Rime 的马来语输入方案。
