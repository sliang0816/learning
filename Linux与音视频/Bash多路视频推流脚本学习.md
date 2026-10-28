# Bash 多路视频推流脚本学习

学习日期：2026-10-28。范围：仅学习 `local_streams_start.sh` 所涉及的 Bash、FFmpeg 和启动管理知识。示例使用通用路径，不包含业务视频、服务密钥或完整项目源码。

## 1. 先弄清脚本在做什么

这个脚本将磁盘上的视频文件循环推送到本地流媒体服务器。它的工作顺序是：准备视频列表 → 清理旧推流 → 检查配置和端口 → 归档选流缓存 → 探测视频编码 → 确认流媒体服务 → 启动多个 FFmpeg → 等待并在退出时清理。

这里的推流不是目标检测。编码是 H.264 的文件直接复制视频数据；HEVC 文件通过 GPU 解码再编码成 H.264。两者最后都封装为 FLV，通过 RTMP 发送。

学习源码入口：`project/camera_sys/tools/local_streams_start.sh`。以下只摘取通用语法，理解后再对照原文件阅读。

## 2. Bash 基础：理解命令怎样组成

### 解释器与失败处理

```bash
#!/usr/bin/env bash
set -euo pipefail
```

- `bash` 指定脚本解释器；数组、进程替换等写法不能随意换成 `sh` 执行。
- `-e`：许多未处理的命令失败会使脚本退出，但 `if` 条件、`&&`、`||` 等有例外，不是遇到任何非零状态都立刻退出。
- `-u`：引用未设置变量通常会报错。
- `pipefail`：管道状态由最后一个失败命令决定；默认通常只看最后一个命令。

遇到允许失败的清理命令时，常见写法为 `some_command || true`。它会吞掉失败状态，应仅用于可接受的失败，不能代替错误处理。

### 路径、变量和命令替换

```bash
ROOT="$(cd "$(dirname "$0")/../../.." && pwd)"
cd "$ROOT"
```

`$0` 是启动脚本时使用的路径，`dirname` 取父目录；`$(...)` 执行命令并取得输出。上述代码从脚本位置计算项目根目录，使脚本可以从不同工作目录启动。它不等于解析所有符号链接后的真实路径。

变量展开应优先加双引号：`"$video"` 将带空格的文件名作为一个参数。这里双引号允许变量展开，而单引号通常保留文字原样。赋值写 `video=...`，等号两侧不能随便加空格。

### 数组与循环

```bash
videos=( '/example/a.mp4' '/example/b with spaces.mp4' )
videos+=("${videos[@]}")
for index in "${!videos[@]}"; do
  stream=$((index + 1))
  video="${videos[$index]}"
done
```

`"${videos[@]}"` 逐个展开数组元素；`"${!videos[@]}"` 展开数组下标。数组下标从 0 开始，显示用的流编号可以通过 `$((index + 1))` 转成从 1 开始。追加自身会把当前列表复制一遍，并没有复制磁盘视频文件。

`codecs` 和 `videos` 是两个按相同下标对应的数组。增加、删除或重排时必须保持对应关系。

## 3. 条件、退出码、管道和重定向

| 写法 | 含义 | 脚本中的用途 |
|---|---|---|
| `[[ -f "$video" ]]` | 判断是否是普通文件 | 检查输入存在 |
| `[[ "$codec" == h264 ]]` | 字符串判断 | 选择直接复制分支 |
| `! command` | 反转退出状态 | 命令失败时进入处理 |
| `cmd1 && cmd2` | 前一个成功才执行后一个 | 成功后继续操作 |
| `cmd1 || cmd2` | 前一个失败才执行后一个 | 处理失败或容忍失败 |
| `cmd1 \| cmd2` | 把前者标准输出交给后者 | 将端口列表交给匹配命令 |
| `>/dev/null` | 丢弃标准输出 | 只关心成功与否 |
| `2>/dev/null` | 丢弃标准错误 | 隐藏可接受的清理报错 |
| `exit 1` | 以失败状态退出 | 停止后续启动 |

`grep -q` 不打印匹配内容，以退出状态表达有没有匹配。`grep -A1` 只读取匹配行和后面一行，不是真正解析 YAML；字段位置变化后可能检查失准。

`ss -lnt` 查看监听中的 TCP 端口；监听 1935 只证明端口被监听，不能证明服务身份正确或某路视频已经推送成功。端口检查与推流检查是两件事。

可用 `command -v ffmpeg`、`command -v ffprobe` 检查命令能否找到；它们不验证编码器、GPU 支持或驱动是否兼容。

## 4. 进程替换、进程匹配与信号

```bash
mapfile -t pids < <(pgrep -f '需要精确限定的进程命令表达式' || true)
```

`<(...)` 将命令输出作为可读取的数据源，前面的 `<` 把它接给 `mapfile`；`mapfile -t` 按行读成数组并去掉换行。这里先容忍“未找到进程”，再通过数组长度判断有没有旧进程。

`pgrep -f` 按完整命令行匹配。`^`、`$` 约束开头和结尾，`([1-9]|1[0-2])` 可以匹配 1～12，不能用 `[1-12]` 表示数字范围 1～12。表达式过宽可能匹配到不相关进程。

`kill -CONT` 让暂停进程继续运行；`kill -TERM` 请求终止。先恢复暂停进程，可以使其有机会处理终止请求。`kill -0 PID` 不发送实际终止信号，只检查能否访问该进程；即使成功也不证明它正在正常推流。

## 5. ffprobe：先问清视频是什么

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=codec_name -of csv=p=0 sample.mp4
```

| 参数 | 含义 |
|---|---|
| `-v error` | 只显示错误日志 |
| `-select_streams v:0` | 选择第一条视频流 |
| `-show_entries stream=codec_name` | 只查询视频编码名 |
| `-of csv=p=0` | 使用简洁 CSV 输出，不加节名前缀 |

脚本用输出的 `h264` 或 `hevc` 选择推流方式。MP4 是容器，H.264/HEVC 是视频编码，文件扩展名不能替代编码探测。探测编码成功不等于整段视频无损坏，也不等于 NVENC 一定可用。

## 6. FFmpeg 两条推流路径

### 路径一：直接复制 H.264

```bash
ffmpeg -nostdin -hide_banner -loglevel error \
  -stream_loop -1 -re -i sample.mp4 \
  -map 0:v:0 -an -c:v copy -f flv \
  rtmp://127.0.0.1:1935/demo/live_1
```

| 参数 | 解释 |
|---|---|
| `-nostdin` | 不读取终端交互输入，适合这里的后台进程 |
| `-hide_banner` | 隐藏版本等启动介绍 |
| `-loglevel error` | 只输出错误，排查时可在测试环境临时提高日志级别 |
| `-stream_loop -1` | 无限循环输入；不保证循环交界完全无抖动 |
| `-re` | 按正常速度读取文件，模拟实时输入；不保证资源不足时仍实时 |
| `-i sample.mp4` | 输入文件 |
| `-map 0:v:0` | 第一个输入的第一条视频轨道 |
| `-an` | 不输出音频 |
| `-c:v copy` | 复制压缩视频包，不重新解码编码 |
| `-f flv` | 输出 FLV 封装 |
| `rtmp://...` | 输出地址，服务端口与流名称由地址指定 |

这条路径主要做视频包读取、封装和网络发送，不运行目标识别，也不主动改变图像大小或帧率。更换封装能否成功，还取决于时间戳和输入编码等条件。

### 路径二：GPU 解码后转成 H.264

```bash
ffmpeg -nostdin -hide_banner -loglevel error \
  -stream_loop -1 -re -hwaccel cuda -hwaccel_output_format cuda \
  -i sample-hevc.mp4 -map 0:v:0 -an \
  -c:v h264_nvenc -preset p4 -tune ll -g 50 -f flv \
  rtmp://127.0.0.1:1935/demo/live_2
```

| 新参数 | 解释 |
|---|---|
| `-hwaccel cuda` | 使用 NVIDIA GPU 硬件解码加速路径 |
| `-hwaccel_output_format cuda` | 解码结果以 CUDA 帧形式留在 GPU 中，供编码使用 |
| `-c:v h264_nvenc` | NVIDIA 硬件编码为 H.264 |
| `-preset p4` | NVENC 编码预设，影响速度与质量取舍 |
| `-tune ll` | 低延迟调优 |
| `-g 50` | 设置 GOP 最大长度为 50 帧；秒数还取决于输出帧率 |

只有 FFmpeg 编译支持相关功能、GPU 支持且驱动兼容时，这条命令才能成功。`-tune ll` 不能消除服务器、播放器缓冲等其他延迟。

输入选项应放在相应 `-i` 前，输出选项放在目标输出前。阅读命令时先找 `-i` 和输出 URL，再把参数分组，避免将解码与编码选项混淆。

## 7. 多路并发：&、$! 与 wait

```bash
publisher_pids=()
ffmpeg ... &
publisher_pids+=("$!")
wait
```

`&` 由 Bash 解释，使 FFmpeg 在后台执行，循环可以继续启动下一路。`$!` 是最近一个后台任务的 PID，必须及时保存。数组保存 PID 后，退出时可只清理自己启动的进程。

最后的 `wait` 让脚本等待后台任务；不带参数的 `wait` 不是“持续检查所有子进程是否健康”，也不能可靠汇总所有子进程的失败原因。脚本启动两秒后逐个 `kill -0` 的检查只是当时的一次进程存活检查，不能发现运行半小时后的断流。

## 8. trap：退出时怎样收尾

```bash
cleanup() {
  trap - EXIT INT TERM
  for pid in "${publisher_pids[@]}"; do
    kill -CONT "$pid" 2>/dev/null || true
    kill -TERM "$pid" 2>/dev/null || true
  done
  wait 2>/dev/null || true
}
trap cleanup EXIT INT TERM
```

`trap` 注册退出或收到信号时执行的动作。`INT` 通常对应 Ctrl+C，`TERM` 是终止请求，`EXIT` 对应 shell 退出。清理开始先移除这些 trap，避免清理流程重复触发；发送终止请求后用 `wait` 等待回收。

这里没有等待超时后再强杀的机制；若子进程一直不退出，清理可能等待。SIGKILL 不能被 trap 捕获，因此不能把 trap 理解为任何情况下都能保证清理。脚本的清理函数只管理已记录的推流进程，不会自动停止独立运行的 Compose 媒体服务。

## 9. 启动准备、归档与服务检查

- `mkdir -p`：创建目录，已有目录不因这一命令而被清空。
- `$(date +%Y%m%d-%H%M%S)`：形成便于辨识的时间戳目录名；只有秒精度，不等于绝对不会重名。
- `{cameras.json,selected.json,pipeline.yaml}`：Bash 大括号展开，生成多个路径。
- `[ ! -f "$file" ] || mv ...`：文件存在才移动；这里归档的是选流缓存，不是告警图片。
- `docker compose -f ... up -d`：按指定配置启动媒体容器，`-d` 使其后台运行。
- `for _ in {1..20}` 加 `sleep 0.5`：循环等待端口就绪，等待时间约十秒，另有命令本身的开销。

当前脚本用文本匹配检查 local_only，启动推流前还处理旧服务与缓存。因此学习时可以先读语法，再在独立示例中练习，直接执行整个业务脚本会有实际进程和文件操作。

## 10. 推荐学习资源与阅读顺序

| 顺序 | 官方资源 | 针对本脚本重点阅读 |
|---|---|---|
| 1 | [Bash 手册](https://www.gnu.org/s/bash/manual/bash.html) | 引号、展开、循环、退出状态 |
| 2 | [Bash 数组](https://www.gnu.org/software/bash/manual/html_node/Arrays.html) | 数组下标、长度、@ 展开 |
| 3 | [Bash 信号](https://www.gnu.org/s/bash/manual/html_node/Signals.html) | 异步命令、wait 和 trap 的关系 |
| 4 | [FFmpeg 文档](https://ffmpeg.org/ffmpeg.html) | 输入输出顺序、Streamcopy、-map、-stream_loop、-re |
| 5 | [FFprobe 文档](https://ffmpeg.org/ffprobe.html) | show_entries、select_streams、输出格式 |
| 6 | [NVIDIA：FFmpeg GPU 加速](https://docs.nvidia.com/video-technologies/video-codec-sdk/13.0/ffmpeg-with-nvidia-gpu/index.html) | CUDA 解码、GPU 帧、NVENC 转码示例 |
| 7 | [Docker Compose up](https://docs.docker.com/reference/cli/docker/compose/up/) | -f 配置选择及 -d 后台启动 |

NVIDIA 文档按 SDK 版本发布，示例还要结合本机 FFmpeg/驱动核对。本机帮助也很实用：`help trap`、`help wait`、`help mapfile`、`man ss`、`man pgrep`、`ffmpeg -h encoder=h264_nvenc`。建议先用 `bash -n your-script.sh` 做语法检查，它不会执行推流，但也不会证明运行逻辑正确。

## 11. 自测与迁移练习

先回答：

1. 为什么视频路径要加引号？`${videos[@]}` 与 `${!videos[@]}` 分别是什么？
2. 为什么 `-c:v copy` 一般比转码开销小？MP4、H.264、FLV、RTMP各处于哪一层？
3. `-g 50` 为什么不能直接说成“每两秒一个关键帧”？
4. `kill -0` 成功和1935监听，分别不能证明什么？
5. 为什么记录 `$!`，为什么 Ctrl+C 后还要 `wait`？

练习只在通用示例中进行：创建两个带空格的虚构文件路径并打印数组；用两个 `sleep` 后台任务练习 PID 数组和 trap；对自备短视频用 ffprobe 查询编码；先做单路复制推流，再比较一条 GPU 转码的资源开销。最后尝试设计“子进程退出时报告哪一路失败”的检查逻辑，解释它与启动时 `kill -0` 的区别。

学到的应是可复用的 Bash 和音视频知识，不需要背下原脚本的目录、视频名称或固定端口。
