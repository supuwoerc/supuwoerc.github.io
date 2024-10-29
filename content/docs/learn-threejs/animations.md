---
title: 03.动画
type: docs
date: 2024-10-25
sidebar:
  open: true
---

本篇文章基于上篇文章[02.移动/缩放/旋转对象](/docs/learn-threejs/transform-objects)，介绍如何在浏览器中使用Threejs渲染动画。

源码文件：[03.animations](https://github.com/supuwoerc/threejs-roadmap/blob/main/03.animations/src/main.ts)

## 什么是动画

动画实际上是利用人眼的视觉暂留，即在很短的时间间隔内渲染多张图片，人眼主观感受就是动画，浏览器提供了 requestAnimationFrame 来允许程序定义在下次重绘之前的方法，我们可以借助它来更新动画。

requestAnimationFrame的主要优点是能够根据浏览器的刷新率自动调整回调的执行频率，从而实现流畅的动画效果，并在页面不可见时暂停动画，节省系统资源。

创建动画的逻辑大致是这样的：

```typescript
const tick = () => {
  // update something...
  // render
  window.requestAnimationFrame(tick);
};
tick();
```

## 快速实现一个动画

我们借助之前 [02.移动/缩放/旋转对象](/docs/learn-threejs/transform-objects) 加以改造就可以实现一个简单的旋转动画，下面是源码：

```typescript
import * as THREE from "three";

const sizes = {
  width: 800,
  height: 600,
};

// 创建一个场景
const scene = new THREE.Scene();
// 创建几何形状
const geometry = new THREE.BoxGeometry(1, 1, 1);
// 创建材质
const material = new THREE.MeshBasicMaterial({ color: "red" });
// 组合mesh
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
// 创建相机
const camera = new THREE.PerspectiveCamera(75, sizes.width / sizes.height);
scene.add(camera);
camera.position.z = 3;
// 创建渲染器
const renderer = new THREE.WebGLRenderer({
  canvas: document.querySelector(".webgl") as HTMLCanvasElement,
});
renderer.setSize(sizes.width, sizes.height);
const tick = () => {
  // 更新物体
  mesh.rotation.z += 0.001;
  // 渲染
  renderer.render(scene, camera);
  window.requestAnimationFrame(tick);
};
tick();
```
可以看到，我们只是在每次重绘时修改了 `mesh.rotation.z` 属性的值，就能让立方体旋转起来！

## 渲染差异

前面我们知道 requestAnimationFrame 是在浏览器每一次重绘时调用，那么现在就存在一个问题：不同电脑的显卡性能不一致导致的渲染帧率不一致导致的渲染差异。

例如：120fps 下的动画和 60fps 下的动画，在一分钟内一个重绘了 120 次，而另外一个只重绘了 60 次，那么这两个动画必然表现是不同的。

如何解决这个问题呢？可以借助下面的两种方法。

### 时间差

**核心思路**：每次更新前计算和上次更新时的时间差，时间差毫秒值 * 1ms时间里的步进值 = 要更新的量

```typescript
// 借助时间抹平渲染差异
let lastRenderTime = Date.now();
const tick = () => {
  const current = Date.now();
  const offset = current - lastRenderTime;
  lastRenderTime = current;
  // 更新物体
  mesh.rotation.z += Math.PI * 0.001 * offset;
  // 渲染
  renderer.render(scene, camera);
  window.requestAnimationFrame(tick);
};
tick();
```

### THREE.Clock

**核心思路**：每次更新前计算开始运行到当前的时间，要设置的属性值 = 运行时长 * 1s需要更新的步进值 

```typescript
const clock = new THREE.Clock();
const tick = () => {
  // 更新物体：clock.getElapsedTime()获取clock运行到现在的秒数
  mesh.rotation.z = Math.PI * clock.getElapsedTime();
  // 渲染
  renderer.render(scene, camera);
  window.requestAnimationFrame(tick);
};
tick();
```

## 圆周动画

在平面直角坐标系中，若以原点为圆心，r 为半径的圆上一点的坐标为 (x, y)，该点与 x 轴正半轴的夹角为 θ（弧度制），则它们之间的关系可以用以下三角函数表示：
* x = r * cos(θ)
* y = r * sin(θ)

当 θ 从 0 增加到 2π 时，(x, y) 将遍历整个圆周，我们可以借助三角函数和 Clock 来快速实现一个圆周动画：

```typescript
const clock = new THREE.Clock();
const tick = () => {
  // 更新物体
  mesh.position.x = Math.cos(clock.getElapsedTime());
  mesh.position.y = Math.sin(clock.getElapsedTime());
  // 渲染
  renderer.render(scene, camera);
  window.requestAnimationFrame(tick);
};
tick();
```
换个方式，我们将动画运用到相机上，得到的动画是一样的效果，不过这次圆周运动的是相机：

```typescript
const clock = new THREE.Clock();
const tick = () => {
  // 更新物体
  camera.position.x = Math.cos(clock.getElapsedTime());
  camera.position.y = Math.sin(clock.getElapsedTime());
  // 渲染
  renderer.render(scene, camera);
  window.requestAnimationFrame(tick);
};
tick();
```
加上一行代码，让相机lookAt立方体中心，又得到一个不一样的动画：

```typescript
const clock = new THREE.Clock();
const tick = () => {
  // 更新物体
  camera.position.x = Math.cos(clock.getElapsedTime());
  camera.position.y = Math.sin(clock.getElapsedTime());
  camera.lookAt(mesh.position)
  // 渲染
  renderer.render(scene, camera);
  window.requestAnimationFrame(tick);
};
tick();
```

## 配合GSAP

GSAP（GreenSock Animation Platform）是一个功能强大且性能高效的 JavaScript 动画库。
为什么选择 GSAP 配合 Three.js 来制作动画：

* 强大的动画控制：GSAP 提供了非常精细和灵活的动画控制能力，可以轻松实现复杂的动画效果，包括缓动函数、动画序列、暂停、恢复、反转等，这对于创建引人入胜的 Three.js 场景动画非常有用。

* 良好的性能：GSAP 经过优化，能够在各种设备和浏览器上提供流畅的动画性能，减少卡顿和跳帧的情况。

* 易于理解的 API：GSAP 的 API 相对简单直观，易于学习和使用，开发者可以快速上手并创建出所需的动画效果，降低了开发难度和时间成本。

* 缓动函数支持：GSAP 拥有丰富的缓动函数库，能够为 Three.js 中的物体运动添加自然而平滑的过渡效果，使动画看起来更加真实和吸引人。

* 跨浏览器兼容性：GSAP 具有出色的跨浏览器兼容性，确保在不同的浏览器环境中动画都能正常工作，为用户提供一致的体验。

* 与 Three.js 集成方便：尽管 Three.js 本身有动画功能，但结合 GSAP 可以补充和增强其动画能力，提供更多的选择和灵活性。

### 安装依赖
```shell
pnpm add gsap@3.5.1
```
### 动画实现

```typescript
gsap.to(mesh.rotation, {
  duration: 2,
  x: 2,
  y: 2,
  yoyo: true,
  repeat: -1,
});
const tick = () => {
  renderer.render(scene, camera);
  requestAnimationFrame(() => {
    tick();
  });
};
tick();
```
以上就是如何简单的借助 Three.js和相关方法和第三方依赖来实现动画的基本示例。