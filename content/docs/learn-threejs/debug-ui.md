---
title: 07.调试
type: docs
date: 2024-10-31
sidebar:
  open: true
---
本篇文章基于上篇文章[06.几何形状](/docs/learn-threejs/geometries)，介绍如何在借助 GUI 库来调试 Three.js 程序。

源码文件：[07.debug-ui](https://github.com/supuwoerc/threejs-roadmap/blob/main/07.debug-ui/src/main.ts)

## 介绍

随着项目的复杂度不断提升，我们需要调试我们的程序，通常前端开发者调试可以通过以下几种方式：

* 使用浏览器的开发者工具：可以查看控制台输出的错误信息、检查元素的属性和样式等。

* 打印日志：在代码中使用 console.log() 输出关键变量和状态的信息，帮助跟踪程序的执行流程。

* 逐步调试：如果您使用的开发环境支持（如 Visual Studio Code），可以设置断点进行逐步调试。

但是 Three.js 的调试比较特殊，我们有时需要所见即所得的查看渲染试图，这种场景下借助以上三种方式会需要开发者不断的去修改源码后查看结果。

于是社区中有了很多 GUI 的调试工具，可以修改程序中要调试的关键参数，达到在网页中调试程序的目的。

本文主要围绕热门的 [lil-gui](https://lil-gui.georgealways.com/) 来介绍。

## 快速入门

关于 lil-gui的使用查看官方文档基本就可以了解：[lil-gui doc](https://lil-gui.georgealways.com/)。

本文直接介绍源码中的 lil-gui 应用，同时添加注释加以解释。

```typescript
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/Addons.js";
import GUI from "lil-gui";
import gsap from "gsap";

const gui = new GUI({
    // 面板宽度
    width: 300,
    // 名称
    title: "调试器",
    // 关闭全部文件夹
    closeFolders: true,
});

// 隐藏面板
gui.hide();

const callback = () => {
    if (window.location.hash == "#debug") {
        gui.show();
    }
};
callback();
// 根据hash判断要不要展示面板
window.addEventListener("onhashchange", () => {
    callback();
});

const sizes = {
    width: window.innerWidth,
    height: window.innerHeight,
};
// 00 创建调试对象
const debugObject: any = {
    materialColor: "#00ff00",
    spin: null,
    segment: 1,
};
const scene = new THREE.Scene();
const geometry = new THREE.BoxGeometry(
    1,
    1,
    1,
    debugObject.segment,
    debugObject.segment,
    debugObject.segment
);

const material = new THREE.MeshBasicMaterial({
    color: debugObject.materialColor,
});
const mesh = new THREE.Mesh(geometry, material);

debugObject.spin = () => {
    gsap.to(mesh.rotation, {
        z: mesh.rotation.z - Math.PI * 2,
        duration: 2,
    });
};
// 添加需要调整的属性到gui

// 00 创建文件夹
const meshGui = gui.addFolder("mesh");
const materialGui = gui.addFolder("material");
materialGui.open();

// 01 添加数字类型的属性
meshGui
    .add(mesh.position, "y")
    .min(-2)
    .max(2)
    .step(0.01)
    .name("mesh.pisition.y");

// 02 添加boolean属性
meshGui.add(mesh, "visible").name("mesh.visible");
materialGui.add(material, "wireframe").name("material.wireframe");

// 03 添加color-picker(由于色彩管理，直接设置material的颜色会使得渲染结果颜色不一样，需要借助自定义对象去设置）
materialGui
    .addColor(debugObject, "materialColor")
    .name("material.color")
    .onChange(() => {
        // 直接修改颜色material会根据颜色做一次色彩空间的转换，所以存在色差
        console.log(material.color.getHexString());
        // set 方法接受srgb的颜色参数，内部就处理这种转换
        material.color.set(debugObject.materialColor);
        // 符合预期，渲染的颜色和color-picker选择的颜色一致
        console.log(material.color.getHexString());
    });

// 04 自定义方法
meshGui.add(debugObject, "spin").name("spin");

// 05 添加下拉选择
meshGui
    .add(debugObject, "segment", [1, 2, 3])
    .name("geometry.segments")
    .onFinishChange((c: number) => {
        // 重新创建几何对象（不推荐的行为）
        mesh.geometry = new THREE.BoxGeometry(1, 1, 1, c, c, c);
        // 销毁旧几何对象
        geometry.dispose();
    });

scene.add(mesh);
const camera = new THREE.PerspectiveCamera(75, sizes.width / sizes.height);
camera.position.z = 3;
scene.add(camera);
const renderer = new THREE.WebGLRenderer({
    canvas: document.querySelector<HTMLCanvasElement>(".webgl")!,
});
renderer.setSize(sizes.width, sizes.height);
renderer.setPixelRatio(Math.min(2, window.devicePixelRatio));
const control = new OrbitControls(camera, renderer.domElement);
control.enableDamping = true;
control.dampingFactor = 0.01;
const tick = () => {
    control.update();
    renderer.render(scene, camera);
    window.requestAnimationFrame(tick);
};
tick();

window.addEventListener("resize", () => {
    sizes.width = window.innerWidth;
    sizes.height = window.innerHeight;
    camera.aspect = sizes.width / sizes.height;
    camera.updateProjectionMatrix();
    renderer.setSize(sizes.width, sizes.height);
    renderer.setPixelRatio(Math.min(2, window.devicePixelRatio));
});
```

## 颜色问题

在使用 lil-gui 调试 three.js 程序时，如果你发现使用 gui.addColor(material, "color") 直接设置颜色时存在色差，这可能是由于几个原因：

* 色彩空间不一致：three.js 默认使用线性色彩空间处理颜色，而大多数色彩选择器（包括许多 GUI 库提供的）返回的颜色值是在 sRGB 色彩空间中。当你将从色彩选择器获取的 sRGB 色彩值直接设置到材料的颜色属性时，这可能导致色彩看起来与预期不同，因为没有进行色彩空间的转换。

* Gamma 校正：在 three.js 中，材质的颜色可能会经过 gamma 校正，尤其是在使用 WebGLRenderer 的 { gammaOutput: true } 配置时。如果你的渲染器配置了 gamma 校正，但你通过 GUI 直接设置的颜色未经过相应的校正，就可能会出现色差。

* 材料和光照效果：材质的外观不仅仅由颜色决定，还受到场景中光照条件和材质其他属性（如金属度、粗糙度等）的影响。色差可能是因为环境中的光照效果与你通过 GUI 设置的颜色交互产生了不同的视觉效果。

