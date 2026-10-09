# publish-to-fork.yml 中的 Bash 命令详解

本文逐个解释 `.github/workflows/publish-to-fork.yml` 里出现的**所有** bash 命令和语法——
不只是 `grep`、`git` 这类"看得见的命令"，也包括 `|`、`>>`、`${VAR#v}`、`if...then...fi`
这些"符号和关键字"。条目按 **常见 → 不常见** 排序。

---

## 0. 先弄清楚：这些脚本是 how 运行的

在读具体命令之前，有 5 个 YAML 层的事实需要知道，它们决定了 bash 脚本的行为：

1. **`runs-on: ubuntu-latest`**：每个 job 都在 GitHub 提供的一台临时 Ubuntu 虚拟机里执行，
   跑完即销毁。你本机是不是 Windows 与此无关。
2. **`run: |`**：YAML 的"多行文本块"。竖线 `|` 是 YAML 语法，作用是"保留换行"——
   之后缩进的所有行会拼成一段脚本交给 bash 执行。
   注意它和 bash 的管道 `|`（见第五节）只是长得像，完全不是一回事。
3. **失败即中止**：GitHub 用 `bash -e` 的方式执行脚本——任何一条命令返回非 0（失败），
   整个 step 立刻失败。所以脚本里的 `exit 1`、`if` 判断是"报警刹车"。
4. **每个 step 是独立的 shell**：上一个 step 里 `cd` 过、设过的普通变量，下一个 step 一概不知。
   想跨 step 传值要用 `$GITHUB_ENV`（见第五节）。
5. **`working-directory: packages`**（第 70 行）：执行前先切换到该目录再跑脚本，
   相当于脚本开头有一句 `cd packages`。

---

## 一、最常见的基础命令

### 1.1 echo —— 输出文本

文件中的用法：

- 第 24 行 `echo "Tag: ${TAG}, typst.toml: ${TOML_VERSION}"` —— 打印信息到日志，方便排查。
- 第 29 行 `echo "VERSION=${TAG}" >> "$GITHUB_ENV"` —— 输出并追加写入文件。
- 第 56 行 `echo "::error::..."` —— 输出特殊格式文本（见第九节）。

双引号内的 `${变量名}` 会被替换成变量的值，这叫"变量展开"。

### 1.2 cd —— 切换目录

第 47 行 `cd packages`：进入子目录 `packages`，之后的命令都在这个目录里执行。

记住两点：

- `cd ..` 回上级目录。
- 因为每个 step 是独立 shell，一个 step 里 cd 不影响下一个 step（见第 0 节第 4 条）。

### 1.3 mkdir -p —— 创建目录

第 59 行 `mkdir -p "${DEST}"`。

- `mkdir` = make directory。
- `-p` 有两层含义：**父目录不存在就一并创建**；**目录已存在也不报错**（不加 `-p` 时这两种情况都会失败）。
  `DEST` 是 `packages/packages/preview/theoframe/0.4.2` 这样层层嵌套的路径，所以必须加 `-p`。

### 1.4 cp -rv —— 复制文件

第 61–67 行：

```bash
cp -rv \
  assets \
  lib.typ \
  LICENSE \
  README.md \
  typst.toml \
  "${DEST}/"
```

- 语法：`cp [选项] 源1 源2 ... 目标目录/`。**最后一个参数是目标**，其余全是源。
- `-r`（recursive）：递归复制整个文件夹——`assets` 是目录，不加 `-r` 会报错跳过。
- `-v`（verbose）：把每个复制动作打印出来，方便在 workflow 日志里核对。
- 目标末尾的 `/` 明确表示"复制到这个目录里面"。

---

## 二、git 命令（按文件中出现顺序）

### 2.1 git clone —— 克隆远程仓库

第 45–46 行：

```bash
git clone --depth 1 --no-checkout --filter="tree:0" \
  "https://x-access-token:${GH_TOKEN}@github.com/hexiongwu1995/packages.git" packages
```

`typst/packages` 有几万个包、体积巨大，三个选项全是为了"少下载"：

| 选项 | 含义 |
|---|---|
| `--depth 1` | 浅克隆：只下载最新一次提交，不要全部历史 |
| `--no-checkout` | 只下载仓库元数据，**不**把文件放进工作区 |
| `--filter="tree:0"` | 部分克隆：连目录树信息都先不下载，之后用到哪个再按需取 |

最后的 `packages` 指定本地目录名。

URL 里嵌了 `${GH_TOKEN}`：推送到别人的仓库需要认证，这里把 PAT（个人访问令牌）
直接嵌进 HTTPS 地址。`GH_TOKEN` 来自 step 的 `env:` 定义，是 GitHub Secrets 里的值。

### 2.2 git sparse-checkout —— 稀疏检出

第 48–49 行：

```bash
git sparse-checkout init
git sparse-checkout set packages/preview/theoframe/
```

- `init`：给当前仓库开启"稀疏检出"功能。
- `set <目录>`：声明"我的工作区只要这个目录"。

和上面的 `--filter` 配合，最终只下载 theoframe 包相关的文件，避免拉下整个仓库。

### 2.3 git checkout —— 检出 / 切换分支

两种用法都出现在文件里：

- 第 50 行 `git checkout main`：检出 main 分支（此时按稀疏规则填充工作区文件）。
- 第 74 行 `git checkout -b "theoframe-${VERSION}"`：`-b` 表示**创建新分支并切换过去**。
  比如 `VERSION=0.4.2` 时，新分支叫 `theoframe-0.4.2`。

### 2.4 git config —— 设置提交者信息

第 72–73 行：

```bash
git config user.name "github-actions[bot]"
git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
```

runner 是每次全新启动的机器，没有任何 git 身份配置；不设置的话 `git commit` 会直接失败。

### 2.5 git add / commit / push —— 暂存、提交、推送

第 75–77 行：

```bash
git add "packages/preview/theoframe/${VERSION}"
git commit -m "theoframe:${VERSION}"
git push -u --force origin "theoframe-${VERSION}"
```

- `git add <路径>`：把文件放入暂存区。新文件必须 add 过，commit 才会包含它。
- `git commit -m "<说明>"`：提交，`-m` 后跟提交信息。
- `git push -u --force origin <分支>`：
  - `origin` 是远程仓库的默认名字（即克隆来源那个 fork）。
  - `-u`（--set-upstream）：建立本地分支与远程分支的关联，首次推送新分支时用。
  - `--force`：如果远程已有同名分支就直接覆盖。这是"重跑 workflow 时允许覆盖旧发布分支"的手段。

---

## 三、gh 命令（GitHub 官方命令行）

`gh` 在 GitHub runner 上预装。当环境变量 `GH_TOKEN` 存在时，gh 自动用它认证，不用手动登录。

### 3.1 gh repo sync —— 同步 fork

第 34 行 `gh repo sync hexiongwu1995/packages --branch main`：
把 fork 仓库的 main 分支从上游 `typst/packages` 同步到最新，
保证后面添加新版本目录时基于的是最新代码。

### 3.2 gh release view —— 查询 release

第 118 行：

```bash
if gh release view "${GITHUB_REF_NAME}" --repo "${GITHUB_REPOSITORY}" >/dev/null 2>&1; then
```

查看指定 tag 的 release 信息。关键点：**release 不存在时，命令返回非 0**——
这正是把它放进 `if` 的原因：存在 → 真 → 进 `then` 删除它。
`>/dev/null 2>&1` 把查询输出的文本全部丢弃（见第五节），我们只关心"存在与否"。

### 3.3 gh release delete —— 删除 release

第 119 行 `gh release delete "${GITHUB_REF_NAME}" --repo ... --yes`：
`--yes` 跳过交互式确认——runner 里没有人可以回答 "y/N"。

### 3.4 gh release create —— 创建 release

第 121–124 行：

```bash
gh release create "${GITHUB_REF_NAME}" \
  --repo "${GITHUB_REPOSITORY}" \
  --title "theoframe ${GITHUB_REF_NAME}" \
  --generate-notes
```

- 以 tag `v0.4.2` 创建 GitHub Release。
- `--title` 设置标题。
- `--generate-notes` 让 GitHub 根据提交历史自动生成 release 说明。

---

## 四、管道与重定向

### 4.1 `|` 管道

第 23 行：`grep -m1 '^version' typst.toml | sed -E 's/.../.../'`

把**左边命令的输出**接到**右边命令的输入**上，像流水线一样串联。
这一行拆开看：

1. `grep` 从 `typst.toml` 里筛出 version 所在行 → 输出 `version = "0.4.2"`
2. 管道把这一行递给 `sed`
3. `sed` 从中抠出引号里的 `0.4.2`

### 4.2 `>` 重定向（覆盖写）

第 118 行 `>/dev/null`：把命令的标准输出改道写进 `/dev/null`——
一个"黑洞"文件，写进去的内容全部消失。用途：命令输出不需要看，保持日志干净。

### 4.3 `>>` 重定向（追加写）

第 29 行：`echo "VERSION=${TAG}" >> "$GITHUB_ENV"`

`>` 会覆盖整个文件，`>>` 是**追加到文件末尾**。

`$GITHUB_ENV` 是 GitHub 提供的特殊文件路径：往里写一行 `KEY=value`，
**之后的 step 就能用 `$KEY` 读到这个变量**。这就是为什么第 2 步写入的 `VERSION`，
第 4、5 步（复制文件、推送分支）能直接用 `${VERSION}`。

### 4.4 `2>&1` —— 合并错误输出

第 118 行：`gh release view ... >/dev/null 2>&1`

bash 里每条命令有两条输出通道：`1` = 标准输出（正常内容），`2` = 错误输出（报错内容）。

- `2>&1` 意思是"把 2 号通道接到 1 号通道当前指向的地方"——即让报错也走正常输出的路（这里是黑洞）。
- **顺序重要**：`>/dev/null 2>&1` 才是"输出和报错全丢弃"；
  写反成 `2>&1 >/dev/null`，报错仍会显示在日志里。

---

## 五、变量与展开

### 5.1 变量赋值

第 22 行 `TAG="${GITHUB_REF_NAME#v}"`、第 54 行 `DEST="packages/.../${VERSION}"`。

- 语法 `名字=值`，**等号两边不能有空格**。
  `TAG = "x"` 不会报语法错，而是被理解为"运行名为 TAG 的命令"——经典坑。
- 值用双引号包住，防止内容里有空格时被拆成多个参数。

### 5.2 `$VAR` 与 `${VAR}`

两种写法完全等价，都是取变量值。推荐 `${VAR}`，因为边界清晰：
`"theoframe-${VERSION}"` 若写成 `"$VERSIONabc"`，bash 会去找名为 `VERSIONabc`
的变量（不存在 → 取空值），而 `${VERSION}abc` 没有歧义。

### 5.3 `${GITHUB_REF_NAME#v}` —— 去掉开头的一段

第 22 行。这叫"参数展开"（parameter expansion）：`${变量#模式}` 表示
**取变量的值，但从开头删掉最短的一段匹配"模式"的内容**。

- `GITHUB_REF_NAME` 是 GitHub 自动注入的环境变量，值为触发本次运行的 tag 名，如 `v0.4.2`。
- `#v` 删掉开头的 `v` → 得到 `0.4.2`，用来和 `typst.toml` 里的版本号比较。
- 相关写法：`#` 最短匹配 / `##` 最长匹配；`%` / `%%` 从**结尾**删。本文件只用到了 `#v`。

### 5.4 `$(命令)` —— 命令替换

第 23 行 `TOML_VERSION=$(grep -m1 '^version' typst.toml | sed ...)`。

先执行括号里的命令，再把它的**输出文本**原位放进来——
即"把一条命令的结果存进变量"。括号里可以放任意复杂命令（含管道）。

### 5.5 环境变量一览

文件里用到的自动变量：

| 变量 | 值 | 来源 |
|---|---|---|
| `$GITHUB_REF_NAME` | 触发的 tag / 分支名（如 `v0.4.2`） | GitHub 自动注入 |
| `$GITHUB_REPOSITORY` | 当前仓库全名（`owner/repo`） | GitHub 自动注入 |
| `$GITHUB_ENV` | 跨 step 传变量的特殊文件路径 | GitHub 自动注入 |
| `$GH_TOKEN` | PAT 令牌 | step 的 `env:` ← GitHub Secrets |
| `$VERSION` | `0.4.2` | 脚本自己写入 `$GITHUB_ENV` 产生的 |

---

## 六、条件判断与流程控制

### 6.1 if ... then ... fi

文件里出现了两种形态。

**形态一：判断字符串是否相等**（用 `[ ]`），第 25–28 行：

```bash
if [ "${TAG}" != "${TOML_VERSION}" ]; then
  echo "::error::..."
  exit 1
fi
```

**形态二：判断"上一条命令是否执行成功"**（不用 `[ ]`，直接放命令），第 118–120 行：

```bash
if gh release view ... >/dev/null 2>&1; then
  gh release delete ...
fi
```

结构规则：

- `if 条件; then 结果; fi`。`fi`（if 反着写）是结束标志，必须配对出现。
- `;` 是命令分隔符，让 `then` 能和 `if` 挤在同一行。如果把 `then` 单独放一行，分号可以省略。
- 条件成立（返回 0）才执行 `then` 和 `fi` 之间的部分。

### 6.2 `[ ... ]` —— test 判断命令

`[` 其实是一个**命令**（等价于 `test`），不是标点符号！所以：

- **括号内侧必须有空格**：`[ "$A" != "$B" ]` 对，`["$A"!="$B"]` 错。
- `!=`：两个**字符串**不相等时为真（第 25 行，tag 与 toml 版本不一致就报错退出）。
- `-d 路径`：路径**存在且是目录**时为真（第 55 行 `[ -d "${DEST}" ]`，
  检查目标版本目录是否已存在——存在就报错退出，防止重复发布覆盖）。

### 6.3 exit —— 以指定状态退出

第 27、58 行 `exit 1`。

- 结束脚本并返回状态码。约定：`0` = 成功，非 0 = 失败。
- 配合第 0 节说的 `bash -e`：`exit 1` 会立刻让 step 失败、整个 workflow 停止——
  这是脚本里主动"拉响警报"的手段。

---

## 七、文本处理：grep 与 sed（Windows 用户最陌生的两个）

### 7.1 grep —— 在文件里"找行"

第 23 行：`grep -m1 '^version' typst.toml`

- 语法：`grep 模式 文件`，输出文件里**匹配模式的所有行**。
- 模式 `^version`：`^` 表示"行首"，即只匹配以 `version` 开头的行（排除注释、其他字段）。
- `-m1`（max-count）：找到 1 条匹配后立即停止。version 只有一行，这是双保险。

实际效果：从 `typst.toml` 取出 `version = "0.4.2"` 这一行。

### 7.2 sed —— 对文本做查找替换

第 23 行：`sed -E 's/version\s*=\s*"([^"]+)".*/\1/'`

- sed 是"流编辑器"，最常用的是 `s/查找/替换/`（substitute）。
- `-E`：使用扩展正则表达式，`(...)`、`+` 等不用加反斜杠。
- 逐段解读这个正则：

| 片段 | 含义 |
|---|---|
| `version` | 字面文字 version |
| `\s*` | 任意多个空白字符 |
| `=` | 等号 |
| `\s*` | 又是空白 |
| `"` | 一个引号 |
| `([^"]+)` | **捕获组**：`[^"]` = 不是引号的任意字符，`+` = 一个或多个 → 即"引号里的内容"，括号把它记下来 |
| `"` | 闭合引号 |
| `.*` | 剩余的所有内容 |

- 替换部分 `\1` 表示"第 1 个捕获组"，即刚才括号里记住的内容。
- 整体效果：`version = "0.4.2"` → `0.4.2`。

外层单引号的作用见 8.3。

---

## 八、其他小语法

### 8.1 行尾 `\` —— 续行符

第 45、61、121–123 行。一条命令太长时，行尾放 `\` 表示"下一行接着本条命令"，
效果与写成一行完全一样，纯粹为了可读。注意 `\` 之后必须**直接换行**，不能有空格。

### 8.2 `#` —— 注释

第 3–4、36–40、60 行。`#` 开头到行尾的内容是注释，bash 不执行。

### 8.3 三种引号

| 写法 | 行为 | 文件中的例子 |
|---|---|---|
| 双引号 `"..."` | 保留空格，`${VAR}` 会展开 | `"${DEST}"`、`"theoframe-${VERSION}"` |
| 单引号 `'...'` | 内容**原样**，什么都不展开 | `'^version'`、`'s/version\s*=\s*"([^"]+)".*/\1/'` |
| 不加引号 | 值里有空格会拆成多个参数 | 文件里没用（好习惯） |

单引号是正则的标配：保证 `"`、`*`、`$`、`\` 这些字符原样交给 grep/sed，
不被 bash 抢先解释。

---

## 九、长得像 bash、其实不是 bash 的东西

### 9.1 `::error::` —— GitHub Actions 日志标注

第 56 行：`echo "::error::Tag (${TAG}) does not match version in typst.toml (${TOML_VERSION})."`

命令本身是 bash 的 echo，但 `::error::` 开头的**特定格式文本**会被 GitHub Actions
解析器识别：在运行日志里渲染成醒目的红色错误标注，并汇总到摘要页。
同族还有 `::warning::`、`::notice::`。它只是"约定格式的文本"，不是 bash 语法。

### 9.2 `${{ ... }}` —— GitHub Actions 表达式

`${{ secrets.PACKAGES_PAT }}`、`${{ secrets.GITHUB_TOKEN }}`（YAML 的 `env:` 块里）。

在 bash 启动**之前**，GitHub 就把这个占位符替换成真实值了，bash 根本看不到 `${{ }}`。
它与 bash 的 `${VAR}` 是两个世界的东西：

- `${{ }}` —— YAML / workflow 层，写在工作流的 YAML 结构里；
- `${VAR}` —— bash 层，写在 `run:` 的脚本里。

### 9.3 `GITHUB_` 开头的变量

不需要定义，GitHub 运行时自动注入，清单见官方文档
"Default environment variables"（本文 5.5 节列了文件里用到的四个）。

---

## 十、速查表（按文件中出现位置）

| 行号 | 写法 | 类别 | 一句话解释 |
|---|---|---|---|
| 22 | `TAG="..."` | 变量 | 赋值；等号两边不能有空格 |
| 22 | `${GITHUB_REF_NAME#v}` | 参数展开 | 取值并去掉开头的 `v` |
| 23 | `$( ... )` | 命令替换 | 把命令输出存进变量 |
| 23 | `grep -m1 '^version' 文件` | 命令 | 取第一行 version 开头的行 |
| 23 | `\|` | 管道 | 左边输出 → 右边输入 |
| 23 | `sed -E 's/…/\1/'` | 命令 | 正则替换，捕获组提取版本号 |
| 24 | `echo "..."` | 命令 | 输出文本 |
| 25–28 | `if [ A != B ]; then … fi` | 条件 | 字符串不相等则执行 |
| 26 | `::error::` | Actions | 日志错误标注（非 bash） |
| 27 | `exit 1` | 命令 | 以失败状态结束脚本 |
| 29 | `>> "$GITHUB_ENV"` | 重定向 | 追加写入；实现变量跨 step |
| 34 | `gh repo sync 仓库 --branch main` | 命令 | 从上游同步 fork |
| 45 | 行尾 `\` | 语法 | 续行 |
| 45 | `git clone --depth 1 --no-checkout --filter=...` | 命令 | 浅 + 部分克隆 |
| 47 | `cd packages` | 命令 | 切换目录 |
| 48–49 | `git sparse-checkout init / set` | 命令 | 稀疏检出指定目录 |
| 50 | `git checkout main` | 命令 | 检出分支 |
| 54 | `DEST="..."` | 变量 | 赋值目标路径 |
| 55 | `[ -d "${DEST}" ]` | 条件 | 目录是否已存在 |
| 59 | `mkdir -p "${DEST}"` | 命令 | 递归创建目录 |
| 61 | `cp -rv 源… 目标/` | 命令 | 递归 + 显示过程地复制 |
| 72–73 | `git config user.name / user.email` | 命令 | 设置提交者身份 |
| 74 | `git checkout -b 分支` | 命令 | 创建并切换新分支 |
| 75–77 | `git add / commit -m / push -u --force` | 命令 | 暂存、提交、推送 |
| 118 | `>/dev/null 2>&1` | 重定向 | 丢弃标准输出和错误输出 |
| 118 | `if 命令; then … fi` | 条件 | 命令成功（返回 0）则执行 |
| 118–124 | `gh release view / delete / create` | 命令 | 查询、删除、创建 Release |

---

## 十一、本地练习（Git Bash 即可）

Windows 上装了 Git for Windows 就自带 **Git Bash**（开始菜单搜索 "Git Bash"），
它使用与 Ubuntu runner 完全相同的 bash 语法，可以逐条试上面的命令。

练习一：最难的 `${var#v}`：

```bash
GITHUB_REF_NAME="v0.4.2"
echo "${GITHUB_REF_NAME#v}"    # 输出 0.4.2
```

练习二：完整复现 workflow 第 2 步"校验版本"：

```bash
mkdir -p ~/bash-demo
cd ~/bash-demo
printf 'name = "theoframe"\nversion = "0.4.2"\n' > typst.toml

TAG="v0.4.2"
TOML_VERSION=$(grep -m1 '^version' typst.toml | sed -E 's/version\s*=\s*"([^"]+)".*/\1/')
echo "Tag: ${TAG}, typst.toml: ${TOML_VERSION}"

if [ "${TAG#v}" != "${TOML_VERSION}" ]; then
  echo "版本不一致"
else
  echo "版本一致，校验通过"
fi
```

（`printf` 是往文件写几行文本的命令，这里只是造一个测试用的 `typst.toml`；
`if ... else ... fi` 是文件里 if 的完整形态。）

练习三：管道与重定向的手感：

```bash
echo "hello" > demo.txt     # 覆盖写入
cat demo.txt                # 输出 hello（cat = 打印文件内容）
echo "world" >> demo.txt    # 追加
cat demo.txt                # 两行：hello / world
ls 不存在的文件 2>/dev/null  # 报错被吞掉，日志干净
```
