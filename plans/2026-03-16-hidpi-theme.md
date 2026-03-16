---
name: HiDPI Theme Support
type: feature
---

## 概述

fcitx5 ClassicUI 的 9-patch 主题渲染在 HiDPI 下存在质量问题：

1. `Theme::paint()` 在 `scale != 1.0` 时创建**逻辑分辨率**的临时 surface，9-patch 渲染完成后再被缩放上下文放大，导致模糊
2. 主题作者无法提供高清资源——即使提供更大的图片，margin 值同时控制源图裁切位置和逻辑输出尺寸，角落永远只取 margin 个像素

本次改动：
- 移除临时 surface，9-patch 直接在缩放后的 cairo context 上绘制
- 支持 `@2x`/`@3x` 高清资源自动探测（如 `panel.png` → 探测 `panel@2x.png`、`panel@3x.png`）
- Margin 值始终表示 1x 逻辑像素，高清资源的源图裁切自动按倍率换算

## 设计

### 核心思路：cairo_surface_set_device_scale

加载 @2x 图片后，调用 `cairo_surface_set_device_scale(surface, 2, 2)`。这告诉 Cairo 该 surface 的像素密度是 2x，效果是：

- `cairo_set_source_surface(cr, image, sx, sy)` 中的偏移量仍以逻辑单位计算
- Cairo 自动将逻辑坐标乘以 device_scale 来采样源像素
- 例：margin=20，device_scale=2 → 角落采样 40x40 源像素，渲染到 20x20 逻辑区域

`paintTile` 的改动极小——只需用逻辑尺寸（pixel_dims / device_scale）计算 resizeWidth/resizeHeight，其余所有 `cairo_set_source_surface`、`cairo_rectangle`、`cairo_translate` 调用不变。

### 文件名约定

```
panel.png      → 1x（基准）
panel@2x.png   → 2x 高清
panel@3x.png   → 3x 高清
```

配置文件中只写 `Image=panel.png`，代码自动探测同目录下的高清变体。

### 变体选择策略

始终加载最高可用变体（@3x > @2x > 1x）。主题图片尺寸极小（48x48 → 144x144 约 83KB ARGB），内存影响可忽略。Cairo 在低 scale 显示器上会自动降采样，质量无损。

### 改动范围限定

本次只改 **9-patch 背景图片**（Background/Highlight），不改 Action 图片（翻页按钮）和 Overlay 图片。

### 关键数据结构

**ThemeImage** — 无新字段。device_scale 存储在 `cairo_surface_t` 自身，通过 `cairo_surface_get_device_scale()` 读取。

**paintTile 签名** — 不变：

```cpp
void paintTile(cairo_t *c, int width, int height, double alpha,
               cairo_surface_t *image, int marginLeft, int marginTop,
               int marginRight, int marginBottom);
```

paintTile 内部通过 `cairo_surface_get_device_scale(image, &dx, &dy)` 获取倍率，用 `pixel_width / dx` 计算逻辑尺寸。

### 关键函数

**`resolveHiDPIImage`**（新增，匿名命名空间内）：

```cpp
// 给定基准路径 "panel.png"，依次尝试 @3x、@2x、原始文件。
// 返回打开的 fd 和实际 path，以及对应的 scale (3, 2, 1)。
std::tuple<UnixFD, std::filesystem::path, int>
resolveHiDPIImage(const std::string &baseName,
                  const std::string &themeName,
                  bool isSystemTheme);
```

路径构造逻辑：

```cpp
// "panel.png" → stem="panel", ext=".png"
// scale=2 → "panel@2x.png"
std::filesystem::path hiDPIPath(const std::filesystem::path &base, int scale) {
    return base.parent_path() /
           (base.stem().string() + "@" + std::to_string(scale) + "x"
            + base.extension().string());
}
```

## 改动范围

- `src/ui/classic/`
  - `theme.cpp` - 修改
  - `theme.h` - 不变

## Tasks

- src/ui/classic/theme.cpp
  - [x] 新增 `hiDPIPath()` 路径构造辅助函数
  - [x] 新增 `resolveHiDPIImage()` 高清资源探测函数
  - [x] 修改 `ThemeImage::ThemeImage(BackgroundImageConfig)` 构造函数：调用 `resolveHiDPIImage` 加载最高可用变体，对加载到的 surface 调用 `cairo_surface_set_device_scale`
  - [x] 修改 `paintTile()`：开头通过 `cairo_surface_get_device_scale` 获取倍率，用逻辑尺寸（`pixel_dims / scale`）计算 `resizeWidth`/`resizeHeight`
  - [x] 修改 `Theme::paint(BackgroundImageConfig)`：移除 `scale != 1.0` 时的临时 surface 分支，统一直接调用 `paintTile`
- 构建与验证
  - [x] 编译 `libclassicui.so`
  - [x] 替换系统 `/usr/lib64/fcitx5/libclassicui.so` 并验证
