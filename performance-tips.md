# 一些关于性能优化的要点

## 性能优化

### 闲杂要点

使用`defer async`标识使js不阻塞页面渲染，`async`不保证顺序执行（所以加载的脚本不能有依赖关系）。

css置顶，js置底，合理利用js异步加载。

css模块化：`link`支持并发，`@import`需知道引用后才能进行加载—可通过构建工具优化

对于函数来说，应该尽可能避免声明嵌套函数（类也是函数），因为这样会造成函数的重复解析。

```javascript
function test1() {
  // 会被重复解析
  function test2() {}
}
```


### V8编译Js代码的过程


<img src='./image/performance/V8.png' width='400px'>

`Ignition` 负责将 `AST` 转化为 `Bytecode`，`TurboFan` 负责编译出优化后的 `Machine Code`，并且 `Machine Code` 在执行效率上优于 `Bytecode`

<img src='./image/performance/codelevel.png' width='500px'>

尽可能保证传入的类型一致。避免去优化（DeOptimized）

`Lazy-Compile`，当函数没有被执行的时候，会对函数进行一次预解析，直到代码被执行以后才会被解析编译，立即执行的函数可以套上括号防止预解析：

```javascript
(function test(obj) {
  return x + x
})
```

一些tips:

+ 可以通过 Audit 工具获得网站的多个指标的性能报告
+ 可以通过 Performance 工具了解网站的性能瓶颈
+ 可以通过 Performance API 具体测量时间
+ 为了减少编译时间，我们可以采用减少代码文件的大小或者减少书写嵌套函数的方式
+ 为了让 V8 优化代码，我们应该尽可能保证传入参数的类型一致。这也给我们带来了一个思考，这是不是也是使用 TypeScript 能够带来的好处之一

### CDN

CDN 的原理是尽可能的在各个地方分布机房缓存数据，这样即使我们的根服务器远在国外，在国内的用户也可以通过国内的机房迅速加载资源。

因此，我们可以将静态资源尽量使用 CDN 加载，由于浏览器对于单个域名有并发请求上限，可以考虑使用多个 CDN 域名。并且对于 CDN 加载静态资源需要注意 CDN 域名要与主站不同，否则每次请求都会带上主站的 Cookie，平白消耗流量。

### 懒加载

懒加载就是将不关键的资源延后加载。

懒加载的原理就是只加载自定义区域（通常是可视区域，但也可以是即将进入可视区域）内需要加载的东西。对于图片来说，先设置图片标签的 src 属性为一张占位图，将真实的图片资源放入一个自定义属性中，当进入自定义区域时，就将自定义属性替换为 src 属性，这样图片就会去下载资源，实现了图片懒加载。

懒加载不仅可以用于图片，也可以使用在别的资源上。比如进入可视区域才开始播放视频等等。

### 懒执行

懒执行就是将某些逻辑延迟到使用时再计算。该技术可以用于首屏优化，对于某些耗时逻辑并不需要在首屏就使用的，就可以使用懒执行。懒执行需要唤醒，一般可以通过定时器或者事件的调用来唤醒。

### 预渲染

可以通过预渲染将下载的文件预先在后台渲染，可以使用以下代码开启预渲染

```html
<link rel="prerender" href="http://example.com">
```

预渲染虽然可以提高页面的加载速度，但是要确保该页面大概率会被用户在之后打开，否则就是白白浪费资源去渲染。

### 预加载

有些资源不需要马上用到，但是希望尽早获取，这时候就可以使用预加载。

预加载其实是声明式的 fetch ，强制浏览器请求资源，并且不会阻塞 onload 事件，可以使用以下代码开启预加载

```html
<link rel="preload" href="http://example.com">
```

预加载可以一定程度上降低首屏的加载时间，因为可以将一些不影响首屏但重要的文件延后加载，唯一缺点就是兼容性不好。

### 图片加载优化

+ 不用图片。很多时候会使用到很多修饰类图片，其实这类修饰图片完全可以用 CSS 去代替。
+ 对于移动端来说，屏幕宽度就那么点，完全没有必要去加载原图浪费带宽。一般图片都用 CDN 加载，可以计算出适配屏幕的宽度，然后去请求相应裁剪好的图片。
+ 小图使用 base64 格式
+ 将多个图标文件整合到一张图片中（雪碧图）
+ 选择正确的图片格式：
  + 对于能够显示 WebP 格式的浏览器尽量使用 WebP 格式。因为 WebP 格式具有更好的图像数据压缩算法，能带来更小的图片体积，而且拥有肉眼识别无差异的图像质量，缺点就是兼容性并不好
  + 小图使用 PNG，其实对于大部分图标这类图片，完全可以使用 SVG 代替
  + 照片使用 JPEG

### DNS预解析

需要一定时间，可以通过预解析的方式来预先获得域名所对应的 IP。

```html
<link rel="dns-prefetch" href="www.example.com">
```

### 防抖

考虑一个场景，有一个按钮点击会触发网络请求，但是我们并不希望每次点击都发起网络请求，而是当用户点击按钮一段时间后没有再次点击的情况才去发起网络请求，对于这种情况我们就可以使用防抖。

防抖函数

```javascript
// func是用户传入需要防抖的函数
// wait是等待时间
const debounce = (func, wait = 50) => {
  // 缓存一个定时器id
  let timer = 0
  // 这里返回的函数是每次用户实际调用的防抖函数
  // 如果已经设定过定时器了就清空上一次的定时器
  // 开始一个新的定时器，延迟执行用户传入的方法
  return function(...args) {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => {
      func.apply(this, args)
    }, wait)
  }
}
```

### 节流

考虑一个场景，滚动事件中会发起网络请求，但是我们并不希望用户在滚动过程中一直发起请求，而是隔一段时间发起一次，对于这种情况我们就可以使用节流。

节流函数：

```javascript
// func是用户传入需要节流的函数
// wait是等待时间
const throttle = (func, wait = 50) => {
  // 上一次执行该函数的时间
  let lastTime = 0
  return function(...args) {
    // 当前时间
    let now = +new Date()
    // 将当前时间和上一次执行函数时间对比
    // 如果差值大于设置的等待时间就执行函数
    if (now - lastTime > wait) {
      lastTime = now
      func.apply(this, args)
    }
  }
}

setInterval(
  throttle(() => {
    console.log(1)
  }, 500),
  1
)
```

**节流和防抖的区别：**节流中用户的操作最终都会被执行，防抖中多余的操作会被忽略