# 一键部署到 HuggingFace + 魔搭（ModelScope）

> 一个 Claude Code Skill，帮你同时把 AI 项目发布到国内外两大主流模型托管平台，再也不用手动踩一遍老坑。

---

## 这是什么？

vibe codding一个小web app，免费公网跑的话，hugging face和国内的魔塔是两个不错的选择

- **HuggingFace Spaces** — 国际主流，国内访问慢但圈子大
- **魔搭 ModelScope Studios** — 国内主流，速度快、用户多

每次改完代码，要推送两个 remote、还要手动触发魔搭重启……如果哪步漏了，就会出现"我明明 push 了，为什么没更新"的灵魂拷问。

这个 Skill 装好之后，你只需要跟 Claude 说一句**"部署"**，它会自动帮你完成全套流程。

---

## 我踩过哪些坑（血泪史）

### 坑一：魔搭 push 了没用

这是最坑的地方，而且文档完全没说清楚。

`git push` 到魔搭成功了，文件确实更新了，但页面就是没变化。原因是：**魔搭的容器不会因为 push 而自动重启**。如果容器之前失败过或者已经停掉了，你 push 多少次都没用，老容器还在跑。

你必须在 push 之后，额外调用一次 restart API，才能让新代码生效。

### 坑二：restart API 的认证方式是反直觉的

想着用 AccessToken 直接鉴权，结果：

```
{"message": "user not logged in"}
```

状态码还是 200，没报错，就是认证没过。试了半天发现：**魔搭的 restart API 不接受 Bearer Token**，必须先用 Token 登录拿到 session cookie，再用 cookie + CSRF token 去调接口。整个流程是这样的：

```bash
# 第一步：用 AccessToken 登录，换取 session cookie
curl -X POST "https://www.modelscope.cn/api/v1/login" \
  -H "Content-Type: application/json" \
  -c /tmp/ms_cookies.txt \
  -d '{"AccessToken": "你的token"}'

# 第二步：从 cookie 文件里提取 csrf_token
CSRF=$(grep csrf_token /tmp/ms_cookies.txt | awk '{print $NF}')

# 第三步：带着 cookie 和 CSRF 调 restart
curl -X PUT "https://www.modelscope.cn/api/v1/studio/你的用户名/你的空间名/restart" \
  -b /tmp/ms_cookies.txt \
  -H "X-CSRF-Token: $CSRF" \
  -H "Content-Type: application/json"
```

### 坑三：requirements.txt 写多了会炸

魔搭环境预装了 `torch 2.3.1` 和 `gradio 6.2.0`。如果你在 `requirements.txt` 里写了这些包（哪怕版本对的），pip 会尝试重新安装，然后报错，整个容器启动失败。

错误信息很长很吓人，但根本原因就是：**不要在 requirements.txt 里写平台已经装好的包**。

安全的写法（以典型 AI 应用为例）：

```
Pillow>=10.0.0
numpy>=1.26.0
huggingface_hub>=0.27.0
```

不要写：`torch`、`gradio`、`modelscope-studio`。

---

## 装了之后有什么变化

**装之前**，每次部署你要：
1. `git push hf master`
2. `git push ms master`
3. 打开魔搭网页，手动点重启（或者写一堆 curl 命令）
4. 等半天，发现没生效，开始 debug

**装之后**，你说一句"部署"，Claude 帮你：
1. 读取项目的 git remote，自动识别 hf 和 ms 对应哪个
2. 推送到 HuggingFace（它会自动重建，不用管）
3. 推送到魔搭
4. 自动执行 cookie 认证 → 调 restart API
5. 告诉你两个平台的 URL 和大概需要等多久

---

## 怎么安装

把 `SKILL.md` 复制到 Claude Code 的 skills 目录：

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/deploy-huggingface-modelscope
cp SKILL.md ~/.claude/skills/deploy-huggingface-modelscope/SKILL.md

# Windows（Git Bash）
mkdir -p /c/Users/你的用户名/.claude/skills/deploy-huggingface-modelscope
cp SKILL.md /c/Users/你的用户名/.claude/skills/deploy-huggingface-modelscope/SKILL.md
```

重启 Claude Code 会话后自动生效。

---

## 项目怎么配置

### 1. 设置 git remote

```bash
# HuggingFace
git remote add hf https://huggingface.co/spaces/你的用户名/你的空间名

# 魔搭
git remote add ms https://www.modelscope.cn/studios/你的用户名/你的空间名.git
```

### 2. 把魔搭的 Token 告诉 Claude

在项目的 `CLAUDE.md` 或 `.claude/memory/MEMORY.md` 里加上这几行，Claude 下次部署就不用再问你了：

```markdown
## ModelScope
- AccessToken: ms-你的token
- 用户名: 你的用户名
- 空间名: 你的空间名
```

Token 在哪里找：登录 [modelscope.cn](https://www.modelscope.cn) → 右上角头像 → 个人中心 → Access Token

---

## 支持的指令

说什么都行，Claude 能理解意图：

| 你说 | Claude 做什么 |
|------|-------------|
| 部署 / deploy | 推送到两个平台 + 触发魔搭重启 |
| 发布 / 上线 | 同上 |
| 只推 HuggingFace | 只推 hf，不动魔搭 |
| 只推魔搭 | 推 ms + 触发重启 |

---

## 两个平台的特性对比

| | HuggingFace Spaces | 魔搭 ModelScope |
|---|---|---|
| push 后自动重启 | ✅ 会 | ❌ 不会，必须手动触发 |
| 国内访问速度 | 慢 | 快 |
| 认证方式 | 标准 git | Cookie + CSRF（见上方说明） |
| 预装包 | 看 runtime 配置 | torch 2.3.1, gradio 6.2.0 |
| 服务启动时间 | 1–3 分钟 | 2–5 分钟（首次运行更久）|
| 端口配置 | 自动 | 需手动指定 `server_port=7860` |

---

如果你也在同时维护 HuggingFace 和魔搭，希望这个 Skill 能帮你省点时间。有问题欢迎开 Issue。
