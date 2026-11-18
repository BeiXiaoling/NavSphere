# NavSphere 部署到 Vercel

> 适用于本 fork（`BeiXiaoling/NavSphere`）。数据存储在 GitHub 仓库，**不需要服务器、不需要数据库**。

## 0. 前置条件

- [ ] GitHub 账号已 fork 本仓库
- [ ] Vercel 账号（用同一 GitHub 账号登录）
- [ ] 本地有 pnpm（仅本地预览需要）

## 1. 创建 GitHub OAuth App

GitHub → **Settings → Developer settings → OAuth Apps → New OAuth App**

| 字段 | 值 |
|------|-----|
| Application name | `NavSphere` |
| Homepage URL | `https://<最终域名>`（可先填占位，稍后改） |
| Authorization callback URL | `https://<最终域名>/api/auth/callback/github` |

创建后记录：

- **Client ID**
- **Client Secret**（点 Generate a new client secret，只显示一次）

> 回调地址必须精确到 `/api/auth/callback/github`，末尾不要带斜杠。

## 2. 创建 Fine-grained PAT

GitHub → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**

| 项 | 值 |
|----|-----|
| Token name | `NavSphere Data` |
| Expiration | 1 年（到期前记得换） |
| Repository access | **Only select repositories** → `BeiXiaoling/NavSphere` |
| Permissions | `Contents: Read and write`，其余全部 `No access` |

生成后复制 token（`github_pat_...`，只显示一次）。

## 3. 生成 AUTH_SECRET

```bash
openssl rand -base64 32
```

Windows PowerShell：

```powershell
[Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))
```

## 4. 在 Vercel 导入项目

1. 打开 <https://vercel.com/new>
2. 选择 `BeiXiaoling/NavSphere` → **Import**
3. Framework Preset 自动识别为 **Next.js**，Root Directory 保持 `./`
4. 构建配置**不要手动改**，`vercel.json` 已定义：
   - install：`pnpm install --no-frozen-lockfile`
   - build：`pnpm build`
   - output：`.next`

### 环境变量

在 **Environment Variables** 中逐个添加（Production 与 Preview 都要勾选）：

| 变量 | 示例值 / 说明 |
|------|--------------|
| `GITHUB_OWNER` | `BeiXiaoling` |
| `GITHUB_REPO` | `NavSphere` |
| `GITHUB_BRANCH` | `main` |
| `GITHUB_CLIENT_ID` | 步骤 1 的 Client ID |
| `GITHUB_CLIENT_SECRET` | 步骤 1 的 Client Secret |
| `GITHUB_PAT` | 步骤 2 的 PAT |
| `AUTH_SECRET` | 步骤 3 生成的随机串 |
| `NEXTAUTH_URL` | `https://<项目名>.vercel.app`，**先填这个，绝不能留 localhost** |
| `NEXT_PUBLIC_API_URL` | 同 `NEXTAUTH_URL` |
| `GA_ID` | 可选，Google Analytics 衡量 ID |

填完点 **Deploy**，首次构建约 1–3 分钟。

## 5. 部署后回填（必做）

拿到真实域名（形如 `navsphere-xxxx.vercel.app`）后，**三处都要改**：

1. **Vercel → Settings → Environment Variables**
   把 `NEXTAUTH_URL` 与 `NEXT_PUBLIC_API_URL` 改为真实域名
2. **GitHub OAuth App**
   更新 Homepage URL 与 Authorization callback URL
3. **Vercel → Deployments → 最新 Deployment → ⋯ → Redeploy**

> 环境变量修改后不会自动生效，**必须 Redeploy**。

## 6. 绑定自定义域名（可选）

1. Vercel → Settings → Domains → 添加域名，按提示配置 DNS
2. 重复步骤 5 的三处修改（域名换成自定义域名）

## 7. 本地预览（可选）

```bash
pnpm install
cp .env.example .env.local   # 填入真实值，NEXTAUTH_URL=http://localhost:3000
pnpm dev                     # http://localhost:3000
```

本地调试时，OAuth App 的回调地址要临时加上 `http://localhost:3000/api/auth/callback/github`（GitHub 允许填多个回调地址）。

---

## 常见问题

| 现象 | 原因 | 解决 |
|------|------|------|
| 点登录白屏 / 回调 404 | `NEXTAUTH_URL` 还是 localhost 或与 OAuth 回调不一致 | 三处域名统一后 Redeploy |
| `redirect_uri_mismatch` | OAuth App 回调地址与访问域名不符 | 检查协议、域名、路径，末尾无斜杠 |
| 保存导航报 401 / 403 | PAT 权限不足或仓库选错 | 重新建 PAT：`Contents: R/W`，仅授权本仓库 |
| 保存成功但刷新没变 | 数据写入仓库有延迟或页面缓存 | 等几秒强刷；确认 `GITHUB_BRANCH` 为 `main` |
| 每次改数据都触发 Vercel 构建 | 数据提交推送到仓库触发自动部署 | Settings → Git → Ignored Build Step 按提交信息前缀过滤 |
| 构建失败 `ERR_PNPM_OUTDATED_LOCKFILE` | 锁文件与依赖不同步 | 已在 `vercel.json` 用 `--no-frozen-lockfile` 规避，勿改 |

## 安全检查清单

- [ ] PAT 只授权单个仓库，权限仅 `Contents: Read and write`
- [ ] `GITHUB_CLIENT_SECRET`、`AUTH_SECRET`、`GITHUB_PAT` 未出现在任何提交中
- [ ] `.env.local` 已在 `.gitignore` 内
- [ ] PAT 过期时间已记录在日历提醒
