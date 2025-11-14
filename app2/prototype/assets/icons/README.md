# ScoreKeeper 图标库

## 📦 图标列表

本图标库包含所有前端页面需要的SVG图标，采用统一的赛博朋克风格设计。

### 🎨 设计特点

- **霓虹发光效果**: 所有图标都有高斯模糊发光
- **渐变色彩**: 使用青色、紫粉、金色等渐变
- **统一风格**: 2px描边，圆角端点
- **矢量格式**: SVG格式，无损缩放

---

## 📱 TabBar 图标 (4个)

| 图标 | 文件名 | 颜色 | 用途 |
|-----|--------|------|------|
| 🏠 | home.svg | 青色渐变 | 首页导航 |
| 📊 | chart-bar.svg | 紫粉渐变 | 统计页面 |
| 🕐 | history.svg | 青色渐变 | 历史记录 |
| 👤 | user-circle.svg | 紫粉渐变 | 个人中心 |

---

## 🔧 常用操作图标 (10个)

| 图标 | 文件名 | 颜色 | 用途 |
|-----|--------|------|------|
| ← | arrow-left.svg | 青色渐变 | 返回按钮 |
| ➕ | plus.svg | 青色渐变 | 加号/添加 |
| 📤 | share.svg | 紫粉渐变 | 分享功能 |
| 📱 | qrcode.svg | 青色渐变 | 二维码 |
| ⚙️ | settings.svg | 紫粉渐变 | 设置 |
| 🔔 | bell.svg | 青色渐变 | 通知 |
| 🔍 | search.svg | 青色渐变 | 搜索 |
| ✓ | check.svg | 青色渐变 | 确认/完成 |
| ✕ | close.svg | 红色渐变 | 关闭/取消 |
| 🔄 | refresh.svg | 青色渐变 | 刷新 |

---

## 🎮 游戏相关图标 (7个)

| 图标 | 文件名 | 颜色 | 用途 |
|-----|--------|------|------|
| 👥 | users.svg | 青色渐变 | 玩家/用户 |
| 🎮 | gamepad.svg | 混合渐变 | 游戏/对局 |
| 🏆 | trophy.svg | 金色渐变 | 奖杯/排名 |
| 👑 | crown.svg | 金色渐变 | 冠军/第一名 |
| 🔥 | fire.svg | 火焰渐变 | 热门/活跃 |
| 🕐 | clock.svg | 青色渐变 | 时间/时长 |
| ⭐ | star.svg | 金色渐变 | 星星/收藏 |

---

## 📷 其他功能图标 (2个)

| 图标 | 文件名 | 颜色 | 用途 |
|-----|--------|------|------|
| 📥 | download.svg | 紫粉渐变 | 下载/保存 |
| 📷 | camera.svg | 青色渐变 | 相机/扫描 |

---

## 🎨 颜色方案

### 青色渐变 (Cyan Gradient)
```css
#22d3ee → #0ea5e9
```
**用途**: 主要操作、导航、确认

### 紫粉渐变 (Purple Gradient)
```css
#ff48f9 → #a855f7
```
**用途**: 次要操作、辅助功能

### 金色渐变 (Gold Gradient)
```css
#fbbf24 → #f59e0b
```
**用途**: 奖励、排名、重要标识

### 红色渐变 (Red Gradient)
```css
#ef4444 → #dc2626
```
**用途**: 警告、删除、关闭

### 混合渐变 (Mixed Gradient)
```css
#22d3ee → #ff48f9
```
**用途**: 特殊功能、品牌标识

### 火焰渐变 (Fire Gradient)
```css
#ff48f9 → #f59e0b → #ef4444
```
**用途**: 热门、活跃、强调

---

## 📝 使用方法

### 方法一: 直接引用SVG文件

```html
<img src="assets/icons/home.svg" alt="首页" width="24" height="24">
```

### 方法二: 内联SVG

```html
<svg width="24" height="24">
  <!-- 复制SVG文件内容 -->
</svg>
```

### 方法三: CSS背景图

```css
.icon-home {
  width: 24px;
  height: 24px;
  background: url('assets/icons/home.svg') no-repeat center;
  background-size: contain;
}
```

---

## 🔧 自定义修改

### 修改颜色

找到SVG文件中的渐变定义：

```svg
<linearGradient id="cyan-gradient">
  <stop offset="0%" style="stop-color:#22d3ee"/>  <!-- 修改起始色 -->
  <stop offset="100%" style="stop-color:#0ea5e9"/> <!-- 修改结束色 -->
</linearGradient>
```

### 修改发光强度

调整高斯模糊的标准差：

```svg
<filter id="glow">
  <feGaussianBlur stdDeviation="1"/>  <!-- 增大=更强发光 -->
</filter>
```

### 修改描边宽度

```svg
<path stroke-width="2"/>  <!-- 修改描边宽度 -->
```

---

## 📐 尺寸规范

- **默认尺寸**: 24×24 px
- **TabBar图标**: 24×24 px
- **页面图标**: 20-24 px
- **大图标**: 48-64 px

---

## 💡 最佳实践

1. **保持尺寸统一**: 同一场景使用相同尺寸
2. **配色一致性**: 相同功能使用相同颜色
3. **语义化命名**: 文件名清晰表达用途
4. **适当留白**: 图标周围保持适当间距
5. **考虑可访问性**: 提供alt文本说明

---

## 📊 图标总览

**总计**: 23个SVG图标

- TabBar图标: 4个
- 操作图标: 10个
- 游戏图标: 7个
- 其他图标: 2个

---

## 🎯 版本信息

- **版本**: v1.0
- **创建日期**: 2025-11-04
- **设计风格**: 赛博朋克 (Cyberpunk)
- **适用项目**: ScoreKeeper 多人在线计分小程序

---

## 📄 许可说明

本图标库为 ScoreKeeper 项目专用，可自由使用和修改。

