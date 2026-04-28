# 🗽鼠标交互式浮雕效果实现说明

注意：本效果参照https://wild.as/labs/building-wild-week-athens

并非一比一实现，只是个人学习。

## 🎯1. 效果目标

这个案例要实现的是一种基于鼠标位置的材质 reveal 效果：

- 默认状态下，画面显示为白色石膏/浮雕材质。
- 鼠标移动到浮雕图像区域内时，鼠标附近的区域逐渐显现金属黄金质感。
- 鼠标离开图像实际显示区域后，金属光照和高光消失。
- 浮雕的立体感不是通过真实 3D 模型实现，而是通过 **Normal Map 法线贴图 + Shader 光照计算** 模拟出来。

我认为这个方案适合用于网页中的交互式视觉实验、作品集首页、品牌展示页、艺术化 landing page 等场景，当然事实上Wild就是这样做的（有点像是我在说废话）

---

## 🗒️2. 核心原理

整个效果由三张贴图和一个 WebGL shader 组成：

```text
relief.png   基础浮雕图 / base color
normal.png   法线贴图 / normal map
mask.png     黑白遮罩 / material mask
```

Shader 会根据鼠标位置动态计算一个局部光源，并结合 normal map 重新计算每个像素的光照。然后通过 mask 决定哪些区域可以被切换成黄金材质。

---

## 🧩3. 主要先自己生成三张贴图

### 3.1 relief.png

`relief.png` 是画面中默认可见的白色浮雕图。

它主要提供：

- 浮雕的基础颜色；
- 原始明暗层次；
- 石膏或陶瓷质感的底色；
- 最终和金属材质混合的 base color。

那经过我个人测试：

- 不要有太强的方向性阴影；
- 不要有过曝高光；
- 最好是低对比度的白色/米白色浮雕图。

如果 `relief.png` 本身已经有很重的阴影，而 shader 又重新计算光照，最终会出现“双重光照”的问题。

这个我直接把素材导入Gemini让它帮我生成的，提示词如下：

```
生成雕塑浅浮雕图 低饱和 背景干净 阴影不要太黑 主体边缘清楚;雕像不要粗糙，光滑质感 降低阴影;降低高光保留整体白色/石膏色不要有强方向光
```

---

### 3.2 normal.png

`normal.png` 是法线贴图，通常呈现紫蓝色。

这个我是使用了网站https://cpetry.github.io/NormalMap-Online/直接对我的雕塑图进行调整生成，

它不是普通颜色图，而是把每个像素的表面朝向编码到了 RGB 通道里：

```text
R 通道：X 方向法线
G 通道：Y 方向法线
B 通道：Z 方向法线
```

Shader 读取 normal map 后，可以知道某个像素表面是朝左、朝右、朝上、朝下，还是正对屏幕。这样即使画面本身只是一个 2D 平面，也能模拟出浮雕在不同光照下的明暗变化。

代码中的关键逻辑：

```glsl
vec3 n = texture(u_normal, uv).rgb * 2.0 - 1.0;
n.y = mix(n.y, -n.y, u_flipNormalY);
n.xy *= u_normalStrength;
return normalize(n);
```

其中：

- `texture(u_normal, uv).rgb * 2.0 - 1.0` 把 0–1 的颜色值转换成 -1 到 1 的法线向量；
- `u_flipNormalY` 用于切换 OpenGL / DirectX normal map 的绿色通道方向；
- `u_normalStrength` 用于控制凹凸强度。

注意：不要直接从带有复杂阴影的图片自动生成 normal map。这样会把“阴影”误判为“高度差”，导致光照方向混乱。

这个是我上传到网站给的图，这个没用Gemini生成因为AI比较不可控。

![image-20260429014502406](C:\Users\Jinju\AppData\Roaming\Typora\typora-user-images\image-20260429014502406.png)

同时我将上图的文件中提出的参数作出解释：

**Strength 强度**

控制法线扰动幅度。数值越大，材质表面的凹凸感越强，光影变化越明显；数值太大会显得塑料感、噪点感或边缘过硬。数值小则更平滑、更接近原始平面。

一般用法：石头、砖墙、粗糙木纹可以高一些；皮革、织物、细微纹理应低一些。

**Level 层级 / 高度基准**

它通常决定从原图亮度推导高度时的基准或中间值。可以理解为调整“哪些区域被当成高，哪些区域被当成低”。数值变化会影响凹凸的整体分布，有时会让原本只是颜色变化的地方被解释成高度变化。

如果材质看起来整体鼓起来、凹进去，或者高低关系不自然，可以调这个。

**Blur/Sharp 模糊 / 锐化**

控制生成法线前对高度信息的平滑或锐化。

数值偏模糊时，细碎噪点会减少，凹凸过渡更柔和，适合皮肤、布料、泥土等自然材质。数值偏锐化时，边缘会更硬，细节更突出，适合砖缝、金属刻痕、硬表面纹理。但过锐会产生锯齿、闪烁和不自然的高光。

**Filter 滤波器，比如 Sobel**

这是从灰度/高度图计算法线方向的方法。Sobel 会强调边缘，是很常见的边缘检测滤波器。它适合从图片亮暗变化中提取凹凸边界，比如砖块、石纹、浮雕纹理。

不同滤波器的区别主要在于：有的更平滑，有的更锐利，有的更强调细节。Sobel 通常会让边缘感更明显。

**Invert 反转 R / G / Height**

这几个很关键。

R 通道通常对应法线的 X 方向，影响左右方向的凹凸光照。

G 通道通常对应法线的 Y 方向，影响上下方向的凹凸光照。不同引擎对 G 通道方向要求不同。比如有些软件使用 OpenGL 法线方向，有些使用 DirectX 法线方向。如果导入后凹凸看起来“反了”，最常见就是需要反转 G 通道。

Height 是反转高度关系。勾选后，原本凸出的区域会变成凹陷，原本凹陷的区域会变成凸起。比如砖缝应该凹下去，如果生成后砖缝看起来凸出来，就反转 Height。

**Z Range：-1 to +1**

这影响法线向量的 Z 分量范围，也就是表面法线朝外的程度。开启 `-1 to +1` 通常表示使用完整的法线取值范围。它会影响法线图被解释时的方向范围和强度表现。

实际调材质时，这个一般不如 Strength、Invert G、Height 反转常用。除非你发现导出的法线图在目标软件里光照方向明显异常，否则通常保持默认。



而且目前得出的**结论**是：

如果凹凸太弱，调高 Strength。

如果纹理太噪，调 Blur/Sharp 往模糊方向调。

如果砖缝、裂缝、刻痕看起来凸出来，反转 Height。

如果导入 Unity、Unreal、Blender 等软件后，光从上方照却像从下方照，通常反转 G 通道。

如果边缘太硬或假，降低 Strength 或减少锐化。

对大多数 PBR 材质，建议先用：Strength 中低、Blur/Sharp 轻微平滑、Filter 用 Sobel，然后重点检查 **G 通道方向** 和 **Height 是否反了**。

---

### 3.3 mask.png

`mask.png` 是控制金属 reveal 区域的黑白遮罩。

推荐语义：

```text
黑色 = 不参与金属效果
灰色 = 弱参与
白色 = 强参与
```

理想情况下：

```text
人物浮雕主体：白色
底板区域：深灰
外部背景：黑色
```

如果 mask 是整块白色矩形，那么整块底板都会被金属化，画面容易变成一大片发亮的金属板，而不是浮雕主体局部变金属。

代码中把 mask 拆成了两层：

```glsl
float revealMask = smoothstep(u_revealMaskLow, u_revealMaskHigh, rawMask);
float specMask = smoothstep(u_specMaskLow, u_specMaskHigh, rawMask);
```

其中：

- `revealMask` 控制哪里允许从石膏材质过渡到黄金材质；
- `specMask` 控制哪里允许出现强高光。

这样可以让底板弱参与、人物主体强参与，从而得到更干净的视觉效果。

这一步依旧Gemini发力（偷懒的我）

提示词如下：

```
生成黑白遮罩图：人物浮雕：白色 255底板：深灰 35–80外部背景：黑色 0
```

![image-20260429015113392](C:\Users\Jinju\AppData\Roaming\Typora\typora-user-images\image-20260429015113392.png)

---

## 4. 鼠标交互逻辑（直接交给Agent来做可以不用看）

页面中的 canvas 和图像并不一定完全等比例铺满。为了让鼠标光照准确跟随图像，需要把鼠标坐标从 canvas 空间转换到图像 UV 空间。

关键代码：

```js
const canvasUvX = px / rect.width;
const canvasUvY = py / rect.height;

const imageU = (canvasUvX - 0.5) / imageScale[0] + 0.5;
const imageV = (canvasUvY - 0.5) / imageScale[1] + 0.5;
```

其中：

- `canvasUvX / canvasUvY` 是鼠标在整个 stage 里的归一化坐标；
- `imageScale` 是图像在 canvas 中 contain 缩放后的比例；
- `imageU / imageV` 是真正对应到图片上的 UV 坐标。

为了实现“鼠标不在图片区域内不要光”，需要判断 UV 是否在 0–1 之间：

```js
const isInsideImage =
  imageU >= 0.0 &&
  imageU <= 1.0 &&
  imageV >= 0.0 &&
  imageV <= 1.0;
```

然后把结果传给 shader：

```js
targetLightActive = isInsideImage ? 1.0 : 0.0;
```

在 fragment shader 中用 `u_lightActive` 控制 reveal 和高光：

```glsl
reveal *= revealMask * u_lightActive;
localLight *= u_lightActive;
```

这样鼠标虽然还在 stage 内，但只要不在图像实际显示区域内，金属效果就不会出现。

---

## 5. 光照计算（直接交给Agent来做可以不用看）

Shader 中把鼠标当作一个局部光源：

```glsl
vec3 pos = vec3(p, 0.0);
vec3 lightPos = vec3(lp, u_lightHeight);

vec3 L = normalize(lightPos - pos);
vec3 V = vec3(0.0, 0.0, 1.0);
vec3 H = normalize(L + V);
```

含义：

- `pos` 是当前像素在平面上的位置；
- `lightPos` 是鼠标对应的光源位置；
- `L` 是当前像素指向光源的方向；
- `V` 是视线方向；
- `H` 是半角向量，用于计算高光。

漫反射：

```glsl
float ndotl = max(dot(normal, L), 0.0);
```

高光：

```glsl
float ndoth = max(dot(normal, H), 0.0);
float spec = pow(ndoth, u_specularPower) * u_specularIntensity;
```

Fresnel 边缘反射：

```glsl
float fresnel = pow(1.0 - ndotv, u_fresnelPower) * u_fresnelIntensity;
```

---

## 6. 黄金材质的实现（直接交给Agent来做可以不用看）

**需要说明一点的是Wild并不是采用黄金，而是一种非常Amazing的金属效果，这种效果他们甚至构建了WebGL工具在Framer方便调整，令人出乎意料。**

**而目前黄金材质是因为本人太菜而且花了极少时间不出错情况下调整的，如果不想要黄金材质可以参考原帖指引Agent给你写一个工具方便你调参然后调整到你想要的效果。**

黄金质感不是简单把颜色改成黄色。真实的黄金视觉通常包含：

- 深棕色暗部；
- 暖黄色中间调；
- 少量高亮但不纯白的反射；
- 边缘或斜面处更明显的 Fresnel 反射。

代码中用三组颜色控制黄金：

```js
goldDark:  [0.16, 0.075, 0.025],
goldMid:   [0.95, 0.58, 0.13],
goldLight: [1.00, 0.82, 0.34],
```

其中：

- `goldDark` 决定暗部厚重感；
- `goldMid` 决定整体黄金底色；
- `goldLight` 决定高光颜色。

如果想要亮黄金，可以提高：

```js
ambient
lightPower
specularIntensity
fresnelIntensity
goldMid
goldLight
```

但不建议无限提高 `lightPower`，否则会发白，变成黄色塑料。更好的方式是提高金色中间调和集中高光。

推荐亮黄金参数：

```js
ambient: 0.24,
lightPower: 1.45,
specularPower: 88.0,
specularIntensity: 0.95,
fresnelIntensity: 0.28,

goldDark: [0.16, 0.075, 0.025],
goldMid: [0.95, 0.58, 0.13],
goldLight: [1.00, 0.82, 0.34],
```

---

## 7. 材质混合（直接交给Agent来做可以不用看）

最终画面不是直接显示黄金，而是在石膏和黄金之间做混合：

```glsl
vec3 objectColor = mix(plaster, gold, reveal);
```

其中 `reveal` 由三部分共同决定：

```text
鼠标距离
mask 强度
鼠标是否在图像区域内
```

核心逻辑：

```glsl
float reveal = 1.0 - smoothstep(
  u_highlightRadius,
  u_highlightRadius + u_revealSoftness,
  dist
);

reveal *= revealMask * u_lightActive;
```

含义：

- 鼠标越近，`reveal` 越接近 1；
- 鼠标越远，`reveal` 越接近 0；
- mask 为黑色时，永远不 reveal；
- 鼠标不在图片区域内时，永远不 reveal。

---

## 8. 关键参数说明（只是单纯实现可以不看这一部分）

### reliefScale

控制图像在 stage 中的显示大小。

```js
reliefScale: 0.92
```

数值越小，图像越小；数值越大，图像越接近铺满。

---

### flipNormalY

控制是否反转 normal map 的绿色通道。

```js
flipNormalY: true
```

如果浮雕看起来“凹凸反了”，比如鼻梁像凹进去、衣服褶皱方向不对，就切换这个值：

```js
flipNormalY: false
```

---

### normalStrength

控制法线凹凸强度。

```js
normalStrength: 1.28
```

增大后浮雕细节更明显，但过大会产生硬边、噪声和不自然高光。

---

### highlightRadius

控制鼠标影响范围。

```js
highlightRadius: 0.34
```

数值越大，金属 reveal 范围越大。

---

### revealSoftness

控制 reveal 边缘柔和度。

```js
revealSoftness: 0.22
```

数值越大，过渡越柔和；数值越小，边界越硬。

---

### specularPower

控制高光集中程度。

```js
specularPower: 88.0
```

数值越大，高光越小、越锐利，金属感更强；数值越小，高光越散，容易像塑料。

---

### specularIntensity

控制高光亮度。

```js
specularIntensity: 0.95
```

数值越大，高光越亮。过高会发白。

---

### revealMaskLow / revealMaskHigh

控制 mask 中哪些灰度可以参与材质 reveal。

```js
revealMaskLow: 0.16,
revealMaskHigh: 0.88
```

如果底板也被金属化太严重，可以调高这两个值。

---

### specMaskLow / specMaskHigh

控制 mask 中哪些区域可以出现强高光。

```js
specMaskLow: 0.58,
specMaskHigh: 0.98
```

如果整块板都在发亮，调高 `specMaskLow`。

---

## ❔9. 常见问题（我遇到的一些问题）

**注意：其实最最最重要的是三张图，三张图按照原贴的效果标注生成就能避免很多材质效果实现不理想的问题。**

**注意：其实最最最重要的是三张图，三张图按照原贴的效果标注生成就能避免很多材质效果实现不理想的问题。**

**注意：其实最最最重要的是三张图，三张图按照原贴的效果标注生成就能避免很多材质效果实现不理想的问题。**

**重要事情说三遍!!!**

### 9.1 整块板都变成黄金

原因通常是 `mask.png` 太白，或者整块矩形都是白色。

解决：

- 把人物主体设为白色；
- 把底板改成深灰；
- 把背景设为黑色；
- 提高 `specMaskLow` 和 `revealMaskLow`。

---

### 9.2 黄金太亮、发白

降低：

```js
lightPower
specularIntensity
goldLight
```

也可以降低 shader 中的高光项。

---

### 9.3 黄金太暗、不明显

提高：

```js
goldMid
goldLight
specularIntensity
fresnelIntensity
```

如果只是“不够金属”，优先提高 `specularPower`，让高光更集中。

---

### 9.4 凹凸方向反了

切换：

```js
flipNormalY: true
```

或：

```js
flipNormalY: false
```

这是 normal map 的 OpenGL / DirectX 方向差异导致的。

---

### 9.5 鼠标和高光位置对不上

通常是鼠标坐标没有转换到图像 UV，或者图像被 contain 缩放后没有补偿 `imageScale`。

需要确认代码中使用的是：

```js
const imageU = (canvasUvX - 0.5) / imageScale[0] + 0.5;
const imageV = (canvasUvY - 0.5) / imageScale[1] + 0.5;
```

而不是直接把 canvas 坐标传给 shader。

---

## 🛠️10. 推荐资产制作流程(快速看着一个流程了解，其他Agent做就行)

推荐流程：

1. 准备一张干净的白色浮雕图，作为 `relief.png`。
2. 生成或绘制对应的 `normal.png`。
3. 手工或通过分割工具制作 `mask.png`。
4. 保证三张图构图完全一致。
5. 尽量保证三张图尺寸一致，例如都是 `1024 × 1024`。
6. 在代码中测试 `flipNormalY`。
7. 根据目标视觉调节黄金参数。

举例子，三个图片尺寸一致：

```text
relief.png  1024 × 1024
normal.png  1024 × 1024
mask.png    1024 × 1024
```

---

## 📇11. 文件结构

```text
project/
├── index.html
├── relief.png
├── normal.png
└── mask.png
```

本项目不依赖 Three.js，直接使用 WebGL2。

---

## ⚙️12. 运行方式

该项目如果你拷贝我的由于浏览器对本地图片加载有安全限制，建议不要直接双击打开 HTML 文件，而是通过本地服务器运行。

例如使用 VS Code Live Server，或者使用 Python：

```bash
python -m http.server 5173
```

然后访问：

```text
http://localhost:5173
```

---

## 🌟13. 要点总结

这个效果的关键不是单一 shader 参数，而是资产和 shader 的配合：

```text
relief.png 决定默认视觉
normal.png 决定浮雕光照细节
mask.png 决定哪些区域能变金属
鼠标 UV 决定光源位置
u_lightActive 决定鼠标离开图片区域后是否关闭光照
goldDark / goldMid / goldLight 决定黄金材质层次
specularPower / specularIntensity 决定金属高光质量
```

**最最最后我还是说明下效果不佳优化三张图效果，然后调整材质参数，最后再光线。**

祝你玩的愉快🫰😉也感谢Wild能带来如此惊艳的作品！！！！！！！！
