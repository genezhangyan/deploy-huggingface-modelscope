---
name: deploy-huggingface-modelscope
description: 部署项目到 HuggingFace Spaces 和/或 ModelScope Studios。当用户说"部署"、"deploy"、"push 到平台"、"发布"、"上线"、"推送到 hf"、"推送到 ms/魔塔"时立即使用此 skill。适用于任何同时维护 HuggingFace 和 ModelScope 两个平台的项目，也支持单平台部署。
---

# 部署到 HuggingFace / ModelScope

## 第一步：读取项目配置

运行以下命令了解当前项目的 git remotes：

```bash
git remote -v
```

找出：
- **HuggingFace remote 名**（通常是 `hf`，指向 `huggingface.co/spaces/...`）
- **ModelScope remote 名**（通常是 `ms`，指向 `modelscope.cn/studios/...`）
- **当前分支**（`git branch --show-current`）

如果用户没有指定平台，默认推送到两个平台。

---

## 第二步：推送代码

### 推送到 HuggingFace

```bash
git push <hf-remote> <branch>
```

HuggingFace 推送后会自动重建 Space，**无需额外操作**。

### 推送到 ModelScope

```bash
git push <ms-remote> <branch>
```

> **关键陷阱**：ModelScope 的 `git push` 仅更新文件，**不会自动重启**容器。每次推送后必须手动调用 restart API，否则改动不生效。

---

## 第三步：触发 ModelScope 重启

需要以下信息（从用户的项目记忆或直接询问用户获取）：
- **AccessToken**：ModelScope 个人设置中的 API Token
- **用户名/空间名**：从 remote URL 中提取，格式 `modelscope.cn/studios/<用户名>/<空间名>.git`

```bash
MS_TOKEN="<用户的 AccessToken>"
MS_USER="<用户名>"
MS_SPACE="<空间名>"

# Step 1: 登录获取 session cookies（Bearer token 无效，必须用此方式）
curl -s -X POST "https://www.modelscope.cn/api/v1/login" \
  -H "Content-Type: application/json" \
  -c /tmp/ms_cookies.txt \
  -d "{\"AccessToken\": \"$MS_TOKEN\"}"

# Step 2: 提取 CSRF token
CSRF=$(grep csrf_token /tmp/ms_cookies.txt | awk '{print $NF}')

# Step 3: 调用 restart
curl -s -X PUT "https://www.modelscope.cn/api/v1/studio/${MS_USER}/${MS_SPACE}/restart" \
  -b /tmp/ms_cookies.txt \
  -H "X-CSRF-Token: $CSRF" \
  -H "Content-Type: application/json"
```

响应 `{"Code": 200}` 表示重启成功。

---

## 第四步：告知用户结果

部署完成后，告诉用户：
- 两个平台的访问 URL（从 remote URL 推断，或查项目文档）
- HuggingFace 通常 1-3 分钟生效
- ModelScope 通常 2-5 分钟生效（如需 lazy install 依赖则更长）

---

## 已知陷阱（所有 ModelScope 项目通用）

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| push 后页面无变化 | push 不触发重启 | 必须调用 restart API |
| restart API 返回 "user not logged in" | Bearer token 方式无效 | 用 cookie 认证（见上方步骤） |
| 容器启动失败 | requirements.txt 含预装包 | 不要写 torch、gradio 等平台预装的包 |
| push 认证失败 | token 过期 | 重新配置 git credentials |

---

## 获取 ModelScope Token 的方法

如果用户不记得 token：
1. 登录 https://www.modelscope.cn
2. 右上角头像 → 个人中心 → Access Token
3. 复制 token（格式：`ms-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`）

建议用户将 token 保存到项目的 CLAUDE.md 或 memory 文件中，避免每次都要重新找。
