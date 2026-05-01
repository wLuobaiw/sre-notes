# 基本概念

## git是什么？

git 是一款**分布式版本控制系统**（VCS），核心作用是：

- 追踪文件 / 代码的修改历史（谁改、改了啥、啥时候改）；
- 随时回滚到任意历史版本（不怕改错、删错）；
- 多人协作时隔离 / 合并代码（避免冲突）。

> 对比理解：可以把 git 想象成 “代码的时光机”，本地就能记录所有修改，远程服务器只是 “备份 / 共享用的云端”。

## git的核心区域

所有本地 Git 操作，都是围绕这 3 个区域流转

|  区域  |       通俗理解       |               关键特征                |
| :----: | :------------------: | :-----------------------------------: |
| 工作区 | 你正在编辑的文件目录 |    看得见、可直接修改的文件都在这     |
| 暂存区 | 待提交的 “临时清单”  | 存放你想纳入版本管理的修改（过渡区）  |
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
> 本地仓库是 Git 的 “主战场”，远程仓库只是 “辅助工具”—— 哪怕没有远程仓库，本地也能完整管理版本；只有执行`git push`，本地代码才会上传到远程。

## 分支（Branch）：独立的开发线

分支是 Git **最强大**的功能之一，通俗理解：

- 主分支（主流是`main`，老版本是`master`）：存放稳定的代码；
- 功能分支（如`feature-login`）：开发新功能时，在独立分支操作，不影响主分支；
- 修复分支（如`bugfix-123`）：修复 bug 时专用，修复后合并回主分支。

> 核心价值：避免在主分支直接改代码，降低出错风险。

# 环境配置

## Windows

1. **安装 Git**

   - **下载地址**：[Git 官方 Windows 版下载页](https://git-scm.com/download/win) 

   - 安装步骤
     1. 双击下载的 `.exe` 安装包，选择好自己需要的配置，开始安装；
        1. 关键选项说明（无需修改，按默认即可）：
        2. 安装路径：默认 `C:\Program Files\Git`（避免中文 / 空格路径）；
        3. 初始分支名：默认选「main」（符合当前主流规范，替代老版本的 master）；
        4. 换行符转换：默认选「Checkout Windows-style, commit Unix-style line endings」（解决跨系统换行符混乱问题）；
        5. 安装完成后，右键桌面 / 文件夹，能看到「Git Bash Here」「Git GUI Here」即安装成功。

2. **验证安装**

    打开「Git Bash」（开始菜单搜索 Git Bash），执行以下命令：

    ```bash
    git --version
    ```

    输出类似 `git version 2.43.0.windows.1` 即安装成功。
    
3.  **核心配置（必做：用户身份）**

    Git 需要识别你的身份来记录提交者信息，在 Git Bash 中执行（替换成你的昵称和邮箱）：

    ```BASH
    # 全局配置（所有本地仓库生效）
    git config --global user.name "你的昵称"
    git config --global user.email "你的邮箱@xxx.com"
    
    # 验证配置是否生效
    git config --global --list
```

4. **可选优化配置（提升使用体验）**

    ```bash
    # 1. 配置换行符（避免跨系统代码格式问题）
    git config --global core.autocrlf true

    # 2. 配置默认编辑器为 VS Code（要求已经安装了vscode，并且配置了环境变量）
    git config --global core.editor "code --wait"

    # 3. 开启命令行颜色高亮（日志/状态更易读）
    git config --global color.ui auto
```

## Linux

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

3. **核心配置（必做：用户身份）**

    Git 需要识别你的身份来记录提交者信息，在 Git Bash 中执行（替换成你的昵称和邮箱）：

    ```BASH
    # 全局配置（所有本地仓库生效）
    git config --global user.name "你的昵称"
    git config --global user.email "你的邮箱@xxx.com"

    # 验证配置是否生效
    git config --global --list
    ```

4. **可选优化配置（提升使用体验）**

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

# 常用操作

## 配置部分

**核心概念**

Git 需要识别你的身份，才能记录 “谁提交了代码”，配置分「全局」（所有仓库生效）和「局部」（仅当前仓库生效）

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

## 本地仓库（核心）

**核心概念**

初始化本地仓库后，通过`add`把工作区的修改放到暂存区，再通过`commit`提交到版本库，完成一次版本记录。

**常用命令**

- **初始化本地仓库**

  ```BASH
  # 效果：生成隐藏的.git文件夹（版本库核心）
  git init
  ```

- **查看仓库状态**

  ```BASH
  # 场景：每次修改文件后，先看status，再决定下一步
  git status
  ```

- **将文件添加到暂存区**

  ```BASH
  git add 文件名.txt       # 添加单个文件
  git add .               # 添加所有修改/新增的文件（最常用）
  git add 文件夹/          # 添加指定文件夹
  ```

- **将暂存区修改提交到版本库**

  ```BASH
  # -m 后是提交信息，要清晰（比如“新增登录功能”“修复用户名显示bug”）
  git commit -m "第一次提交：添加项目核心文件"
  ```

- **查看提交历史**

  ```BASH
  git log                 # 详细版（显示作者、时间、版本号）
  git log --oneline       # 简洁版（只显示版本号和提交说明）
  ```

- **查看文件修改内容**

  ```BASH
  # 场景：不确定自己改了哪些内容时用
  git diff 文件名.txt
  ```

- **撤销工作区的修改**（在add到暂存区之前）

  ```BASH
  # 效果：把文件恢复到最近一次commit的状态（慎用！会丢失未提交的修改）
  git checkout -- 文件名.txt
  ```

- **撤销暂存区的修改**（文件已add，还没commit）

  ```BASH
  # 第一步：把暂存区的修改退回到工作区
  git reset HEAD 文件名.txt
  # 第二步：再撤销工作区的修改（可选）
  git checkout -- 文件名.txt
  ```

- **版本回滚**（已经commit到版本库，想回到旧版本）

  ```BASH
  git reset --hard HEAD^  # 回滚到上一个版本（HEAD^=上一个，HEAD^^=上上个）
  git reset --hard 版本号  # 回滚到指定版本（版本号从git log --oneline获取，写前6位即可）
  # 示例：git reset --hard 8a7b2c
  ```

- **删除文件并提交**（版本库层面删除）

  ```BASH
  git rm 文件名.txt
  git commit -m "删除无用的测试文件"
  ```

## 分支操作

**核心概念**

分支是独立的开发线，创建分支后可在分支里自由修改，完成后合并回主分支，不影响主分支的稳定代码。

**常用命令**

- **查看所有分支**（*标注当前所在分支）

  ```BASH
  git branch
  ```

- **创建新分支**

  ```BASH
  git branch 分支名
  ```

- **创建并切换分支**（最常用）

  ```BASH
  git checkout -b 分支名
  ```

- **切换分支**

  ```BASH
  git checkout 分支名
  ```

- **合并分支**

  ```BASH
  # 步骤：先切到主分支 → 合并目标分支
  git checkout main
  git merge feature-pay  # 把feature-pay分支的修改合并到main
  ```

- **删除分支**（合并完成后，删除无用分支）

  ```BASH
  git branch -d 分支名
  ```

## 远程仓库

**核心概念**

远程仓库是服务器上的 Git 仓库，需先关联本地仓库，再通过`push`推送本地代码，`pull`拉取远程最新代码。

**常用命令**

- **克隆远程仓库到本地**

  ```BASH
  # 场景：别人已经创建了远程仓库，你要下载到本地
  git clone 远程仓库地址
  # 示例：git clone https://gitee.com/你的账号/项目名.git
  ```

- **关联本地仓库有到远程仓库**（本地已初始化，新增远程关联）

  ```BASH
  git remote add origin 远程仓库地址
  # 示例：git remote add origin https://gitee.com/你的账号/项目名.git
  ```

- **查看已关联的远程仓库**

  ```BASH
  git remote -v
  ```

- **推送本地代码到远程仓库**

  ```BASH
  # -u：绑定本地分支和远程分支，后续直接git push即可
  git push -u origin main  # 推送到远程main分支
  ```

- **拉取远程仓库最新代码**（多人协作时，**先拉再推**！）

  ```BASH
  # 场景：别人修改了远程代码，你要同步到本地
  git pull origin main
  ```

- **解除远程仓库关联**

  ```BASH
  git remote remove origin
  ```
