---
name: web-tool-verify
description: 单文件 HTML 网页工具的快速开发与浏览器级验证套路。当需要交付本地零依赖的 HTML 工具（图片/文件批处理、表单、可视化等）并要求"双击即用、离线可用"时使用。
agent_created: true
---

# 单文件 HTML 工具开发与验证

## 适用场景
交付"一个 .html 双击即用、零依赖、不联网"的工具（批量裁剪/重命名/转换、数据看板、表单等）。file:// 协议下运行，所有逻辑内嵌单文件。

## 开发要点
1. **零依赖**:所有 JS/CSS 内嵌;需要 ZIP 打包时用内置 STORE 打包器(见下),不要引 CDN
2. **拖拽文件夹**:`webkitGetAsEntry()` 递归读目录(过滤扩展名),另备 `input[type=file] webkitdirectory` 按钮
3. **EXIF 方向**:`createImageBitmap(url, {imageOrientation:'from-image'})`,失败降级 `new Image()`
4. **裁剪/缩放算法**:目标框覆盖源图 `scale=max(W/sw,H/sh)`,`cropW=W/scale`,`srcX=(sw-cropW)*posX`,焦点 pos 取 0~1
5. **中文文件名**:ZIP 内用 UTF-8 + flag 0x0800,现代系统(Win10+/7-Zip/WinRAR)正常;Git Bash 老 unzip 报 warning 属正常,勿当 bug
6. **导出到文件夹**:`showDirectoryPicker({mode:'readwrite'})` 需 Chrome/Edge,要提示用户兜底 ZIP

## ZIP STORE 打包器(内嵌核心)
- 每个文件:Local header(30B+名字) + 数据;Central directory 记录(46B+名字);EOCD(22B)
- CRC32 查表实现 + `toBlob`/Blob 拼接,method=0 不压缩(图片本身已压缩,体积无损失)
- 关键偏移:local 的 crc@14/size@18/22/nameLen@26;central 的 crc@16/size@20/24/nameLen@28/offset@42;EOCD 的 count@8/10/cdSize@12/cdStart@16

## 验证套路(必做,按序)
1. **JS 语法**:`new Function(提取的<script>内容)` 编译,报错即改
2. **核心逻辑单测**:用 node 提取 script 中核心函数段执行断言(JSON/SRT/正则解析等逻辑类工具通用);ZIP/CRC 用 node 生成,`python -c "import zipfile;z=zipfile.ZipFile(p);z.testzip()"` 验 CRC(比 unzip 可靠)
3. **真实渲染**:Edge 无头执行 + DOM 断言,确认 JS 完整跑完:
   ```
   msedge --headless=new --disable-gpu --no-sandbox --virtual-time-budget=2000 --dump-dom file:///D:/xxx.html
   ```
   在 JS 的 IIFE 末尾永久写一句 `document.body.setAttribute('data-page-ready','1')`(生产代码里保留即可,无需验证后移除),然后 grep 断言。因为它位于 IIFE 最后一行,**只要它出现就证明整段脚本无异常跑完**,这比逐项断言中间状态更省事
4. 截图可用 `--screenshot` 但注意:部分模型不支持读图,优先用 dump-dom 断言

## 渲染验证的四个陷阱(实测踩过,别误判成代码 bug)

### 1. 虚拟时间不推进 CSS transition → 图表显示为空白
用 JS 设置 `width` / `stroke-dashoffset` 并靠 CSS `transition` 做动画的图形,在 `--virtual-time-budget` 下可能停在**起始帧**(宽度 0),截图看起来像"没渲染出来"。
- **不要据此判定 CSS 有 bug**。定位手法:生成一份把 `transition:` 换成 `none` 的副本再截图——若图形正常出现,就是虚拟时间问题,真实浏览器无碍。
- 更根本的修法见下一条。

### 2. 数值的唯一来源不该是 JS(渐进增强)
若图形宽度只由 JS 赋值,则脚本失效、rAF 被节流、过渡被跳过时,用户看到的是空轨道。
**正确写法**:
- 把最终值直接写进 HTML 内联样式(如 `style="width:90%;background:..."`、SVG 的 `stroke-dashoffset="140.7"`)——静态就是对的;
- JS 只负责入场动画:先读 `el.style.width` 存下目标 → 置 `0%` → `void document.body.offsetWidth` 强制回流 → 双 `requestAnimationFrame` 内恢复过渡并写回目标值;
- 再加一个兜底 `setTimeout(1500)`,检测到仍是 `'0%'` 就直接赋终值(此处刻意 `transition='none'`,保证立即生效)。
**验证方式**:用 Python 正则删掉整段 `<script>...</script>` 生成副本再渲染截图,断言图形依然正确——这才是"脚本失效也完整可用"的证据。

### 3. `--window-size` 有最小值,会伪装成"横向溢出"
Edge headless 的 `clientWidth` 最低约 **481**,给更小的值不会生效;截图按你指定的宽度裁切,于是右侧内容被切掉,测量时表现为"内容触达右边缘",**极易误判为元素溢出**。
- 判断真实溢出要看 `scrollWidth` 而不是截图边缘:注入诊断脚本读 `document.documentElement.scrollWidth` 与 `clientWidth` 比较;
- 要定位具体溢出元素,注入一段脚本遍历 `body *` 收集 `getBoundingClientRect().right > clientWidth + 1` 的元素,把结果写进一个 `id="OVERFLOW_REPORT"` 的 div,再用 `--dump-dom` grep 出来。**注意 `overflow-x:auto` 容器(导航滚动条、`pre` 代码块)内的元素报 right 超界是正常的**,只要 `scrollWidth == clientWidth` 就没有真实溢出。

### 4. 用锚点 hash 截取中间区块会失败
`file:///xxx.html#section` 配合 `--screenshot` 常拿到空白图——平滑滚动尚未完成,且滚动显现元素还没被 IntersectionObserver 触发。可靠做法:**直接截一张超长整页**(如 `--window-size=1400,11200`,`--hide-scrollbars`),再用 PIL 按 y 坐标裁剪分段查看,顺便用"内容行范围"检测出页面真实高度。

## 需要读写本地磁盘时：加一个零依赖 Node 本地服务
单文件 HTML 在 `file://` 下读不到任意路径、也放不住密钥。这时别在前端堆奇技淫巧，直接配一个 `server.mjs`（Node 内置 `http`，零依赖）：
- 分工：**服务端只做文件系统 + HTTP + 密钥保管；图像处理交给浏览器 Canvas**（缩放、拼版、编码 JPEG）。这样 Node 侧连图像库都不用装。
- 页面用 `fetch` 调 `/api/*`；进度走 `EventSource`(SSE)，连接时**补发历史日志与当前状态**，刷新页面不丢进度。
- 只绑 `127.0.0.1`，端口被占自动 `+1` 重试；所有路径经 `safeJoin(root, ...)` 校验，杜绝目录穿越。
- 要打 ZIP 就在服务端做 Node 版 STORE 打包器（`Buffer` + CRC32，结构同上面的前端版），成品包体积几乎无损。
- 用户入口给一个 `启动.bat`（`chcp 65001` + `cd /d "%~dp0"` + `node server.mjs --open`）。
- **界面连不上服务时，必须在页面正文里渲染一段"请先启动服务"的指引**，而不是只弹 toast——用户很可能直接双击了 html。
- 不需要 Node 依赖就能验证：写个 mock 端点返回固定图片，跑全链路断言，零外部成本。见 `autohome-image-batch` 的 `.workbuddy/tmp/test-e2e.mjs`。

## 长页面报告类的结构套路(适用于评审/复盘/方案页)
- 顶部 sticky 导航 + `scroll-padding-top` 抵消吸顶高度;滚动高亮用 rAF 节流
- 危险等级用 **pill 徽章 + 左侧色条**双重编码,不要只靠颜色(色盲友好)
- 结论/评分前置,证据(对照实验、验证矩阵)放中段,遗留项与边界诚实收尾
- 所有折叠面板默认折叠、只展开最重要的第一个;纯 CSS `display:none` 切换即可,不必上框架
- 打印样式里把 `.issue-body{display:block !important}` 展开,并隐藏导航,方便一键存 PDF


## Windows 坑(高频踩)
- node 把 `/tmp` 解析为 `C:\tmp`(不存在会 ENOENT),临时文件写项目内绝对路径
- `unzip -t` 对 UTF-8 名返回 exit code 2,`&&` 链会中断 → 残留文件,记得清理
- GBK 控制台中文乱码属显示问题,不影响文件本身
- **Git Bash heredoc 写 JS 测试脚本会吃反斜杠**:`cat > t.js <<'EOF'` 即使 quoted heredoc,`\\` 也可能变 `\`,导致正则字面量/字符串转义静默失效,测试结果失真。含反斜杠(正则、ASS标签、JSON转义)的测试脚本一律用 Write 工具写,不要用 heredoc。含 `${...}` 的脚本同理(会被当成变量替换,报 `Bad substitution`)
- **`path.resolve(base, '/x')` 会跑到盘符根**:静态文件服务里若把以 `/` 开头的 `url.pathname` 直接交给 `path.resolve(publicDir, rel)`,Windows 上会解析成 `D:\x` 而被判越界(静态页返回 403)。先 `rel = pathname.replace(/^\/+/, '')` 再交给 `safeJoin`
- 用端口找进程再杀:`netstat -ano | grep ':端口' | grep -i listening | awk '{print $5}'` 取 PID,再 `node -e "process.kill(PID)"`（Git Bash 里 `taskkill //PID` 会被转义,别用）
