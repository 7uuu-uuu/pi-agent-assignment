# 实践课3：Pi Agent 源码安装与编译报告 LiYunyi

- **姓名拼音：** LiYunyi 
- **小组仓库地址：** https://github.com/open-source-slackers/pi-hunnu-2026
- **本地仓库路径：** `D:\PiAgent-fork\pi-hunnu-2026`
- **提交编号：** c3cc91632d0a1950fb122605b190233f2aa0006e

---

## 一、 环境准备

- **Node.js 版本：** `v24.22.3`
- **npm 版本：** `11.19.1`
- **Git 版本：** `2.55.0.windows.2`

更新版本前：

<img width="638" height="334" alt="version-before" src="https://github.com/user-attachments/assets/eb08c7ff-d561-4691-a111-92525413a733" />

更新版本后：

<img width="1090" height="344" alt="version-after" src="https://github.com/user-attachments/assets/a14966f3-d27e-4b50-b40b-d060a3facef7" />

## 二、 源码确认

已通过 `git clone` 命令下载小组源码。通过执行 `Get‑Location`、`git remote ‑v`、`git branch --show‑current`、`git rev‑parse HEAD`、`git status --short` 完成校验：确认本地路径、远端仓库地址、分支为`main`、提交哈希、工作区干净无修改。

<img width="1398" height="1066" alt="s1-clone-info" src="https://github.com/user-attachments/assets/516aa7b3-c354-4f6b-974d-031fa05a6d2e" />

## 三、 配置依据分析

1. **Node.js 版本要求**写在根 `package.json` 的 `engines.node` 字段中（要求 `>=22.19.0`）。
2. **构建命令读取**：`npm run build` 实际上是从根 `package.json` 的 `scripts.build` 读取命令，它会依次进入多个子包执行各自的构建脚本。

<img width="586" height="140" alt="engine-local" src="https://github.com/user-attachments/assets/485158ac-b354-46c3-9811-084e7a0dbfc5" />

<img width="2114" height="282" alt="build-local" src="https://github.com/user-attachments/assets/64787385-9659-49c8-9a36-0eb7f9a56649" />

## 四、 依赖安装

执行 `npm install --ignore-scripts` 完成项目依赖安装；`--ignore‑scripts` 作用是跳过安装阶段`postinstall`、`prepare`等生命周期脚本，不影响后续手动执行`npm run build`。在 CMD 中查看`%errorlevel%`返回`0`，代表安装无致命错误；为提高包下载速度配置淘宝镜像源。

<img width="886" height="270" alt="image" src="https://github.com/user-attachments/assets/37438632-8edb-4cac-a821-ea2197697c50" />


## 五、 构建与产物验证

执行 `npm run build` 构建成功。通过 `Test‑Path .\packages\coding‑agent\dist\bundle\cli.js` 检查产物路径返回 `True`，再运行 `node .\packages\coding-agent\dist\bundle\cli.js --version` 输出版本号，命令退出码为 `0`。单个子包编译成功不能代表整个项目构建完成，必须验证最终 CLI 产物。

**问题回答：**

- **源码在哪里：** 源码在本地的 `D:\PiAgent-fork\pi-hunnu-2026` 目录中。
- **依赖装在哪里：** 本项目使用 npm workspaces 进行多包管理，全部依赖统一安装在仓库根目录的 `node_modules`，packages 下各个子包不会独立生成`node_modules`。
- **构建执行了什么：** 执行了 TypeScript 源码编译，部分模块执行代码生成，最后对 coding‑agent 执行打包，生成 dist 目录下编译后的产物。
- **正在运行哪个版本：** 正在运行本次本地源码编译得到的开发版本。

<img width="2336" height="648" alt="build-verify" src="https://github.com/user-attachments/assets/12dac2f5-ff00-46f9-833f-a5597e0eda45" />

## 六、 启动本地源码

通过 `.\pi-test.bat` 成功启动了本地源码，并完成了一次对话测试。使用`.\pi‑test.bat`脚本可以保证加载本仓库本地源码；直接在终端输入`pi`会调用系统全局安装版本，无法验证本次编译的源码。

- **源码入口适用场景：** 适合开发调试、验证本地 `src` 目录下的源码修改。
- **dist 产物入口适用场景：** 适合验证最终构建产物（编译后的代码）是否能正常运行。

<img width="2302" height="882" alt="398fd2d3-f12b-429e-a765-4e3cec0b6231" src="https://github.com/user-attachments/assets/f1bd176b-4d86-48ee-98f8-0f43bf167586" />

<img width="2326" height="1210" alt="pi-chat" src="https://github.com/user-attachments/assets/b860cb72-ebd7-4d16-8acb-745bffc75614" />



## 七、 遇到的困难与解决过程

- **困难现象：** 执行 `npm run build` 时，卡在 `Fetching models.dev catalog...`，随后报 `BodyTimeoutError`，导致构建失败。
- **原因分析：** 国内网络环境无法直接连通 `models.dev` 获取模型目录配置元数据，该步骤仅获取配置信息，**不会下载大模型权重文件**。
- **解决过程：** 连接手机热点（或开启代理），重新执行 `npm run build`，绕过网络限制，编译构建成功。
  
<img width="2326" height="1208" alt="05b93c77-ea6e-4da4-857c-e83248e0c5a5" src="https://github.com/user-attachments/assets/b3c53ab1-a8cc-46a5-8ee1-d5a52d3b8f3a" />
