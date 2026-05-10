# YOLOE-26 文本提示最小测试项目

[YOLOE](https://docs.ultralytics.com/models/yoloe/) 是开放词汇检测/分割模型，支持：

- **文本提示**：`set_classes(["person", "bus"])` 后按描述零样本检测  
- **视觉提示**：用参考图框选目标（需 `YOLOEVPSegPredictor`）  
- **Prompt-Free**：`yoloe-26n-seg-pf.pt` 等内置大类表，无需 `set_classes`

本仓库只做最简单的 **文本提示 + YOLOE-26** 推理。

## 环境

```bash
cd yolo26_text_project
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
# 文本提示必须再装 CLIP（见下节）
```

### 安装 CLIP（`git clone` 报 curl 28 / Connection reset 时）

报错原因多为 **访问 GitHub 不稳定**（连接被重置），不是 pip 坏了。按顺序试：

1. **推荐：ZIP 安装（不经过 git clone）**  
   ```powershell
   pip install https://github.com/ultralytics/CLIP/archive/refs/heads/main.zip
   ```
   或在本目录执行：  
   `powershell -ExecutionPolicy Bypass -File .\install_clip.ps1`

2. **仍失败**：浏览器下载 [CLIP-main.zip](https://github.com/ultralytics/CLIP/archive/refs/heads/main.zip)，解压后：  
   `pip install C:\你解压的路径\CLIP-main`

3. **有代理时**：先让终端走代理，再执行上面任一步；或换网络/热点多试几次。

4. **仍用 git 时可加大缓冲再装**（有时能缓解中断）：  
   `git config --global http.postBuffer 524288000`  
   `pip install git+https://github.com/ultralytics/CLIP.git`

**PowerShell 提示「禁止运行脚本」无法 `activate`？** 这是执行策略拦了 `Activate.ps1`。任选其一：

1. 当前窗口临时放行：  
   `Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process`  
   再执行 `.\.venv\Scripts\activate`
2. 用 CMD：`cmd` 里运行 `.venv\Scripts\activate.bat`
3. 不激活也行：  
   `.\.venv\Scripts\python.exe -m pip install -r requirements.txt`

需要 **Python 3.8+**；CLIP 可用 ZIP 安装，**不强制 Git**。有 **NVIDIA GPU** 会快很多（CPU 也可跑，较慢）。

**首次运行**会：下载 `yoloe-26n-seg.pt`（约 11MB）、安装 CLIP、再下载文本侧权重 `mobileclip2_b.ts`（约 242MB）。若终端提示 *Restart runtime*，再执行一次同一条命令即可。

## 运行

```bash
# 默认：用 Ultralytics 自带 bus.jpg，类别 person + bus
python test_yoloe26.py

# 指定自己的图和类别（建议英文短语，与 CLIP 对齐）
python test_yoloe26.py D:\pics\street.jpg --classes "car,person,traffic light"

# 更大模型（更准、更慢）
python test_yoloe26.py photo.jpg --model yoloe-26s-seg.pt

#水花识别示例
python test_yoloe26.py test_picture\photooriginal-2267906-hzYDw.jpg --model yoloe-26m-seg.pt --classes "splashing water,white foam,water spray" --conf 0.1
```

结果保存在目录 `output/picture_run/`。

### 大华摄像头人物检测（实时流）

用 **`run_camera.py`** 接大华（或任意 RTSP/ONVIF）摄像头做**实时人物检测**，可行且用法简单。

1. **拿到摄像头 RTSP 地址**  
   - 大华常见格式：  
     `rtsp://用户名:密码@摄像头IP:554/cam/realmonitor?channel=1&subtype=0`  
   - `subtype=0` 主码流（清晰、占带宽），`subtype=1` 子码流（省带宽、适合推理）。  
   - 在摄像头 Web 配置页或 NVR 里查看「RTSP」/「主码流」即可得到完整 URL。

2. **运行（人物检测，默认 person）**  
   ```bash
   # 大华 RTSP（替换为你的地址和账号密码）
   python run_camera.py "rtsp://admin:你的密码@192.168.1.100:554/cam/realmonitor?channel=1&subtype=1"

   # 本机摄像头
   python run_camera.py 0
   ```
   - 弹窗会实时显示检测框，按 **Q** 退出。  
   - 若只保存不弹窗（无显示器/远程）：  
     `python run_camera.py "rtsp://..." --save --no-show`

3. **可选参数**  
   - `--conf 0.25`：降低阈值，人多漏检时可试。  
   - `--imgsz 480`：降低分辨率，提高 RTSP 流畅度。  
   - `--model yoloe-26s-seg.pt`：换更大 YOLOE 模型提高准确率（更吃算力）。  
   - `--classes "person,car"`：多类一起检。  

   **权重文件名不要写错（否则会 `FileNotFoundError`）**  
   - `run_camera.py` 用的是 **YOLOE（开放词汇）**，权重名里是 **`yoloe`**（中间有字母 **e**），例如：**`yoloe-26s-seg.pt`**、**`yoloe-26m-seg.pt`**。  
   - **不要**写成 `yolo-26s-seg.pt` 或 `yolo26s-seg.pt`：前者是笔误；后者一般是 **普通 YOLO26 封闭类别分割**（`YOLO()`），**不能**替代 YOLOE 的 `set_classes` 文本提示流程。

4. **注意**  
   - 运行机与摄像头需**网络互通**（同一局域网或端口已映射）。  
   - 首次拉流可能较慢，属正常；若一直卡住，检查 RTSP 地址、用户名密码、防火墙。  
   - 人物检测用默认 `yoloe-26n-seg.pt` 即可；需要更高精度再换 `yoloe-26s-seg.pt` 等。

5. **乐橙云 / 大华云 RTSP 报错或黑屏**  
   - **`DESCRIBE failed: 404 Not Found`**：链接里的 **`expire` 已过期** 或 digest 失效，必须在开发者平台**重新生成** RTSP 再粘贴（不要用过期地址）。  
   - **长时间无画面、最后 `KeyboardInterrupt`**：多为拉流卡住；`run_camera.py` 已默认 **RTSP 走 TCP** 并在加载模型前**探测首帧**（约 20 秒内给出明确失败原因）。  
   - 先用 **VLC**（媒体 → 打开网络串流）或 `ffplay "rtsp://..."` 能播，再跑 Python。  
   - 若探测误报可临时加 `--no-probe`（一般不建议）。

6. **HTTPS：FLV 与 HLS（自动识别）**  
   `run_camera.py` / `run_camera_pose.py` 对 **http(s)://** 地址会根据 URL **自动判断**常见形态并设置 FFmpeg：  
   - **HLS**：路径中含 **`.m3u8`** 或 **`/hls/`**（如 `https://xxx.com/live/stream.m3u8`）  
   - **FLV**：路径含 **`.flv`** 或 **`/flv/`**（大华乐橙常见）  
   - **其它 HTTPS 视频**：按通用 HTTP 超时处理  

   大华云 **FLV** 示例（**整段 URL 用英文双引号**，避免 `&` 被 PowerShell 拆开）：  
   ```powershell
   python run_camera.py "https://cmgw-vpc.lechange.com:8890/flv/LCO/你的设备路径/xxx.flv?proto=https&source=open"
   ```  
   **HLS** 示例：  
   ```powershell
   python run_camera.py "https://你的域名/path/playlist.m3u8"
   ```  
   探测前会打印一行 **`流类型（自动）: HLS (.m3u8)`** 或 **`HTTP(S) FLV`** 等。若实际是 HLS 但 URL 里既没有 `.m3u8` 也没有 `/hls/`，会被当成通用 HTTPS，可能仍能播；若不能，请向平台索取带 **`.m3u8`** 的地址。  
   **链接常有时效**，失效后请在控制台重新获取。

7. **卡顿、风扇狂转：是 OpenCV 吗？要单独装 FFmpeg 吗？**  
   - **读流**：Ultralytics 底层用 **OpenCV** 的 `VideoCapture`；你装的 `opencv-python` / `opencv-python-headless` **自带 FFmpeg 解码**（编进 wheel 里），一般**不必**再单独装系统级 `ffmpeg.exe`，装了也不会自动替代 OpenCV 里的解码。  
   - **真正吃性能的是推理**：每一帧（或每隔几帧）跑 **YOLOE + PyTorch**，CPU/GPU 会拉满，这是正常现象。云 FLV 还有网络与解码开销。  
   - **减轻负担**：`--imgsz 480` 或 `320`；`--vid-stride 2`（隔帧推理）。**不要对 YOLOE 分割权重加 `half=True`**：当前 Ultralytics 在掩码后处理里混用 FP16/FP32 会报错（`Half != float`）；提速请靠分辨率与跳帧。安装 **CUDA 版 PyTorch** 后推理默认会用 GPU（任务管理器里 GPU 有占用）。

### 人体姿态（与 YOLOE 分开）

**YOLOE 只做开放词汇检测/分割，不做骨架。** 姿态请用 Ultralytics 自带的 **YOLO Pose**（COCO 17 关键点：鼻、肩、肘、膝等）。

**`run_camera_pose.py` 默认开启 ByteTrack 追踪**：**每帧**做姿态推理，并用 **`persist=True`** 维护人体 ID，框/骨架会**连续贴在人身上**，减轻「隔几帧才画一次」的闪烁。更吃算力时可 **`--imgsz 480`**。

```powershell
# 默认：追踪 + 每帧推理（推荐，画面连贯）
python run_camera_pose.py "https://....flv?proto=https&source=open" --imgsz 480（可选，480p）

# 换 BoT-SORT（可选）
python run_camera_pose.py 0 --tracker botsort.yaml
```

**省算力模式**（关闭追踪 + 降频 + 中间帧沿用上一帧骨架画在当前画面上，减少闪、但快速移动时会有轻微滞后）：

```powershell
python run_camera_pose.py "https://....flv?..." --no-track --pose-every 8 --imgsz 480
# 完全不要中间帧叠画（与旧版一致）：再加 --no-hold-overlay
```

- 模型默认 **`yolo26n-pose.pt`**（轻量）；可换 **`yolo26s-pose.pt`** 等（更准更慢）。若下载失败可试 **`yolo11n-pose.pt`**。（更大模型：--model yolo26s-pose.pt）  
- **不需要 CLIP**。  
- 若需「检测框 + 姿态」同时跑，可开两个终端分别跑 `run_camera.py` 与 `run_camera_pose.py`。

参数	                            含义
（默认）	                 追踪开，每帧推理，最连贯
--no-track	             关追踪，配合 --pose-every
--pose-every N           仅在 --no-track 时生效
--hold-overlay	          默认开；无追踪时中间帧叠上一帧骨架
--no-hold-overlay	       中间帧不叠画
--tracker	             bytetrack.yaml（默认）或 botsort.yaml
--model yolo26s-pose.pt  更大模型，效果更好
## 模型权重（自动下载）

| 模型 | 权重文件名 |
|------|------------|
| 最小/fast | `yoloe-26n-seg.pt` |
| 平衡 | `yoloe-26s-seg.pt` / `yoloe-26m-seg.pt` |
| 高精度 | `yoloe-26l-seg.pt` / `yoloe-26x-seg.pt` |

无提示内置词表版：`yoloe-26n-seg-pf.pt` 等（用法见 [官方文档](https://docs.ultralytics.com/models/yoloe/)）。

### 为什么「水花」很难零样本检出来？

YOLOE 的文本分支是在 **LVIS / COCO 这类「可数物体 + 框标注」** 上对齐的，常见类别是 person、car、cup 等 **边界相对清楚的目标**。而增氧机 **水花** 往往是：

| 因素 | 说明 |
|------|------|
| **几乎不在训练标注里** | 很少有数据集把「一片溅起的水」标成独立实例框，模型没学过把这类区域当成高置信度「物体」。 |
| **形态不固定** | 水花是瞬态、破碎、连成一片，和「车」这种刚性物体差别大，**区域-文本对齐**容易偏弱。 |
| **和背景都是水** | 视觉上仍是水，只是更白、更碎，CLIP 不一定把它拆成你要的那一类。 |

因此 **换更大模型（如 `yoloe-26m-seg.pt` / `yoloe-26l-seg.pt`）可能略有帮助，但通常不能从根上解决**——瓶颈更像是 **任务类型**（非典型「物体」），而不只是容量。

**可先做的尝试（成本低）：**

1. **多写几个英文提示一起搜**（同一轮 `set_classes` 里多类，总有一类可能对上）：  
   `--classes "splashing water,white foam,water spray,churning water"`
2. **放低置信度**（看是否其实有极弱框）：  
   `--conf 0.1` 或 `0.05`
3. **稍大模型 + 上面两条**：  
   `--model yoloe-26m-seg.pt`

若仍不稳定，更靠谱的方向是：**在该场景上微调**、或 **分割/传统 CV**（例如在水面 ROI 内按亮度、纹理找高亮白浪区域），而不是单靠开放词汇检测。

## 参考

- [YOLOE 文档](https://docs.ultralytics.com/models/yoloe/)
- [YOLO26 / YOLOE-26](https://docs.ultralytics.com/models/yolo26/)

## GitHub 版本管理与团队协作（推荐）

下面是一套适合本项目的最小协作流程，建议所有成员统一执行。

### 1) 首次提交到 GitHub

```powershell
# 如果当前分支还是 master，建议改为 main
git branch -M main

# 绑定你在 GitHub 上新建的仓库（替换 URL）
git remote add origin https://github.com/<你的账号或组织>/yolo26_text_project.git

# 首次推送
git push -u origin main
```

### 2) 邀请朋友加入协作

- 个人仓库：`Settings -> Collaborators -> Add people`
- 组织仓库：在 Organization 里创建 Team，再把仓库权限给 Team（建议 `Write`）

### 3) 日常开发流程（不要直接改 main）

```powershell
# 同步主分支
git checkout main
git pull

# 新功能开新分支
git checkout -b feat/<功能名>

# 开发完成后提交
git add .
git commit -m "feat: <一句话说明改动目的>"
git push -u origin feat/<功能名>

# 在副分支上提交新改动
1) 先确认当前分支
git branch --show-current
应显示：feature/splash-pipeline

2) 看改动
git status
3) 只加入你要同步的文件（建议先代码和文档）
例：git add run_splash_pipeline.py README.md
如果你不想把测试视频/大文件传上去，就不要 git add .。

4) 提交
git commit -m "feat: improve splash pipeline with irregular-motion gating"
5) 推送到同一个远程分支
git push
```

然后在 GitHub 发 Pull Request，互相 Review 后再合并到 `main`。

### 4) 提交前自检清单

- 代码可运行：`python test_yoloe26.py`
- 若涉及摄像头：至少本地跑通一次 `run_camera.py` 或 `run_camera_pose.py`
- 确认未误提交大文件（模型、数据集、输出视频等，已由 `.gitignore` 过滤）
- 提交信息尽量说明“为什么改”，不只写“改了什么”

## 水花实时化参数（新）

`run_splash_pipeline.py` 已加入“实时优先”改造，重点解决：
- 高 `imgsz` 导致速度慢
- 输出视频时长异常（尤其流媒体）
- 掩码/框转瞬即逝

新增关键参数：
- `--source-mode {auto,file,stream}`：区分离线视频与实时流处理策略
- `--save-fps`：`stream` 模式输出帧率，建议 `10~15`
- `--infer-interval-ms`：按时间触发推理（毫秒），替代只按帧计数
- `--hold-frames`：短时缺检时保持目标显示，减少闪烁
- `--track-iou-thresh`：目标跟随匹配阈值
- `--preset {realtime,balanced,quality}`：预设参数组合，可手动覆盖

推荐命令：

```powershell
# 实时流（优先流畅）
python run_splash_pipeline.py "https://...m3u8" --save --preset realtime --source-mode stream

# 本地视频（优先质量）
python run_splash_pipeline.py "D:\video.mp4" --save --preset quality --source-mode file --infer-interval-ms 0 --infer-every 1
```

## CPU 开发板人脸识别（白名单 + 陌生人 + 进出记录）

新增脚本：
- `build_face_gallery.py`：从图片构建白名单人脸库
- `run_face_camera.py`：实时人脸识别 + 进出事件入库

### 1) 安装依赖

```powershell
pip install -r requirements.txt
```

首次运行会自动下载 insightface 的 `buffalo_l` 模型包（内含 SCRFD + ArcFace）。

### 2) 构建白名单人脸库

目录结构示例（每个子目录是一个人名）：

```text
faces_input/
  Alice/
    1.jpg
    2.jpg
  Bob/
    1.jpg
    2.jpg
```

构建命令：

```powershell
python build_face_gallery.py faces_input --gallery-dir face_gallery
```

输出：
- `face_gallery/meta.json`
- `face_gallery/embeddings.npz`

### 3) 实时识别与进出事件

```powershell
# 本机摄像头
python run_face_camera.py 0 --gallery-dir face_gallery

# RTSP / HTTPS 流
python run_face_camera.py "rtsp://admin:密码@192.168.1.100:554/cam/realmonitor?channel=1&subtype=1" --gallery-dir face_gallery
```

可选：
- `--line "x1,y1,x2,y2"`：设置进出线（默认画面中线）
- `--detect-every 2`：每 2 帧检测一次（CPU 提速）
- `--det-size 640`：检测输入尺寸
- `--accept-thresh 0.46 --reject-thresh 0.34`：已知/陌生双阈值
- `--save --no-show`：后台运行并保存结果

输出目录：
- `output/face_run/face_events.db`（SQLite 事件日志）
- `output/face_run/snapshots/`（进出触发抓拍）
- `output/face_run/result.mp4`（可选，`--save`）

### 4) 事件规则

- 轨迹中心跨越进出线触发事件：`IN` / `OUT`
- 识别结果使用轨迹投票稳定（避免抖动）
- 同名同类型事件有冷却时间（默认 15 秒）
- 人脸库中找不到的目标记为 `UNKNOWN`

### 5) CPU-only 调优建议

- 先用 `--det-size 640 --detect-every 2`
- 若仍卡顿，试 `--det-size 480 --detect-every 3`
- 摄像头建议子码流（720p 或更低）做推理
- 入口场景固定后，建议设置 `--line` 与门口 ROI（后续可继续加）

### 6) 验收指标（默认目标）

定义在 `face_defaults.py`：
- 实时性：`FPS >= 10`
- 已知人识别：`Top1 >= 96%`
- 陌生人误报率：`<= 3%`
- 事件延迟：`<= 2s`

## 水花检测（CPU/ONNX/RKNN）

### 1) YAML 配置驱动（推荐）

现在支持 `--config`，推荐用短命令启动，避免超长 CLI：

```powershell
python run_splash_pipeline.py --config configs/splash/cpu_onnx_night.yaml
```

参数优先级固定为：

- `CLI 显式参数 > YAML > preset > argparse 默认值`

`source` 改为可选位置参数：

- CLI 给了 `source` 就用 CLI。
- CLI 没给就从 YAML 读 `source`。
- 两边都没有会直接报错。

### 2) 正负样本融合（主链路已接入）

CPU ROI 主链路已启用“规则 + 样本”融合：

- `score += pos_sim_weight * pos_sim`
- `score -= neg_sim_weight * neg_sim`
- 若 `neg_sim >= negative_sim_thresh`，候选会被硬抑制。

可选诊断：

```powershell
python run_splash_pipeline.py --config configs/splash/cpu_onnx_night.yaml --sample-report --sample-report-out output/splash_run/sample_report.json
```

### 3) ONNX 与 RKNN 后端

- ONNX 后端支持 `.onnx` 直接推理；
- 当 `--inference-backend onnx` 且模型是 `.pt` 时，可配合 `--export-onnx` 自动导出并缓存 ONNX。
- RK3568 生产路径建议使用 RKNN（NPU），不要把 ONNX-CPU 当最终部署方案。

RKNN 转换（开发机 x86，安装 RKNN-Toolkit2）：

```powershell
python tools/export_rknn.py --onnx yoloe-26s-seg.onnx --output yoloe-26s-seg.rknn --target rk3568 --quantize --dataset tools/rknn_calib.txt --input-size 640,640
```

### 4) RK3568 部署建议

- 板端用 `aarch64` Python，不能直接拷 x86 虚拟环境。
- OpenCV 优先系统包：`sudo apt install python3-opencv`
- ONNXRuntime 在板端仅作为回退/对照，生产走 RKNN。
- systemd 服务模板：`deploy/rk/splash-detector.service`
- 健康监控脚本：`deploy/rk/health_monitor.py`


### 水花识别更新代码保存指令

$ts = Get-Date -Format "yyyyMMdd_HHmmss"
>> $zip = "yolo26_text_project_source_$ts.zip"
>> Compress-Archive -Path `
>> README.md,RUN_GUIDE.md,requirements.txt,install_clip.ps1,.gitignore,`                                       
>> test_yoloe26.py,run_camera.py,run_camera_pose.py,run_splash_pipeline.py,stream_utils.py `
>> -DestinationPath $zip -Force
>> Write-Host "Created: $zip"
