# 核心概念

## Git 钩子（Hook）是什么

Git 钩子是一种在 Git 事件发生时自动触发的自定义脚本机制。通俗理解：

- 钩子 = "当 X 发生时，自动执行 Y"
- 存储在仓库的 `.git/hooks/` 目录下
- 支持 Shell、Python、Node.js 等任何可执行脚本语言

> 核心价值：将重复的手动检查自动化，在代码提交、推送、合并等关键节点强制执行团队规范。

## 钩子的生命周期

```
开发者操作 → Git 事件 → 触发对应钩子脚本 → 根据脚本退出码决定是否继续
                ↓
           钩子返回 0 → 操作继续
           钩子返回 非 0 → 操作被拒绝
```

## 钩子分类

| 类型 | 作用位置 | 典型用途 |
| :--- | :------- | :------- |
| 客户端钩子 | 开发者本地，`.git/hooks/` 下 | 代码检查、格式校验、提交信息规范 |
| 服务端钩子 | 远程仓库服务器 | 权限校验、CI 触发、部署 |

# 客户端钩子

## pre-commit

在用户执行 `git commit` 时、编辑器弹出提交信息之前触发。如果脚本返回非 0，提交被拒绝。

| 方面 | 说明 |
| :--- | :--- |
| 触发命令 | `git commit` |
| 参数 | 无 |
| 返回值 0 | 允许提交 |
| 返回值 非 0 | 拒绝提交 |
| 跳过方式 | `git commit --no-verify`（`-n`） |

**示例：自动运行代码检查**

```bash
#!/bin/sh
# .git/hooks/pre-commit

# 运行 ESLint（Node.js 项目）
npm run lint
if [ $? -ne 0 ]; then
    echo "错误：代码检查未通过，提交被拒绝"
    exit 1
fi

# 检查是否有调试代码
if git diff --cached | grep -E "(console\.log|debugger|TODO)" > /dev/null; then
    echo "警告：暂存区中包含调试语句或 TODO 标记"
    echo "请在提交前检查或删除"
    exit 1
fi

exit 0
```

> **注意**
>
> 1. `--no-verify` 会**同时跳过** pre-commit 和 commit-msg 钩子
> 2. 如果钩子脚本执行时间很长（如完整测试套件），会明显拖慢提交速度，建议只放轻量检查
> 3. 如果脚本解释器有误或脚本本身报错，Git 会认为钩子执行失败，提交被拒绝

## commit-msg

在用户输入提交信息后、提交真正生成之前触发，用于校验提交信息的格式。

| 方面 | 说明 |
| :--- | :--- |
| 触发命令 | `git commit` |
| 参数 | 临时文件路径，文件内容为提交信息 |
| 返回值 0 | 接受提交信息 |
| 返回值 非 0 | 拒绝提交 |

**示例：校验 Conventional Commits 格式**

```bash
#!/bin/sh
# .git/hooks/commit-msg

commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,}$"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "错误：提交信息不符合 Conventional Commits 规范"
    echo ""
    echo "正确格式示例："
    echo "  feat: 添加用户登录功能"
    echo "  fix(cart): 修复购物车总数计算错误"
    echo "  docs: 更新 API 文档"
    echo ""
    echo "类型：feat / fix / docs / style / refactor / test / chore / perf / ci / build / revert"
    exit 1
fi

exit 0
```

> **注意**
>
> 1. commit-msg 比 pre-commit 更适合校验提交信息格式 —— 此时提交者已经写完了提交信息
> 2. 如果想同时校验代码和提交信息，需要同时配置 pre-commit 和 commit-msg 两个钩子
> 3. skip 方式与 pre-commit 相同：`git commit --no-verify`

## post-commit

在 `git commit` 成功完成后触发。返回值不影响提交结果 —— 提交已经完成，仅做通知用途。

| 方面 | 说明 |
| :--- | :--- |
| 触发命令 | `git commit` |
| 参数 | 无 |
| 返回值 | 不影响提交 |

**示例：提交后发送通知**

```bash
#!/bin/sh
# .git/hooks/post-commit

commit_hash=$(git rev-parse HEAD)
commit_msg=$(git log -1 --pretty=%B)

# 发送企业微信/Slack 通知（示例）
curl -X POST -H "Content-type: application/json" \
    --data "{\"text\":\"新提交: $commit_hash - $commit_msg\"}" \
    "https://hooks.slack.com/services/YOUR/WEBHOOK/URL" 2>/dev/null

exit 0
```

> **注意**
>
> post-commit 适合做通知、打点等副作用操作。不要在这里做重要验证 —— 即使脚本失败了，提交也已经写入版本库，无法回退。

## pre-push

在 `git push` 执行时、远程引用更新之前触发。

| 方面 | 说明 |
| :--- | :--- |
| 触发命令 | `git push` |
| 参数 | 远程仓库名、远程 URL |
| 标准输入 | 一行一条待推送引用：`<本地引用> <本地SHA> <远程引用> <远程SHA>` |
| 跳过方式 | `git push --no-verify` |

**示例：推送前执行完整测试**

```bash
#!/bin/sh
# .git/hooks/pre-push

# 运行完整测试套件
npm test
if [ $? -ne 0 ]; then
    echo "错误：测试未通过，推送被拒绝"
    exit 1
fi

# 扫描是否包含敏感信息
git diff HEAD | grep -iE "(password|secret|api.?key|token)\s*=" > /dev/null
if [ $? -eq 0 ]; then
    echo "警告：推送内容可能包含敏感信息（password/secret/token）"
    echo "请确认后重新推送"
    exit 1
fi

exit 0
```

> **注意**
>
> 1. pre-push 适合放耗时较长的检查（如完整测试套件），因为推送频率远低于提交
> 2. 标准输入中的四列信息可用于识别正在推送哪些引用，实现精细化控制

## post-merge

在 `git merge` 或 `git pull`（内部执行 merge）成功后触发。常用于自动化依赖安装。

| 方面 | 说明 |
| :--- | :--- |
| 触发命令 | `git merge` / `git pull` |
| 参数 | `1` 表示 squash 合并，`0` 表示非 squash 合并 |
| 返回值 | 不影响合并结果 |

**示例：合并后自动重装依赖**

```bash
#!/bin/sh
# .git/hooks/post-merge

# 检查 package.json 是否在这次合并中发生了变化
if git diff HEAD@{1} HEAD --name-only | grep -q "package.json"; then
    echo "检测到 package.json 变更，正在更新依赖..."
    npm install
fi

exit 0
```

## 其他客户端钩子简介

| 钩子 | 触发时机 | 典型用途 |
| :--- | :------- | :------- |
| prepare-commit-msg | 提交信息生成后、编辑器打开前 | 自动填入模板、追加分支名到提交信息 |
| pre-rebase | `git rebase` 开始时 | 阻止对已推送分支执行 rebase |
| post-checkout | `git checkout` / `git switch` 成功后 | 切换分支后自动重载环境变量、安装依赖 |
| post-rewrite | `git commit --amend` 或 `git rebase` 后 | 同步更新其他引用 |

# 服务端钩子

服务端钩子部署在远程仓库服务器上，在接收推送数据时触发。

```
开发者推送 → 远程仓库接收
              ├→ pre-receive（推送前，可拒绝）
              ├→ update（每个分支分别触发，可拒绝）
              └→ post-receive（推送后，仅通知/触发部署）
```

## pre-receive

在远程仓库接收推送后、更新引用之前触发。

| 方面 | 说明 |
| :--- | :--- |
| 返回值 非 0 | 拒绝所有推送的引用更新 |
| 适用场景 | 全局性策略检查：必须签 CLA、禁止 force push 到 main、检查提交者身份 |

## update

与 pre-receive 类似，但为**每个即将更新的引用**分别触发一次，可针对特定分支做精细控制。

| 方面 | 说明 |
| :--- | :--- |
| 参数 | `<引用名> <旧SHA> <新SHA>` |
| 适用场景 | 禁止向 main 分支直接推送、禁止删除特定保护分支 |

**示例：禁止强制推送到 main 分支**

```bash
#!/bin/sh
# .git/hooks/update

ref="$1"
old="$2"
new="$3"

# 禁止删除 main 分支
if [ "$ref" = "refs/heads/main" ] && [ "$new" = "0000000000000000000000000000000000000000" ]; then
    echo "错误：禁止删除 main 分支"
    exit 1
fi
```

## post-receive

在远程仓库引用更新完成后触发，推送已经成功。

| 方面 | 说明 |
| :--- | :--- |
| 适用场景 | 自动部署、触发 CI/CD、发送通知、更新 issue 系统 |

> **注意**
>
> 1. 服务端钩子的可用性取决于远程仓库平台 —— GitHub 通过 Webhook（非传统 Git 钩子）实现类似功能；GitLab 和自建 Git 服务器可以直接部署钩子脚本
> 2. 服务端钩子是团队规范的最终执行者，一旦拒绝推送，开发者必须修正后才能重新推送
> 3. `pre-receive` 和 `update` 的区别：前者全局触发一次，后者每个分支触发一次

# 钩子管理实战

## 启用钩子

Git 仓库初始化后，`.git/hooks/` 目录下已包含示例钩子文件（以 `.sample` 结尾）：

```bash
ls .git/hooks/
# 输出示例：
# applypatch-msg.sample      pre-applypatch.sample
# commit-msg.sample          pre-commit.sample
# post-update.sample         pre-push.sample
# prepare-commit-msg.sample  pre-rebase.sample
```

启用钩子只需两步：

```bash
# 1. 创建可执行脚本（移除 .sample 后缀）
mv .git/hooks/pre-commit.sample .git/hooks/pre-commit

# 2. 确保脚本有可执行权限（Linux/macOS）
chmod +x .git/hooks/pre-commit

# Windows（Git Bash 中）只要脚本有 shebang（#!/bin/sh）即可执行
```

## 团队共享钩子方案

`.git/hooks/` 目录不会被提交到版本库，因此需要额外策略让团队成员共享钩子配置。

### 方案一：core.hooksPath（最轻量，推荐）

在项目根目录创建 `githooks/` 文件夹存放钩子脚本，提交到版本库：

```bash
# 1. 在仓库中创建 hooks 目录
mkdir githooks

# 2. 将钩子脚本放入 githooks/ 目录
cp .git/hooks/pre-commit githooks/pre-commit

# 3. 提交到版本库
git add githooks/
git commit -m "chore: 添加共享 Git 钩子"

# 4. 每位团队成员克隆后配置
git config core.hooksPath githooks/
```

### 方案二：Husky（Node.js 项目首选）

```bash
# 安装 husky
npm install husky --save-dev

# 初始化（创建 .husky/ 目录）
npx husky init

# 添加钩子
npx husky add .husky/pre-commit "npm run lint"
npx husky add .husky/commit-msg "npx commitlint --edit $1"
```

Husky 的优势：
- 自动处理跨平台兼容性
- `.husky/` 目录可提交到版本库，团队 `npm install` 后自动生效
- 搭配 `lint-staged` 可以只检查暂存区文件，速度更快

### 方案三：pre-commit 框架（多语言通用）

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
```

```bash
# 安装 pre-commit 并激活
pip install pre-commit
pre-commit install
```

> **注意**
>
> 1. 方案一（`core.hooksPath`）是最通用、最轻量的方案，不依赖任何外部工具
> 2. Husky 适合 Node.js/JavaScript 项目，配置简单且生态成熟
> 3. pre-commit 框架适合 Python 或多语言项目，提供大量现成钩子
> 4. 无论使用哪种方案，建议在项目 README 中注明钩子配置方式，方便新人上手

## 调试钩子

| 问题 | 排查方法 |
| :--- | :------- |
| 钩子未触发 | 检查文件名是否正确、是否有可执行权限、`core.hooksPath` 是否配置正确 |
| 钩子执行失败 | 手动执行 `bash .git/hooks/hook-name` 测试脚本逻辑 |
| 跳过钩子 | `git commit --no-verify` 跳过 pre-commit 和 commit-msg；`git push --no-verify` 跳过 pre-push |
| 查看钩子路径 | `git config core.hooksPath`（如未配置则默认 `.git/hooks/`） |

# 综合示例

以下是一个实际可用的 pre-commit 脚本，整合了多个检查：

```bash
#!/bin/sh
# .git/hooks/pre-commit —— 一站式提交前检查

# 步骤 1: 检查暂存区是否有调试代码
if git diff --cached | grep -E "(console\.log|debugger|FIXME|TODO)" > /dev/null; then
    echo "--- pre-commit: 检测到调试代码 ---"
    echo "请移除 console.log / debugger / TODO 后重新提交"
    exit 1
fi

# 步骤 2: 运行代码格式化检查
npm run format-check 2>/dev/null
if [ $? -ne 0 ]; then
    echo "--- pre-commit: 代码格式检查未通过 ---"
    echo "请运行 npm run format 修复格式"
    exit 1
fi

# 步骤 3: 运行 lint（仅暂存区文件）
npx lint-staged 2>/dev/null
if [ $? -ne 0 ]; then
    echo "--- pre-commit: Lint 检查未通过 ---"
    exit 1
fi

echo "--- pre-commit: 所有检查通过 ✓ ---"
exit 0
```

# 最佳实践

1. **轻量检查放 pre-commit，重量检查放 pre-push** —— 提交前检查应在几秒内完成；耗时长的测试放在推送前
2. **通过 `core.hooksPath` 或 Husky 共享钩子**，不要让每个开发者手动复制 `.git/hooks/` 脚本
3. **永远不要禁用在 main 分支上的 pre-receive 钩子**（服务端）—— 它是保护生产分支的最后一道防线
4. **钩子脚本也应纳入版本管理** —— 使用 `githooks/` 目录或 `.husky/` 目录，并在项目文档中说明配置方法
5. **CI/CD 环境中可跳过本地钩子** —— CI 流水线通常不需要本地钩子，可通过环境变量检测跳过
