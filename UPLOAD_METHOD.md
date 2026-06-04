# 大文件上传到 GitHub 的方法

## 问题
本地网络对 SSH（22/443）和大 HTTPS 请求有限制，`git push` 和 Contents API 在大文件传输时会连接重置。

## 解决方案：Git Database API + 并行分块上传

### 原理
1. 把大文件切成 1MB 小块
2. 用 Git Blobs API 5 路并行上传每个小块（每块独立，无冲突）
3. 最后用 git tree + commit + ref 一次合成为完整提交

### 步骤

#### 1. 切块 (1MB)
```bash
split -b 1M bigfile.pdf bigfile.pdf.part.
```

#### 2. 并行创建 blob (5 路)
```http
POST /repos/{owner}/{repo}/git/blobs
Content-Type: application/json

{"content": "<base64 编码的文件块>", "encoding": "base64"}
```
5 条线程同时上传，每个 blob 独立创建，无 sha 冲突。

#### 3. 一次合成
```http
POST /repos/{owner}/{repo}/git/trees
# 把所有 blob sha 按路径组装成一棵树

POST /repos/{owner}/{repo}/git/commits
# 用这棵树创建一个提交

PATCH /repos/{owner}/{repo}/git/refs/heads/main
# 更新分支指针
```

### 为什么这方法可靠

| 方法 | 问题 |
|------|------|
| `git push` SSH | 端口 22/443 被封 |
| `git push` HTTPS | 大包被中间代理重置 |
| Contents API 单文件 | 大于 1MB 的请求会断 |
| **Git Blobs API + 1MB 分块** | ✅ 1MB 请求极少断，断了重试成本低 |
| **并行上传** | ✅ 5 路并发，速度提升 5 倍 |

### 合并

上传完成后，用 GitHub Actions 在工作流里 `cat` 拼回原文件：

```yaml
- run: |
    cat bigfile.pdf.part.* > bigfile.pdf
    rm -f bigfile.pdf.part.*
    git add bigfile.pdf
    git commit -m "combine chunks"
    git push
```
