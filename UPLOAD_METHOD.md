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

---

## 实际成功案例（本仓库）

### 环境
- 本地网络：SSH 端口 22/443 被封，HTTPS 大包连接重置
- 工具链：`gh api` + `GH_TOKEN`（通过 `require_escalated` 授权通道运行）
- 最大单次请求：1MB（再大连接会断）

### 步骤

#### 1. 切 1MB 块
```bash
split -b 1M 5上.pdf 5上.pdf.part.
# → 5上.pdf.part.aa, 5上.pdf.part.ab, ...
```

#### 2. 5 路并行创建 blob
```python
# Python ThreadPoolExecutor + subprocess 调用 gh api
with concurrent.futures.ThreadPoolExecutor(max_workers=5) as ex:
    fut = {ex.submit(create_blob, f): f for f in chunks}

def create_blob(fpath):
    with open(fpath, 'rb') as f:
        b64 = base64.b64encode(f.read()).decode('ascii')
    payload = json.dumps({"content": b64, "encoding": "base64"})
    r = subprocess.run(
        ['gh', 'api', f'/repos/{owner}/{repo}/git/blobs', '--input', '-'],
        input=payload.encode(), capture_output=True, timeout=120
    )
```

#### 3. 一次合成 tree → commit → ref
```python
# 创建 tree（包含所有 blob）
POST /repos/{owner}/{repo}/git/trees
{"tree": [{"path": "5上.pdf.part.aa", "mode": "100644", "type": "blob", "sha": "..."}, ...]}

# 创建 commit
POST /repos/{owner}/{repo}/git/commits
{"message": "add all chunks", "tree": "<tree_sha>", "parents": ["<parent_sha>"]}

# 更新分支
PATCH /repos/{owner}/{repo}/git/refs/heads/main
{"sha": "<commit_sha>", "force": true}
```

### 为什么不直接用 git push
| 尝试 | 结果 |
|------|------|
| `git push` SSH (22) | 端口被封 |
| `git push` SSH (443) | 端口被封 |
| `git push` HTTPS | 140MB 大包传输中断 |
| 5MB 块 + HTTPS | 连接重置 |
| 2MB 块 + HTTPS | 连接重置 |
| **1MB 块 + Blobs API** | ✅ 稳定上传 127 个块 |

### 关键教训
1. **1MB 是临界点** — 这个网络环境下 1MB 请求基本不会断
2. **Blobs API 优于 Contents API** — 创建 blob 不需要 sha，可以任意并行
3. **gh api 优于 urllib** — 通过授权通道跑 `gh` 比 Python 直连更稳定
4. **并行度 5** — 再多反而容易触发网络限制
