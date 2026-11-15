# 口语易答案查看器

一个 Windows 桌面小工具：读取本机口语易（听说练习）客户端缓存的试卷包，把**题目、参考答案、得分要点**整理出来，并支持用**本地神经网络语音**把答案读出来。

单文件 exe（约 91 KB），不需要安装 Python、不需要任何运行时依赖（用 Windows 自带的 .NET Framework 4.x）。

![主界面](docs/screenshots/main.png)

---

## 功能

- **试卷列表**：自动扫描本机已解包的试卷，点选即看
- **答案速查**：勾选后压缩成「题号 → 首选答案」清单
- **参考答案分级**：每题列出服务端给出的所有可接受表述，最完整的标 ★
- **本地语音合成**：Piper(VITS) ONNX 模型，中英文各一个，离线、CPU 推理约 10 倍实时
- **单题播放 / 批量生成**：可挑单题朗读，也可整卷生成 wav（自动存到 `exe目录\答案语音\试卷号\`）
- **输出设备可选**：默认扬声器，或 `CABLE Input` 之类的虚拟声卡（可当作麦克风输入）
- **导出**：Word（手写 OOXML，无需 Office）、HTML、Markdown
- **在线取卷**：输入试卷编号，借客户端已登录会话取到该卷口令，下载并解包

![批量生成完成](docs/screenshots/batch-done.png)

## 快速开始

1. 下载 `口语易答案查看器.exe`（Releases 里附带，或按下文自行编译），双击运行（首次运行 Windows 可能提示“已保护你的电脑”，点“更多信息 → 仍要运行”）。
2. 左侧选择已有试卷；没有试卷时，输入编号（例如 `11895`）点「获取/更新该试卷」。
3. 想听答案：点某道题 → 点「▶ 朗读答案」，或直接用「题目」下拉框挑一题播放。

首次使用语音功能需要下载模型（约 150 MB），点「下载/修复语音模型」，程序会用 hf-mirror / GitHub 代理自动拉取：

```
%LOCALAPPDATA%\KyyAnswers\tts\
  piper\piper.exe          语音引擎（含 onnxruntime、espeak-ng-data）
  models\en_US-lessac-medium.onnx    英文女声
  models\zh_CN-huayan-medium.onnx    中文女声
```

## 工作原理

1. **试卷从哪来**：口语易把每套听说练习打包成 `zipPackage/<试卷编号>.zip`，客户端解包到 `unit_zip_dl\<版本>\<编号>\`。包里 `data.json` 保存了整卷结构：
   `partList[] → actionList[] → questionList[]`，每题的 `evalContent` 就是评测基准（多条可接受表述用 `|` 分隔），`answer` 是客观题答案，`keypoints` 是采分点。
2. **包是加密的**：zip 用 ZipCrypto 加密，口令是客户端向服务器请求试卷详情时拿到的 32 位 `encryptKey`（不是 `data.json` 里那个 64 位的）。程序通过 CEF 调试端口在客户端页面里调用 `kyyQuery(...)` 取得口令，再用自带的 ZIP 解析 + ZipCrypto 解密。
3. **语音**：Piper 模型本地推理，`--length_scale` 控制语速；播放走 winmm `waveOut`，可指定输出设备。

> 程序不采集、不上传任何数据；所有解析和合成都在本机完成。

## 目录结构

```
src\KyyViewer.cs    1365 行  数据模型 / ZIP 解密 / 取口令 / 导出 / 界面
src\Tts.cs           270 行  Piper + SAPI 语音合成
src\AudioOut.cs      199 行  waveOut 播放，可指定设备
src\build.ps1               编译脚本（调用系统自带 csc.exe）
src\build.cmd               双击编译
src\app.ico                 图标
docs\screenshots\           截图
```

更细的类/方法索引见 [src\源码说明.md](src/源码说明.md)。

## 从源码编译

需要 Windows 自带的 .NET Framework 4.x（Win10/11 默认有），无需安装 Visual Studio。

```bat
cd src
build.cmd
```

或者：

```powershell
powershell -ExecutionPolicy Bypass -File src\build.ps1
```

编译产物在仓库根目录：`口语易答案查看器.exe`。

> 编译器是 `%WINDIR%\Microsoft.NET\Framework64\v4.0.30319\csc.exe`，引用了
> WinForms / Drawing / Web.Extensions / WebSockets / Compression / System.Speech。

## 命令行参数

```
口语易答案查看器.exe <试卷编号>                 直接打开某卷
口语易答案查看器.exe --fetch <编号>              只下载并解包
口语易答案查看器.exe --export <编号> <文件>      导出 .md / .html / .docx
口语易答案查看器.exe --say "文本" <wav> [音色]    合成一句
口语易答案查看器.exe --batch <编号> <目录>       批量生成整卷音频
```

## 常见问题

**Q：提示“还没有本机试卷”？**
A：说明本机没有解包过的卷。输入编号点「获取/更新该试卷」，程序会自动拉起口语易客户端取口令。

**Q：取卷失败 / 一直转圈？**
A：需要本机装有口语易客户端且能自动登录（记住密码）。程序会用 `--remote-debugging-port=9222` 启动它；若 9222 被占用，先关掉已运行的客户端再试。

**Q：下载模型很慢？**
A：默认走 `hf-mirror.com`，失败会回退 GitHub 代理。也可以手动把 `piper` 和两个 `.onnx` 放到上面列出的目录。

**Q：想把语音“喂”给麦克风？**
A：输出设备选 `CABLE Input`（需装 VB-Audio Virtual Cable），再把 Windows 默认录音设备设为 `CABLE Output`。

**Q：杀毒软件报毒？**
A：程序是自编译的未签名 exe，部分杀软会提示。可以自行用源码编译，或加入信任。

## 免责声明

本项目仅用于**本机已购账号的练习卷答案核对与跟读练习**。程序不修改客户端、不伪造录音、不向任何服务器提交数据。
请勿用于代考、刷分或任何违反服务条款的用途，使用产生的后果由使用者自负。

## License

MIT，见 [LICENSE](LICENSE)。
