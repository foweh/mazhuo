# 反诈小易 · 校园反诈 AIGC 智能体（演示版）

基于《反诈小易AIGC智能体策划方案 V3.0》与《马茁软件设计特点》设计的**单文件 Vue3 聊天页面**，可直接运行、可部署到 GitHub Pages。

> 项目成员：马茁、张馨颖、郭宇航

---

## 文件结构

```
反诈小易-web/
├── fanzha-xiaoyi/       # ★ 部署目录（已推送至 github.com/foweh/mazhuo 仓库）
│   ├── index.html      # 部署壳：浏览器内直接编译加载 ChatPage.vue（GitHub Pages 零构建运行）
│   ├── ChatPage.vue    # ★ 核心交付物：反诈小易聊天页（Vue3 单文件组件）
│   └── README.md       # 本说明
├── 方案全文.txt         # 客户策划方案 docx 提取的参考文本（仅本地，未推送）
└── .gitignore
```

## 设计要点（对应马茁软件设计特点）

- 明亮浅蓝主色 `#4A8FE7` + 暖黄 `#FFC53D` 点缀，白/浅灰中性色
- 卡片式布局、大圆角（14–22px）、线性图标、卡通小熊 IP（内联 SVG，无外部图片依赖）
- 移动端优先：`100dvh` 全屏、安全区适配、底部输入栏、触控反馈
- 聊天交互：流式输出（SSE）、打字指示、快捷玩法卡片、Markdown 轻渲染（加粗/代码/换行）

## 功能对照（方案 V2.0 增强版）

| 方案功能 | 本演示版实现 |
| --- | --- |
| 🕵️ 海龟汤 / 🎭 剧本杀 / 👀 火眼金睛 / 🏆 PK晋级赛 | 快捷卡片一键进入，由大模型按系统提示词现场出题/主持 |
| 📚 19 种诈骗类型知识库 | 注入系统提示词，支持"查知识"问答 |
| ✍️ 提交案例 / 🚨 紧急求助 | 求助弹窗：四步指引 + 复制辅导员求助模板；案例提交走对话 |
| 积分 · 段位（王者 8 段） | 会话内积分 + 段位徽章（内存态，刷新清零，符合"隐私不存储"） |
| 内容审核 / 合规边界 | 系统提示词安全护栏 + 客户端敏感词预筛拦截 |
| 教师端 / 联机 / AIGC 图片题 | 需后端与易班平台对接，本演示版不包含 |

## 密钥说明（重要）

DeepSeek API Key 以**混淆形式**存储于 `ChatPage.vue` 顶部：

- 混淆方式：字符串反转 → 三段拆分 → 逐字节异或 → HEX 编码，运行时在内存中重组，用完即弃
- ⚠️ **诚实声明**：静态网页无法真正加密。此方案只能提高"随手复制"的提取门槛，无法抵御故意逆向（DevTools 断点即可还原）
- 该 key 余额约 2 元，建议：
  1. 演示结束后在 [platform.deepseek.com](https://platform.deepseek.com) **重置/删除**该 key
  2. 正式上线请改用 **Cloudflare Workers 免费代理**转发请求（key 只存服务端，浏览器拿不到），可参考下方代码
  3. 页面上的 `Authorization` 请求头拦截器/代理工具都能看到明文 key，属于同类风险

### 更换/重新混淆 key 的方法

打开 `ChatPage.vue`，修改顶部配置区三个片段，可用下面的命令重新生成（Windows PowerShell / Python）：

```python
# python 一键生成混淆片段
key = "sk-你的新密钥"
rev = key[::-1]
salts = [0x5A, 0xC7, 0x2F]
parts = [rev[:9], rev[9:19], rev[19:]]
for i, p in enumerate(parts):
    print(f"片段{i+1} HEX:", bytes(b ^ salts[i] for b in p.encode()).hex().upper())
# 把输出的 3 段 HEX 依次填入 ChatPage.vue 的 _cfg1 / _tips[0] / _tips[1]
```

### 进阶：Cloudflare Workers 代理（真隐藏 key，推荐正式版）

```js
// worker.js —— 部署为 Cloudflare Worker 后，前端改为请求你的 worker 域名
export default {
  async fetch(request) {
    const KEY = 'sk-你的key'; // 只存在 Worker 环境变量里
    const url = 'https://api.deepseek.com/chat/completions';
    const body = await request.text();
    return fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + KEY },
      body
    });
  }
};
```

## 本地预览

```bash
# 在 反诈小易-web 目录下（任选其一）
python -m http.server 8080
# 然后浏览器打开 http://localhost:8080
```

> 直接双击 index.html 也能打开（`file://` 下 fetch .vue 在部分浏览器受限，推荐用本地服务器方式）。

## 部署到 GitHub Pages

本项目已部署到现有仓库 `github.com/foweh/mazhuo` 的 `fanzha-xiaoyi/` 子目录（不覆盖仓库根目录已有网站）：

- **演示地址**：`https://foweh.github.io/mazhuo/fanzha-xiaoyi/`
- 更新方式：修改 `fanzha-xiaoyi/` 内文件后提交推送即可，Pages 自动重新构建

如果以后要独立成站（新建仓库）：

```bash
git init
git add .
git commit -m "反诈小易演示版"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<新仓库名>.git
git push -u origin main
# 仓库 Settings → Pages → Deploy from branch / main / 根目录
```

## 免责声明

- 本页面为比赛/演示用途，AI 生成内容仅供参考，不构成任何法律或安全承诺
- 紧急情况请直接拨打 **96110**（全国反诈专线）或 **110**
- 页面不收集、不存储任何用户隐私数据（会话与积分为内存态，刷新即清）
