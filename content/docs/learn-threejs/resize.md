---
title: 05.全屏 & 调整大小
type: docs
date: 2024-10-27
sidebar:
  open: true
---
本篇文章基于上篇文章[04.相机](/docs/learn-threejs/cameras)，介绍如何调整Threejs渲染到网页中的画布尺寸。

源码文件：[05.resize](https://github.com/supuwoerc/threejs-roadmap/blob/main/05.resize/src/main.ts)

## 页面尺寸

这部分我们只需要调整 css 和画布大小即可，需要重置网页元素的默认边距和设置对应元素的宽高。

```css
    * {
        padding: 0;
        margin: 0;
    }
    html,
    body,
    #app {
        height: 100%;
        overflow: hidden;
    }
    .webgl {
         outline: none;
    }
```
之后再调整画布的尺寸：

```typescript
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight,
};
```
这样我们就可以使得画布占满视窗，但是不能保证视窗大小改变的时候自动适应，下面我们需要监听 resize 来做出调整。

## Resize Handle 

需要让渲染和 resize 自适应，我们需要监听网页的相关事件，同时调整相机及渲染器的相关设置，具体步骤如下：

* 监听网页 resize 事件，修改相机的裁剪面宽高比例。

* 修改渲染器的尺寸，它将自动帮我们调整画布的大小。

```typescript
// 添加resize事件监听
window.addEventListener("resize", () => {
  sizes.width = window.innerWidth;
  sizes.height = window.innerHeight;
  // 更新相机裁剪面宽高比例
  camera.aspect = sizes.width / sizes.height;
  camera.updateProjectionMatrix();
  renderer.setSize(sizes.width, sizes.height);
});
```
这里需要注意的是，在调整完 `camera.aspect` 后，相机并未更新，我们需要调用 updateProjectionMatrix 来触发矩阵的相关计算。

updateProjectionMatrix 主要作用是根据相机的当前参数（如 fov（视野角度）、aspect（宽高比）、near（近裁剪面距离）、far（远裁剪面距离）等）重新计算和更新投影矩阵。

投影矩阵用于将 3D 场景中的坐标转换为 2D 屏幕坐标，从而确定场景中物体在最终渲染图像中的显示位置和大小。

当相机的相关参数发生变化时（例如窗口大小调整导致宽高比改变），需要调用 updateProjectionMatrix 方法来确保正确的投影效果，以保证渲染结果的准确性和视觉上的正确性。

这里代码编辑器经常会提示另外一个方法：`updateMatrix`，这里介绍一下这两个方法的区别：

* updateProjectionMatrix 方法：主要用于根据相机的当前参数（如视野角度、宽高比、近裁剪面和远裁剪面距离等）重新计算和更新投影矩阵。这个矩阵决定了如何将 3D 场景中的物体投影到 2D 屏幕上，影响物体的显示大小和形状。

* updateMatrix 方法：通常用于更新对象的模型矩阵。模型矩阵描述了对象在场景中的位置、旋转和缩放。当对象的位置、旋转或缩放发生变化时，调用 updateMatrix 来更新这个矩阵，以确保在后续的渲染和计算中能够正确处理对象的变换。

## 渲染像素比

像素比例决定了渲染器在屏幕上绘制的像素数量与设备屏幕物理像素数量的比例，不同用户的设备像素比不一致，同时很多用户可能拥有多台像素比不一样的显示器，

我们需要设置相关渲染逻辑中的像素比，避免产生锯齿，提高渲染的视觉效果和一致性。

在 Three.js 中，`renderer.setPixelRatio()` 方法用于设置渲染器的像素比例。

例如，如果将像素比例设置为 2，并且设备屏幕的分辨率为 800x600 像素，那么渲染器实际上会以 1600x1200 像素进行渲染，然后再缩放到设备屏幕上显示。

这可以提高渲染的清晰度和细节，特别是在高分辨率设备（如 retina 显示屏）上。

如果设置为 1，则渲染器将以与设备屏幕物理像素相同的数量进行渲染。我们需要平衡的是渲染的的性能开销及实际的渲染效果。

MDN 上关于设备像素比的文档：[devicePixelRatio](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/devicePixelRatio)

设置渲染器的像素比很简单，我们只需要直接调用 setPixelRatio 方法：

```typescript
// 设置渲染的像素比
renderer.setPixelRatio(window.devicePixelRatio)
```
如果你的显示器 devicePixelRatio 为 1，调整前后的渲染是没任何变化的，但是如果你的 devicePixelRatio > 1 ，你的渲染结果会更加细腻，图像边缘的锯齿变的很小。 

### 平衡性能

很多时候我们需要平衡渲染效果和性能，例如用户的显示器 devicePixelRatio 很大，但是显卡却不能支撑很大量的图像矩阵的计算，那么用户的主观感受就是渲染十分卡顿。

一般来说，我们会限制渲染器的 devicePixelRatio 最多为 2 ,超过这个比例实际上带来的性能开销远大于渲染效果的提升。

```typescript
// 设置渲染的像素比,限制最大像素比为2
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```
同时，为了避免用户在不同显示器内拖动浏览器窗口导致 devicePixelRatio 的变化带来的挑战，我们需要在 resize 逻辑中也加上这部分的逻辑。

## 全屏

很多场景我们需要设置视窗全屏展现，这部分我们借助浏览器提供的 全屏 API 来实现画布的全屏展示。

MDN 上的全屏 API 介绍文档：[Fullscreen_API](https://developer.mozilla.org/zh-CN/docs/Web/API/Fullscreen_API)

这个 API 需要考虑兼容性，NPM 上有很多已经实现兼容各类浏览器的库，需要的话可以使用相关的库。

我们简单的在 Chrome 中实现：

```typescript

```