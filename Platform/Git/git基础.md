# 基本概念

## git是什么？

git 是一款**分布式版本控制系统**（VCS），核心作用是：

- 追踪文件 / 代码的修改历史（谁改、改了啥、啥时候改）；
- 随时回滚到任意历史版本（不怕改错、删错）；
- 多人协作时隔离 / 合并代码（避免冲突）。

> 对比理解：可以把 git 想象成 "代码的时光机"，本地就能记录所有修改，远程服务器只是 "备份 / 共享用的云端"。

## git的核心区域

所有本地 Git 操作，都是围绕这 3 个区域流转

|  区域  |       通俗理解       |               关键特征                |
| :----: | :------------------: | :-----------------------------------: |
| 工作区 | 你正在编辑的文件目录 |    看得见、可直接修改的文件都在这     |
| 暂存区 | 待提交的 "临时清单"  | 存放你想纳入版本管理的修改（过渡区）  |
| 版本库 |   git 的本地数据库   | 最终保存的版本历史（存在.git 文件夹） |

## 本地仓库 vs 远程仓库

|   维度   |           本地仓库            |  远程仓库（GitHub/Gitee/GitLab）  |
| :------: | :---------------------------: | :-------------------------------: |
| 存储位置 |          自己的电脑           |     互联网 / 公司内网的服务器     |
| 依赖网络 |     不需要（断网也能用）      |     必须联网（推送 / 拉取时）     |
| 核心作用 |     本地版本管理（核心）      |      备份代码、多人协作共享       |
| 交互方式 | 直接操作（git add/commit 等） | 需主动命令（git push/pull/clone） |
| 数据归属 |          仅自己可见           |  可设置私有 / 公开，协作者可访问  |

> **关键结论**：
>
> 本地仓库是 Git 的 "主战场"，远程仓库只是 "辅助工具"—— 哪怕没有远程仓库，本地也能完整管理版本；只有执行`git push`，本地代码才会上传到远程。

# 环境配置

## Windows 安装

1. **下载与安装**

   - **下载地址**：[Git 官方 Windows 版下载页](https://git-scm.com/download/win) 

   - 关键选项说明（无需修改，按默认即可）：
     - 安装路径：默认 `C:\Program Files\Git`（避免中文 / 空格路径）；
     - 初始分支名：默认选「main」（符合当前主流规范，替代老版本的 master）；
     - 换行符转换：默认选「Checkout Windows-style, commit Unix-style line endings」（解决跨系统换行符混乱问题）；
     - 安装完成后，右键桌面 / 文件夹，能看到「Git Bash Here」「Git GUI Here」即安装成功。

2. **验证安装**

    打开「Git Bash」（开始菜单搜索 Git Bash），执行以下命令：

    ```bash
    git --version
    ```

    输出类似 `git version 2.43.0.windows.1` 即安装成功。

3. **可选优化配置**

    ```bash
    # 1. 配置换行符（避免跨系统代码格式问题）
    git config --global core.autocrlf true
    
    # 2. 配置默认编辑器为 VS Code（要求已经安装了vscode，并且配置了环境变量）
    git config --global core.editor "code --wait"
    
    # 3. 开启命令行颜色高亮（日志/状态更易读）
    git config --global color.ui auto
    ```

## Linux 安装

1. **安装 Git**

    - **Ubuntu/Debian 系列**：

      ```BASH
      sudo apt update
      sudo apt install git -y
      ```

    - **CentOS/RHEL 系列**：

      ```BASH
      sudo yum install git -y
      ```

2. **验证安装**

    在终端执行：

    ```BASH
    git --version
    ```

    输出类似 `git version 2.34.1` 即安装成功。

3. **可选优化配置**

    ```bash
    # 1. 配置换行符（Linux 默认 LF 格式）
    git config --global core.autocrlf input
    
    # 2. 配置默认编辑器为 VS Code（要求已经安装了vscode，并且配置了环境变量）
    git config --global core.editor "code --wait"
    
    # 3. 开启命令行颜色高亮（日志/状态更易读）
    git config --global color.ui auto
    
    # 4. （可选）升级 Git 到最新版本（Ubuntu 示例）
    sudo add-apt-repository ppa:git-core/ppa
    sudo apt update
    sudo apt upgrade git -y
    ```

## 用户身份配置（必做，所有平台通用）

Git 需要识别你的身份来记录提交者信息，在终端中执行（替换成你的昵称和邮箱）：

```BASH
# 全局配置（所有本地仓库生效）
git config --global user.name "你的昵称"
git config --global user.email "你的邮箱@xxx.com"

# 验证配置是否生效
git config --global --list
```

## 管理已有配置

**核心概念**

Git 的配置分「全局」（所有仓库生效）和「局部」（仅当前仓库生效）。

**常用命令**

- **修改已配置的项**：

    ```bash
    git config --global 项名 新值
    # 实例：修改用户名、修改邮箱
    git config --global user.name "新的昵称"
    git config --global user.email "你的邮箱@xxx.com"
    ```

- **删除某一项配置**：

    ```BASH
    git config --global --unset 项名
    # 示例：删除默认编辑器配置
    git config --global --unset core.editor
    ```

- **查看所有配置文件**：

    ```BASH
    git config --list --show-origin
    ```

# .gitignore 配置

**核心概念**

`.gitignore` 文件用于告诉 Git **忽略哪些文件**，让它们不被纳入版本管理。常见场景：编译产物、依赖包、日志文件、敏感配置等不应提交的文件。

> 核心价值：保持仓库干净，避免误提交无关文件（如 `node_modules`、`.env`、`*.log` 等）。

**忽略规则语法**

| 语法               | 说明                                           | 示例                              |
| :----------------- | :--------------------------------------------- | :-------------------------------- |
| `*.后缀`            | 匹配所有同后缀文件                             | `*.log` 忽略所有日志文件          |
| `**/目录/`          | 匹配任意层级下的指定目录                       | `**/node_modules/` 忽略所有嵌套的 node_modules |
| `目录/`             | 匹配当前目录下的指定目录                       | `build/` 忽略 build 目录          |
| `?`                | 匹配单个任意字符                               | `file?.txt` 匹配 file1.txt/fileA.txt |
| `[abc]`            | 匹配括号内任意一个字符                         | `file[12].txt` 匹配 file1.txt/file2.txt |
| `!`                | 取反，让匹配到的文件不被忽略（需先被忽略）     | `!important.log` 不忽略 important.log |

**常用场景示例**

- **Node.js 项目**：

    ```
    node_modules/
    dist/
    .env
    *.log
    npm-debug.log*
    ```

- **Python 项目**：

    ```
    __pycache__/
    *.py[cod]
    *.egg-info/
    .venv/
    venv/
    .env
    *.log
    ```

- **Java 项目**：

    ```
    target/
    *.class
    *.jar
    *.war
    .idea/
    *.iml
    ```

- **通用（IDE / 系统文件）**：

    ```
    .vscode/
    .idea/
    *.swp
    *.swo
    *~
    .DS_Store
    Thumbs.db
    ```

> **注意**
>
> 1. `.gitignore` 文件本身应该被提交到版本库（团队共享忽略规则）
> 2. 忽略规则**仅对未跟踪文件生效**——已经被 Git 跟踪的文件，即使写入 `.gitignore` 也仍会被管理，需先用 `git rm --cached` 取消跟踪
> 3. 每个目录下都可以有各自的 `.gitignore`，规则从上到下逐级生效
> 4. 全局忽略规则可配置 `git config --global core.excludesfile ~/.gitignore_global`

# 推荐学习路径

基础概念和环境准备就绪后，按以下顺序逐步深入：

**第 1 步：创建第一个本地仓库** → [本地仓库管理.md](本地仓库管理.md)

掌握 init、status、add、commit、log、diff、reset、restore 等核心本地操作，理解"工作区 → 暂存区 → 版本库"的完整流转。

**第 2 步：学习分支管理** → [分支管理.md](分支管理.md)

Git 最强大的功能——让团队在不同开发线上并行工作。涵盖 branch、checkout、switch、merge、rebase、cherry-pick、stash。

**第 3 步：远程仓库协作** → [远程仓库管理.md](远程仓库管理.md)

备份代码与多人协作。涵盖 clone、remote、fetch、pull、push、远程分支管理、冲突解决。

**第 4 步：版本标记与发布** → [标签管理.md](标签管理.md)

为里程碑提交打上版本号标记，理解轻量标签与附注标签的区别。

**第 5 步：Git 钩子自动化** → [Git钩子管理.md](Git钩子管理.md)

将重复检查自动化——在提交、推送等节点自动执行代码检查、格式校验、测试套件。
