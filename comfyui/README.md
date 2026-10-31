# RuiC 本地换装工作流（ComfyUI 模板）

把 **RuiC 的姿势 + 服装 + 人像** 一次性送进 ComfyUI，一步生成「同姿态、同脸、穿指定衣服」的角色图。

> ⚠️ 你的显卡是 **RTX 4060 8GB**。本模板默认走「SD1.5 底座 + ControlNet(OpenPose) 控姿态 + IP-Adapter 保脸」，**服装用提示词描述**（最稳、8GB 能跑）。需要「精确把 RuiC 那件衣服原样穿上」则加一个轻量试穿节点（CatVTON，见文末），但显存会更紧。

---

## 1. 工作流在做什么

```
[姿势图 RuiC] ──DWPreprocessor──▶ ControlNetApply ──┐
[人像图 RuiC] ──IP-Adapter────────▶ 改 MODEL        │
[Checkpoint] ──► MODEL ──────────► IP-Adapter ───────┤
[正/负提示词] ──► CLIPTextEncode ─► ControlNetApply ─┤
                                                     ▼
                              KSampler → VAE Decode → SaveImage
```

- **姿态**：RuiC 姿势图 → OpenPose 骨架 → ControlNet 约束生成姿态（可按角度选正面/侧面等）。
- **脸**：RuiC 人像图 → IP-Adapter 锁定脸型/发型，出图保持同一张脸。
- **服装**：在正向提示词里写 RuiC 服装描述（studio.html 已自动生成）；或加试穿节点精确穿原图（见第 6 节）。

---

## 2. 安装 ComfyUI + 必需节点

```bash
# 1) 装 ComfyUI（已有可跳过）
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
python -m venv venv && venv\Scripts\activate
pip install -r requirements.txt

# 2) 启动后在浏览器打开 http://127.0.0.1:8188
#    菜单里点「Manager」→「Install Custom Nodes」搜索安装下面三个：
#      - ComfyUI-ControlNet-Aux      （提供 DWPreprocessor / OpenPose 预处理）
#      - ComfyUI-IPAdapter-Plus      （提供 IPAdapter / IPAdapterModelLoader）
#      - ComfyUI-CatVTON             （可选，第 6 节精确试穿用）
```

> 启动显存不够时加参数：`python main.py --medvram`（或 `--lowvram`），并建议加 `--force-fp16`。

---

## 3. 下载模型权重（放进对应子目录）

网络提示：你这台机器走代理、大文件易断流，建议用镜像 `hf-mirror.com`（把 `huggingface.co` 换成 `hf-mirror.com` 或用 `HF_ENDPOINT=https://hf-mirror.com`）。

| 文件 | 放到 | 说明 |
|------|------|------|
| `v1-5-pruned-emaonly.ckpt` | `ComfyUI/models/checkpoints/` | SD1.5 底座（8GB 首选，比 SDXL 省显存） |
| `control_v11p_sd15_openpose.pth` | `ComfyUI/models/controlnet/` | OpenPose 控制网 |
| `clip_vision_g.safetensors` | `ComfyUI/models/clip_vision/` | IP-Adapter 需要的 CLIP Vision |
| `ip-adapter_sd15_vit.pt` + `ip-adapter-plus_sd15.bin` | `ComfyUI/models/ipadapter/` | SD1.5 版 IP-Adapter（若用 SDXL 则换对应 sdxl 版） |
| `vae-ft-mse-840000-ema-pruned.ckpt` | `ComfyUI/models/vae/` | SD1.5 的 VAE |

> 想用 SDXL 底座（脸/质感更好但更吃显存）：把 Checkpoint 换成对应的 SDXL 模型即可（本机已用 `WAI-Nsfw-Illustrious-17.safetensors`），IP-Adapter 换成 `ip-adapter-plus-face_sdxl_vit-h.safetensors` + `CLIP-ViT-H-14-laion2B-s32B-b79K.safetensors` 图像编码器，ControlNet 换成 `controlnet-union-sdxl-1.0.safetensors`（xinsir union 版，openpose/canny/depth 通用）。8GB 下 SDXL 建议 `--medvram` 且分辨率别超 1024×1024。

---

## 4. 载入模板并接线

1. ComfyUI 菜单 → **Load** → 选本文件同目录的 `studio_workflow.json`。
2. 模板里的文件名是占位符，按你实际下载的改：
   - `CheckpointLoaderSimple` → 选你的 `v1-5-pruned-emaonly.ckpt`
   - `ControlNetLoader` → 选 `control_v11p_sd15_openpose.pth`
   - `CLIPVisionLoader` → 选 `clip_vision_g.safetensors`
   - `IPAdapterModelLoader` → 选 `ip-adapter-plus_sd15.bin`
   - `VAELoader` → 选 `vae-ft-mse-840000-ema-pruned.ckpt`
3. 把模型名对不上的红色节点重新连一下插座（不同节点包版本输入名可能略有差异，拖线即可）。

---

## 5. 喂入 RuiC 的图（姿势 + 人像）

在 `studio.html` 选好三张后点 **⬇️ 下载三张参考图**，会得到：
- `pose_*.jpg`（你选的角度，如 `img/full_sexy_P001_front.jpg`）
- `face_*.png`（如 `portrait/img/f71f980dde3.png`）
- `costume_*.webp`

把 **pose 和 face 两张** 复制到 `ComfyUI/input/` 目录，然后在模板的两个 `LoadImage` 节点里分别选它们：
- 节点 1「Load Pose」→ 选 pose 图
- 节点 10「Load Face」→ 选 face 图

**服装**：打开 `studio.html` 点 **📋 复制合成提示词**，把里面的服装描述粘进节点 6「Positive」的提示词（替换 `<RuiC costume>` 那段）。

> 分辨率：节点 12「Empty Latent」默认 `768×1024`（竖图，适合全身）。8GB 卡别超过这个值；若 OOM 降到 `640×960`。

---

## 6.（可选）精确穿上 RuiC 原图服装 —— 加 CatVTON 节点

提示词描述服装只是"近似"。要**把 RuiC 那件衣服原样穿上去**，在上面的出图后再接一个试穿：

1. 安装 `ComfyUI-CatVTON`（第 2 节）。
2. 下载 CatVTON 权重到 `ComfyUI/models/catvton/`（按该节点仓库说明）。
3. 在画布上加 `CatVTON` 节点：
   - `person_image` ← 第 15 节点（VAE Decode 的输出，即生成的角色）
   - `garment_image` ← 新增一个 `LoadImage`，选 RuiC 的 `costume_*.webp`
   - 输出接 `SaveImage`。
4. CatVTON 在 8GB 下把分辨率压到 `768×576` 左右再跑，关掉其他程序。

> 不想本地压显存？这一步也可改用线上：把生成的角色 + RuiC 服装图丢进 `huggingface.co/spaces/yisol/IDM-VTON`（免登录，但非商用）。即「本地出角色 → 线上试穿」，绕开 8GB 限制。

---

## 7. 一键到底的两种路线总结

| 路线 | 姿态 | 脸 | 服装 | 显存 | 商用 |
|------|------|----|------|------|------|
| **A（本模板默认）** 本地 ComfyUI | ControlNet | IP-Adapter | 提示词描述 | 8GB 可 | 看底座授权 |
| **B** 本地出角色 + 线上 IDM-VTON 试穿 | ControlNet | IP-Adapter | 精确试穿 | 出角色 8GB | 非商用 |
| **C** 全本地精确试穿 | ControlNet | IP-Adapter | CatVTON | 紧（8GB 压分辨率） | 看授权 |

建议先用 **A** 跑通，确认姿态/脸满意，再按需接 **B/C** 做精确服装。

---

## 8. SDXL 版模板（`studio_workflow_sdxl.json`）

同目录另有 `studio_workflow_sdxl.json`：**接线逻辑与 SD1.5 版完全一致**，只是换成 SDXL 底座，脸和整体质感更好、细节更稳，但更吃显存。

### 与 SD1.5 版的差异

| 项 | SD1.5 版 `studio_workflow.json` | SDXL 版 `studio_workflow_sdxl.json` |
|----|------|------|
| Checkpoint | `v1-5-pruned-emaonly.ckpt` | 本机用 `WAI-Nsfw-Illustrious-17.safetensors`（SDXL 架构动漫底模） |
| 文本编码节点 | `CLIPTextEncode`（单串） | `CLIPTextEncodeSDXL`（clip_g + clip_l 双串 + 尺寸条件） |
| ControlNet | `control_v11p_sd15_openpose.pth` | `controlnet-union-sdxl-1.0.safetensors`（xinsir union，openpose/canny/depth 通用） |
| IP-Adapter | `ip-adapter-plus_sd15.bin` | `ip-adapter-plus-face_sdxl_vit-h.safetensors`（脸部专用版） |
| 图像编码器 | `clip_vision_g.safetensors` | `CLIP-ViT-H-14-laion2B-s32B-b79K.safetensors` |
| VAE | `vae-ft-mse-840000-ema-pruned.ckpt` | 用底模自带 VAE（模板已直连 Checkpoint 输出，无需单独下载） |
| KSampler 默认 | steps 25 / cfg 7.0 / euler | steps 30 / cfg 6.5 / dpmpp_2m / karras |
| 显存占用 | 低（8GB 首选） | 高（8GB 偏紧） |

### 载入 SDXL 版时注意
1. 菜单 → **Load** → 选 `studio_workflow_sdxl.json`。
2. 第 6 / 7 节点的 `CLIPTextEncodeSDXL` 有两个文本框：`text_g`（全局/风格，如 `masterpiece, best quality...`）和 `text_l`（细节/主体，把 studio.html 复制的服装描述粘进这里，替换 `<RuiC costume>`）。下方 4 个数字是出图宽高 + 裁剪，保持与节点 12「Empty Latent」一致（`768×1024`）。
3. ControlNet / IP-Adapter 节点必须换成 **SDXL 对应权重**，否则会红节点报错（SD1.5 权重喂不进 SDXL 模型）。

### 8GB 显存下的取舍
- SDXL 在 8GB 上**能跑但要精打细算**：启动务必加 `--medvram --force-fp16`；
- 节点 12「Empty Latent」默认 `768×1024`，8GB 不建议超过；OOM 就降到 `640×960` 或 `512×768`；
- 若经常 OOM，退回 SD1.5 版更稳（脸略逊但省心）。

> 决策建议：追求"脸一模一样 + 衣服描述到位 + 不折腾显存"→ SD1.5 版；追求"更高清、质感更好、能接受偶尔 OOM 调参数"→ SDXL 版。两者服装都用提示词描述；要精确穿原图都走第 6 节 CatVTON 或线上 IDM-VTON。
