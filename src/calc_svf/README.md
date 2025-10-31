# Calc_svf

简介：从百度街景抓取四个方向的街景图（0/90/180/270），对每张图片进行裁剪与天空分割，合成鱼眼图，最后计算天空视角因子（SVF）。
## 代码结构
- `fisheye_svfcalculator.py` - 主脚本，入口为 `main()`
- `baiduStreetViewSpider.py` - 包含读取 CSV、坐标转换、抓取图片（同步实现）、部分坐标工具函数
- `fisheye.py` - 鱼眼图合成函数 `generate_fisheye_image`
- `sky_segmentation.py` - 天空分割，提供 `extract_sky_mask_from_image(image)`
- `svf.py` - SVF 计算逻辑 `calculate_svf(fisheye_path, svf_rings)`（使用 OpenCV）


## 主要功能与流程
1. 读取 CSV 列表（默认 `example_coorddinates.csv`），提取经纬坐标。
2. 将 WGS84 转换为百度投影并获取街景 pano id（svid）。
3. 并发抓取四个方向图片（已改为 `aiohttp` 异步抓取，受 Semaphore 限制以控制并发）
4. 在内存中对每张图片进行左右裁剪，并通过 `transformers` 的语义分割模型提取天空掩码。
5. 使用四个方向的天空掩码图片合成鱼眼图（`generate_fisheye_image`，在线程池运行以避免阻塞事件循环）。
6. 将鱼眼图路径提交到进程池执行 `calculate_svf` 以计算 SVF。
7. 将结果写入 `coordinate_svf_results.csv`，同时鱼眼图保存在 `fisheye_output/`。

## 核心模块说明
- `fisheye_svfcalculator.py`
  - main_async: 异步主流程，使用 `aiohttp.ClientSession` 抓取图片。
  - process_coordinate: 负责单个坐标的抓取、裁剪、分割、合成与 SVF 计算。
  - fetch_image: aiohttp 异步获取图片 bytes。
  - discard_image_regions_pil: 在内存中裁剪图片。

- `fisheye.py`
  - generate_fisheye_image(image_paths, output_path, temp_dir=None):
    - 支持 `image_paths` 中的值为 file path、bytes 或 PIL.Image。
    - 在内存中拼接为全景并用 numpy/cv2 转换为鱼眼。

- `sky_segmentation.py`
  - extract_sky_mask_from_image(image): 接受 PIL.Image 或 bytes，返回 PIL.Image 的 sky mask。

- `svf.py`
  - calculate_svf(fisheye_path, svf_rings): 从鱼眼图读入（BGR），阈值分割天空（简单灰度阈值），按同心环统计天空占比并累加得到 SVF 值。

- `baiduStreetViewSpider.py`
  - read_csv / write_csv: CSV I/O
  - getPanoId: 现为同步 requests 实现，返回 pano id（svid）
  - grab_img_baidu: 原为同步 requests 获取图像（本次改造里我们使用 aiohttp 替代该功能）
  - wgs2bd09mc: 坐标投影转换工具


## 运行
在激活的环境中运行：

```cmd
python fisheye_svfcalculator.py
```

默认读取 `example_coordinates.csv`，并将鱼眼图存入 `fisheye_output/`，输出 CSV 为 `coordinate_svf_results.csv`。

可选参数（可在脚本中修改）：
- `max_conn`：aiohttp 并发上限（Semaphore 控制）
- 线程池/进程池大小：脚本基于本机 CPU 自动设置，必要时可在代码中硬编码调整

