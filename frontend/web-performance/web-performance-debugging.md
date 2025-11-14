# 前端网页性能调试问题与解决方案研究

## 目录
1. [概述](#概述)
2. [常见性能问题](#常见性能问题)
3. [调试工具与方法](#调试工具与方法)
4. [解决方案详解](#解决方案详解)
5. [实战案例](#实战案例)
6. [最佳实践](#最佳实践)

---

## 概述

前端性能优化是一个系统工程，涉及加载速度、渲染效率、交互响应等多个方面。在调试和优化过程中，开发者经常会遇到各种问题。本文档系统地整理了这些问题及其解决方案。

### 性能指标
- **FCP (First Contentful Paint)**: 首次内容绘制时间
- **LCP (Largest Contentful Paint)**: 最大内容绘制时间
- **FID (First Input Delay)**: 首次输入延迟
- **CLS (Cumulative Layout Shift)**: 累积布局偏移
- **TTI (Time to Interactive)**: 可交互时间
- **TBT (Total Blocking Time)**: 总阻塞时间

---

## 常见性能问题

### 1. 加载性能问题

#### 问题1.1: 首屏加载时间过长
**现象**:
- 用户看到白屏时间超过3秒
- FCP和LCP指标过高
- 用户体验差，跳出率高

**常见原因**:
- 打包后的JavaScript文件过大（超过1MB）
- 没有代码分割，首页加载了所有代码
- 图片资源未优化，体积过大
- 字体文件加载阻塞渲染
- 服务器响应慢或网络延迟高
- 过多的同步脚本阻塞HTML解析

**调试方法**:
```bash
# 使用 Lighthouse 进行性能分析
npx lighthouse https://your-website.com --view

# 使用 webpack-bundle-analyzer 分析包体积
npm install --save-dev webpack-bundle-analyzer
```

**Chrome DevTools 调试步骤**:
1. 打开 DevTools -> Performance 面板
2. 点击录制按钮，刷新页面
3. 查看 Waterfall 图，找出阻塞渲染的资源
4. 分析 Main Thread 活动，找出长任务

**解决方案**:
```javascript
// 1. 路由级代码分割（React示例）
import { lazy, Suspense } from 'react';

const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </Suspense>
  );
}

// 2. 组件级代码分割
const HeavyComponent = lazy(() => import('./HeavyComponent'));

// 3. 预加载关键资源
<link rel="preload" href="/critical.css" as="style">
<link rel="preload" href="/hero-image.jpg" as="image">

// 4. 图片优化 - 使用 WebP 格式和响应式图片
<picture>
  <source srcset="hero.webp" type="image/webp">
  <source srcset="hero.jpg" type="image/jpeg">
  <img src="hero.jpg" alt="Hero" loading="lazy">
</picture>

// 5. 字体优化
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<style>
  @font-face {
    font-family: 'MyFont';
    src: url('/fonts/myfont.woff2') format('woff2');
    font-display: swap; /* 避免字体加载阻塞 */
  }
</style>
```

#### 问题1.2: 网络请求瀑布流问题
**现象**:
- 请求串行执行，互相等待
- 关键资源加载被非关键资源阻塞
- Network 面板显示长时间的 Stalled 状态

**常见原因**:
- 浏览器并发请求数限制（HTTP/1.1通常为6个）
- 依赖链过长（A请求依赖B，B依赖C）
- 没有使用HTTP/2或HTTP/3
- DNS查询和TCP连接耗时

**调试方法**:
```javascript
// Chrome DevTools Network 面板
// 1. 查看 Timing 标签页
// 2. 关注这些指标：
//    - Queueing: 请求在队列中等待时间
//    - Stalled: 请求被阻塞时间
//    - DNS Lookup: DNS查询时间
//    - Initial connection: 建立连接时间
//    - Waiting (TTFB): 等待服务器响应时间

// 使用 Resource Timing API 编程监控
performance.getEntriesByType('resource').forEach(resource => {
  console.log({
    name: resource.name,
    duration: resource.duration,
    ttfb: resource.responseStart - resource.requestStart,
    downloadTime: resource.responseEnd - resource.responseStart
  });
});
```

**解决方案**:
```javascript
// 1. 使用 DNS 预解析和预连接
<link rel="dns-prefetch" href="//api.example.com">
<link rel="preconnect" href="https://api.example.com">

// 2. 资源预加载
<link rel="prefetch" href="/next-page.js"> <!-- 低优先级 -->
<link rel="preload" href="/critical.js" as="script"> <!-- 高优先级 -->

// 3. 并行请求优化
// 不好的做法：
async function loadData() {
  const user = await fetch('/api/user');
  const posts = await fetch('/api/posts'); // 等待user完成
  const comments = await fetch('/api/comments'); // 等待posts完成
}

// 好的做法：
async function loadData() {
  const [user, posts, comments] = await Promise.all([
    fetch('/api/user'),
    fetch('/api/posts'),
    fetch('/api/comments')
  ]);
}

// 4. 使用 HTTP/2 Server Push（服务端配置）
// Nginx 配置示例
http2_push /css/critical.css;
http2_push /js/app.js;

// 5. 资源合并（适度使用）
// 小图标使用 SVG Sprite
<svg style="display: none;">
  <symbol id="icon-home" viewBox="0 0 24 24">
    <path d="..."/>
  </symbol>
</svg>
<svg><use href="#icon-home"/></svg>
```

#### 问题1.3: 第三方脚本拖慢页面
**现象**:
- 页面加载被第三方脚本阻塞
- 即使自己的代码优化了，页面仍然慢
- 广告、分析工具、社交媒体插件等影响性能

**常见原因**:
- 同步加载第三方脚本
- 第三方CDN响应慢或不稳定
- 第三方脚本执行耗时长
- 第三方脚本触发额外的网络请求

**调试方法**:
```javascript
// Chrome DevTools Coverage 工具
// 1. 打开 DevTools -> More tools -> Coverage
// 2. 点击录制，刷新页面
// 3. 查看未使用的代码比例

// 使用 Performance Monitor
// 1. DevTools -> More tools -> Performance monitor
// 2. 监控 CPU usage, JS heap size
```

**解决方案**:
```javascript
// 1. 异步或延迟加载第三方脚本
<script async src="https://analytics.example.com/script.js"></script>
<script defer src="https://ads.example.com/script.js"></script>

// 2. 使用 Facade 模式（如 YouTube 视频）
// 先显示缩略图，点击后再加载真实的 iframe
<div class="video-facade"
     style="background-image: url('thumbnail.jpg')"
     onclick="loadYouTubeVideo()">
  <button>播放</button>
</div>

<script>
function loadYouTubeVideo() {
  const iframe = document.createElement('iframe');
  iframe.src = 'https://www.youtube.com/embed/VIDEO_ID';
  // ... 替换 facade
}
</script>

// 3. Self-hosting 第三方资源（适用于稳定的库）
// 下载 Google Analytics 到自己服务器
// 使用 Partytown 在 Web Worker 中运行第三方脚本
import { Partytown } from '@builder.io/partytown/react';

function App() {
  return (
    <>
      <Partytown debug={true} forward={['dataLayer.push']} />
      <script>
        {/* Google Analytics 在 worker 中运行 */}
      </script>
    </>
  );
}

// 4. 条件加载（只在需要时加载）
function loadChatWidget() {
  if (window.scrollY > 1000) {
    const script = document.createElement('script');
    script.src = 'https://chat.example.com/widget.js';
    document.head.appendChild(script);
  }
}

window.addEventListener('scroll', loadChatWidget, { once: true });
```

### 2. 渲染性能问题

#### 问题2.1: 频繁的重排（Reflow）和重绘（Repaint）
**现象**:
- 滚动卡顿，FPS低于60
- 动画不流畅
- DevTools Performance 显示大量 Layout 和 Paint 事件

**常见原因**:
- 读写DOM属性交替进行（Layout Thrashing）
- 使用会触发重排的CSS属性（如width, height, top等）
- 频繁修改DOM结构
- 没有使用CSS transform和opacity做动画

**调试方法**:
```javascript
// 1. Chrome DevTools Rendering 工具
// 打开 DevTools -> More tools -> Rendering
// 启用：
// - Paint flashing: 显示重绘区域
// - Layout Shift Regions: 显示布局偏移
// - Frame Rendering Stats: 显示FPS

// 2. 使用 Performance API 监控长任务
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      console.warn('Long task detected:', entry);
    }
  }
});
observer.observe({ entryTypes: ['longtask'] });

// 3. 检测 Layout Thrashing
let height = element.offsetHeight; // 读取，触发 layout
element.style.height = height + 10 + 'px'; // 写入
height = element.offsetHeight; // 再次读取，再次触发 layout
element.style.height = height + 10 + 'px'; // 再次写入
```

**解决方案**:
```javascript
// 1. 批量读写DOM（使用 FastDOM 或手动批处理）
// 不好的做法：
elements.forEach(el => {
  const height = el.offsetHeight; // 读
  el.style.height = height * 2 + 'px'; // 写
});

// 好的做法：
const heights = elements.map(el => el.offsetHeight); // 批量读
elements.forEach((el, i) => {
  el.style.height = heights[i] * 2 + 'px'; // 批量写
});

// 2. 使用 requestAnimationFrame
function updateLayout() {
  // 在浏览器下一次重绘前执行
  const height = element.offsetHeight;
  requestAnimationFrame(() => {
    element.style.height = height * 2 + 'px';
  });
}

// 3. 使用 CSS Transform 代替定位属性
// 不好：
element.style.left = x + 'px'; // 触发 layout
element.style.top = y + 'px';

// 好：
element.style.transform = `translate(${x}px, ${y}px)`; // 只触发 composite

// 4. 使用 will-change 提示浏览器优化
.moving-element {
  will-change: transform; /* 创建新的合成层 */
}

// 5. 虚拟化长列表
import { FixedSizeList } from 'react-window';

function VirtualList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
    >
      {({ index, style }) => (
        <div style={style}>{items[index]}</div>
      )}
    </FixedSizeList>
  );
}

// 6. 离屏渲染复杂DOM操作
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const item = document.createElement('div');
  item.textContent = `Item ${i}`;
  fragment.appendChild(item);
}
container.appendChild(fragment); // 只触发一次 reflow
```

#### 问题2.2: JavaScript执行时间过长
**现象**:
- 主线程阻塞，页面无响应
- Performance 面板显示黄色或红色的长任务
- 用户输入延迟，FID指标差

**常见原因**:
- 复杂计算在主线程执行
- 大量数据处理没有分片
- 同步的for循环处理大数组
- 没有使用防抖节流处理高频事件

**调试方法**:
```javascript
// 1. 使用 Chrome DevTools Performance Profiler
// 录制 -> 查看 Call Tree 和 Bottom-Up
// 找出耗时最长的函数

// 2. 使用 console.time 测量性能
console.time('heavyTask');
heavyComputation();
console.timeEnd('heavyTask');

// 3. User Timing API
performance.mark('start-heavy-task');
heavyComputation();
performance.mark('end-heavy-task');
performance.measure('heavy-task', 'start-heavy-task', 'end-heavy-task');

const measures = performance.getEntriesByType('measure');
console.log(measures[0].duration);
```

**解决方案**:
```javascript
// 1. 使用 Web Worker 处理重计算
// worker.js
self.addEventListener('message', (e) => {
  const result = heavyComputation(e.data);
  self.postMessage(result);
});

// main.js
const worker = new Worker('worker.js');
worker.postMessage(data);
worker.addEventListener('message', (e) => {
  console.log('Result:', e.data);
});

// 2. 时间切片（Time Slicing）
function processLargeArray(array, chunkSize = 100) {
  let index = 0;

  function processChunk() {
    const endIndex = Math.min(index + chunkSize, array.length);

    for (; index < endIndex; index++) {
      // 处理 array[index]
      processItem(array[index]);
    }

    if (index < array.length) {
      // 让出主线程，下一帧继续处理
      requestIdleCallback(processChunk, { timeout: 1000 });
    }
  }

  processChunk();
}

// 3. 使用 scheduler API (实验性)
if ('scheduler' in window) {
  scheduler.postTask(() => {
    heavyComputation();
  }, { priority: 'background' });
}

// 4. 防抖和节流
// 防抖：最后一次触发后才执行
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

const handleSearch = debounce((query) => {
  fetch(`/api/search?q=${query}`);
}, 300);

// 节流：固定时间间隔执行
function throttle(fn, delay) {
  let last = 0;
  return function(...args) {
    const now = Date.now();
    if (now - last >= delay) {
      last = now;
      fn.apply(this, args);
    }
  };
}

const handleScroll = throttle(() => {
  console.log('Scroll position:', window.scrollY);
}, 100);

// 5. 优化React渲染
import { memo, useMemo, useCallback } from 'react';

// memo 避免不必要的重新渲染
const ExpensiveComponent = memo(({ data }) => {
  return <div>{/* 渲染 data */}</div>;
});

// useMemo 缓存计算结果
function DataList({ items, filter }) {
  const filteredItems = useMemo(() => {
    return items.filter(item => item.includes(filter));
  }, [items, filter]);

  return <div>{/* 渲染 filteredItems */}</div>;
}

// useCallback 缓存函数引用
function Parent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <Child onClick={handleClick} />;
}
```

#### 问题2.3: 内存泄漏
**现象**:
- 页面使用时间越长越慢
- 内存占用持续增长
- 最终导致标签页崩溃

**常见原因**:
- 事件监听器没有移除
- 定时器没有清除
- 闭包引用了大对象
- DOM引用没有释放
- 全局变量累积

**调试方法**:
```javascript
// 1. Chrome DevTools Memory Profiler
// Memory 面板 -> Heap snapshot
// 拍摄多个快照，比较内存增长

// 2. Performance Monitor
// 查看 JS heap size 是否持续增长

// 3. 使用 Detached DOM tree 检测
// Memory -> Heap snapshot
// 搜索 "Detached" 找到未被GC的DOM

// 4. 代码检测内存泄漏
let memoryLeakTest = [];
setInterval(() => {
  memoryLeakTest.push(new Array(1000000)); // 故意泄漏
  console.log('Memory:', performance.memory.usedJSHeapSize);
}, 1000);
```

**解决方案**:
```javascript
// 1. 正确清理事件监听器
class Component {
  constructor() {
    this.handleClick = this.handleClick.bind(this);
  }

  componentDidMount() {
    window.addEventListener('click', this.handleClick);
  }

  componentWillUnmount() {
    window.removeEventListener('click', this.handleClick); // 清理
  }

  handleClick() {
    console.log('clicked');
  }
}

// React hooks 版本
function Component() {
  useEffect(() => {
    const handleClick = () => console.log('clicked');
    window.addEventListener('click', handleClick);

    return () => {
      window.removeEventListener('click', handleClick); // 清理
    };
  }, []);
}

// 2. 清理定时器
function Component() {
  useEffect(() => {
    const timer = setInterval(() => {
      console.log('tick');
    }, 1000);

    return () => clearInterval(timer); // 清理
  }, []);
}

// 3. 避免闭包陷阱
// 不好：
function createClosure() {
  const largeData = new Array(1000000);
  return function() {
    console.log(largeData[0]); // 保持对 largeData 的引用
  };
}

// 好：
function createClosure() {
  const largeData = new Array(1000000);
  const firstItem = largeData[0];
  return function() {
    console.log(firstItem); // 只引用需要的数据
  };
}

// 4. WeakMap 和 WeakSet 避免内存泄漏
// 使用 WeakMap 存储DOM相关数据
const domData = new WeakMap();
const element = document.getElementById('myElement');
domData.set(element, { count: 0 });
// 当 element 被移除时，相关数据会被自动GC

// 5. 及时释放引用
let cache = {};
function addToCache(key, value) {
  cache[key] = value;
}

function clearOldCache() {
  const now = Date.now();
  Object.keys(cache).forEach(key => {
    if (cache[key].timestamp < now - 3600000) {
      delete cache[key]; // 清理1小时前的缓存
    }
  });
}
```

### 3. 资源优化问题

#### 问题3.1: 图片资源未优化
**现象**:
- 大尺寸图片加载慢
- 移动端加载桌面图片
- LCP指标差

**解决方案**:
```html
<!-- 1. 响应式图片 -->
<img
  srcset="small.jpg 480w,
          medium.jpg 800w,
          large.jpg 1200w"
  sizes="(max-width: 600px) 480px,
         (max-width: 1000px) 800px,
         1200px"
  src="fallback.jpg"
  alt="Responsive image"
>

<!-- 2. 现代图片格式 -->
<picture>
  <source type="image/avif" srcset="image.avif">
  <source type="image/webp" srcset="image.webp">
  <img src="image.jpg" alt="Modern format image">
</picture>

<!-- 3. 懒加载 -->
<img src="image.jpg" loading="lazy" alt="Lazy loaded image">

<!-- 4. 使用 CDN 和图片处理服务 -->
<img src="https://cdn.example.com/image.jpg?w=800&q=80&format=webp">
```

```javascript
// 5. Intersection Observer 实现自定义懒加载
const imageObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      imageObserver.unobserve(img);
    }
  });
});

document.querySelectorAll('img[data-src]').forEach(img => {
  imageObserver.observe(img);
});

// 6. 预加载关键图片
const link = document.createElement('link');
link.rel = 'preload';
link.as = 'image';
link.href = 'hero-image.jpg';
document.head.appendChild(link);
```

#### 问题3.2: CSS和JavaScript文件过大
**解决方案**:
```javascript
// 1. Tree Shaking（Webpack配置）
// webpack.config.js
module.exports = {
  mode: 'production',
  optimization: {
    usedExports: true,
    minimize: true,
  },
};

// 使用ES6模块，避免导入整个库
// 不好：
import _ from 'lodash'; // 导入整个 lodash
const result = _.debounce(fn, 300);

// 好：
import debounce from 'lodash/debounce'; // 只导入需要的函数

// 2. 代码压缩
// 使用 Terser 压缩 JavaScript
// 使用 cssnano 压缩 CSS

// 3. Gzip/Brotli 压缩（服务器配置）
// Nginx 配置
gzip on;
gzip_types text/css application/javascript;
brotli on;
brotli_types text/css application/javascript;

// 4. 移除未使用的 CSS
// 使用 PurgeCSS
const purgecss = require('@fullhuman/postcss-purgecss');

module.exports = {
  plugins: [
    purgecss({
      content: ['./src/**/*.html', './src/**/*.js'],
    }),
  ],
};

// 5. Critical CSS
// 提取首屏关键CSS内联
<style>
  /* Critical CSS for above-the-fold content */
  .hero { ... }
  .nav { ... }
</style>
<link rel="preload" href="main.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
```

---

## 调试工具与方法

### 1. Chrome DevTools

#### Performance 面板
```javascript
// 性能分析步骤：
// 1. 打开 DevTools -> Performance
// 2. 点击 Record (⏺)
// 3. 执行需要分析的操作
// 4. 停止录制
// 5. 分析结果

// 查看指标：
// - FPS: 帧率图，绿色高度越高越流畅
// - Main: 主线程活动，黄色是JavaScript执行，紫色是渲染
// - Network: 网络请求
// - Frames: 每一帧的详细信息

// 使用 Performance API 编程式监控
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(entry.name, entry.duration);
  }
});
observer.observe({ entryTypes: ['measure', 'navigation'] });
```

#### Network 面板
```javascript
// 关键指标：
// - DOMContentLoaded: DOM解析完成（蓝线）
// - Load: 所有资源加载完成（红线）
// - Finish: 所有请求完成
// - 每个请求的 Timing 信息：
//   - Queueing: 排队时间
//   - Stalled: 阻塞时间
//   - DNS Lookup: DNS查询
//   - Initial connection: TCP连接
//   - SSL: TLS握手
//   - Request sent: 发送请求
//   - Waiting (TTFB): 等待首字节
//   - Content Download: 下载内容

// 过滤技巧：
// - 输入 "larger-than:1M" 找出大于1MB的资源
// - 输入 "is:running" 查看正在进行的请求
// - 按住 Shift 悬停查看请求发起者
```

#### Memory 面板
```javascript
// 1. Heap Snapshot（堆快照）
// 步骤：
// - 拍摄初始快照
// - 执行操作
// - 拍摄第二个快照
// - 对比查看内存增长

// 2. Allocation Timeline（分配时间线）
// 查看内存分配随时间的变化

// 3. Allocation Sampling（分配采样）
// 查找内存分配热点
```

#### Coverage 面板
```javascript
// 查找未使用的代码
// 1. 打开 DevTools -> More tools -> Coverage
// 2. 点击 Record
// 3. 使用应用
// 4. 查看代码使用率
// 红色部分是未使用的代码
```

### 2. Lighthouse

```bash
# 命令行使用
npm install -g lighthouse
lighthouse https://example.com --view

# CI集成
lighthouse https://example.com --output=json --output-path=./report.json

# 编程式使用
const lighthouse = require('lighthouse');
const chromeLauncher = require('chrome-launcher');

(async () => {
  const chrome = await chromeLauncher.launch({chromeFlags: ['--headless']});
  const options = {port: chrome.port};
  const runnerResult = await lighthouse('https://example.com', options);

  console.log('Performance score:', runnerResult.lhr.categories.performance.score * 100);

  await chrome.kill();
})();
```

### 3. 性能监控工具

```javascript
// 1. Web Vitals 库
import {getCLS, getFID, getFCP, getLCP, getTTFB} from 'web-vitals';

getCLS(console.log);
getFID(console.log);
getFCP(console.log);
getLCP(console.log);
getTTFB(console.log);

// 2. 自定义性能监控
class PerformanceMonitor {
  constructor() {
    this.metrics = {};
  }

  mark(name) {
    performance.mark(name);
  }

  measure(name, startMark, endMark) {
    performance.measure(name, startMark, endMark);
    const measure = performance.getEntriesByName(name)[0];
    this.metrics[name] = measure.duration;
    return measure.duration;
  }

  sendToAnalytics() {
    // 发送到分析服务
    fetch('/analytics', {
      method: 'POST',
      body: JSON.stringify(this.metrics)
    });
  }
}

const monitor = new PerformanceMonitor();
monitor.mark('start-api-call');
await fetch('/api/data');
monitor.mark('end-api-call');
monitor.measure('api-call-duration', 'start-api-call', 'end-api-call');

// 3. Long Task Observer
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn('Long task detected:', {
      duration: entry.duration,
      startTime: entry.startTime
    });

    // 发送到监控服务
    sendToMonitoring({
      type: 'long-task',
      duration: entry.duration
    });
  }
});
observer.observe({entryTypes: ['longtask']});
```

---

## 解决方案详解

### 1. 缓存策略

```javascript
// 1. HTTP缓存（服务器配置）
// Nginx 示例
location ~* \.(js|css|png|jpg|jpeg|gif|svg|woff2)$ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}

location /api/ {
  add_header Cache-Control "no-cache";
}

// 2. Service Worker 缓存
// sw.js
const CACHE_NAME = 'v1';
const urlsToCache = [
  '/',
  '/styles/main.css',
  '/script/main.js'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(urlsToCache))
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        // 缓存命中，返回缓存
        if (response) {
          return response;
        }
        // 否则发起网络请求
        return fetch(event.request).then((response) => {
          // 缓存新资源
          return caches.open(CACHE_NAME).then((cache) => {
            cache.put(event.request, response.clone());
            return response;
          });
        });
      })
  );
});

// 3. 浏览器存储缓存
// LocalStorage（同步，最大5MB）
localStorage.setItem('userData', JSON.stringify(data));
const userData = JSON.parse(localStorage.getItem('userData'));

// IndexedDB（异步，大容量）
const request = indexedDB.open('MyDatabase', 1);
request.onsuccess = (event) => {
  const db = event.target.result;
  const transaction = db.transaction(['users'], 'readwrite');
  const objectStore = transaction.objectStore('users');
  objectStore.add({ id: 1, name: 'John' });
};

// 4. Memory Cache（内存缓存）
const cache = new Map();

function memoize(fn) {
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const expensiveFunction = memoize((n) => {
  // 复杂计算
  return n * 2;
});
```

### 2. 预加载策略

```html
<!-- 1. DNS Prefetch -->
<link rel="dns-prefetch" href="//api.example.com">

<!-- 2. Preconnect -->
<link rel="preconnect" href="https://fonts.googleapis.com">

<!-- 3. Prefetch（低优先级，空闲时加载） -->
<link rel="prefetch" href="/next-page.js">

<!-- 4. Preload（高优先级，立即加载） -->
<link rel="preload" href="/critical.css" as="style">
<link rel="preload" href="/hero.jpg" as="image">
<link rel="preload" href="/font.woff2" as="font" type="font/woff2" crossorigin>

<!-- 5. Modulepreload（预加载ES模块） -->
<link rel="modulepreload" href="/app.js">
```

```javascript
// 6. 动态预加载
function prefetchPage(url) {
  const link = document.createElement('link');
  link.rel = 'prefetch';
  link.href = url;
  document.head.appendChild(link);
}

// 鼠标悬停时预加载
document.querySelectorAll('a').forEach(link => {
  link.addEventListener('mouseenter', () => {
    prefetchPage(link.href);
  });
});

// 7. Quicklink（自动预加载可见链接）
import quicklink from 'quicklink';
quicklink.listen();
```

### 3. 构建优化

```javascript
// webpack.config.js

module.exports = {
  // 1. 代码分割
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // 将第三方库分离
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        },
        // 公共代码分离
        common: {
          minChunks: 2,
          name: 'common',
          priority: 5
        }
      }
    },
    // 运行时代码分离
    runtimeChunk: 'single'
  },

  // 2. 生产环境优化
  mode: 'production',

  // 3. Source Map（生产环境使用 hidden-source-map）
  devtool: 'hidden-source-map',

  // 4. 压缩
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true, // 移除 console
          }
        }
      }),
      new CssMinimizerPlugin()
    ]
  },

  // 5. Tree Shaking
  optimization: {
    usedExports: true,
    sideEffects: false
  },

  // 6. 分析包体积
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static'
    })
  ]
};

// 7. 动态导入
const LazyComponent = React.lazy(() => import('./LazyComponent'));

// 8. 路由级分割
const routes = [
  {
    path: '/',
    component: () => import('./pages/Home')
  },
  {
    path: '/about',
    component: () => import('./pages/About')
  }
];
```

---

## 实战案例

### 案例1: 电商首页优化

**初始问题**:
- LCP: 4.5秒
- FCP: 2.8秒
- TBT: 1200ms
- Bundle大小: 2.5MB

**优化步骤**:

```javascript
// 1. 代码分割 - 减少初始包大小
// 之前：所有代码在一个bundle
// 之后：按路由分割
const ProductList = lazy(() => import('./pages/ProductList'));
const ProductDetail = lazy(() => import('./pages/ProductDetail'));
const Cart = lazy(() => import('./pages/Cart'));

// 2. 图片优化
// 使用 next/image 自动优化
import Image from 'next/image';

<Image
  src="/product.jpg"
  width={500}
  height={500}
  loading="lazy"
  placeholder="blur"
/>

// 3. 预加载关键资源
<link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/api/featured-products" as="fetch" crossorigin>

// 4. 服务端渲染首屏
// Next.js getServerSideProps
export async function getServerSideProps() {
  const products = await fetch('https://api.example.com/products');
  return { props: { products } };
}

// 5. 使用 CDN
// next.config.js
module.exports = {
  images: {
    domains: ['cdn.example.com'],
  },
  assetPrefix: 'https://cdn.example.com',
};

// 6. 减少JavaScript执行
// 移除未使用的依赖
// 之前：import moment from 'moment'; (229KB)
// 之后：import { format } from 'date-fns'; (13KB)

// 7. 实现虚拟滚动
import { FixedSizeList } from 'react-window';

function ProductGrid({ products }) {
  return (
    <FixedSizeList
      height={800}
      itemCount={products.length}
      itemSize={250}
      width="100%"
    >
      {({ index, style }) => (
        <ProductCard style={style} product={products[index]} />
      )}
    </FixedSizeList>
  );
}
```

**优化结果**:
- LCP: 1.8秒 ⬇️ 60%
- FCP: 0.9秒 ⬇️ 68%
- TBT: 180ms ⬇️ 85%
- Bundle大小: 450KB ⬇️ 82%

### 案例2: 单页应用（SPA）性能优化

**问题**:
- 路由切换慢
- 内存占用持续增长
- 列表渲染卡顿

**解决方案**:

```javascript
// 1. 路由预加载
import { useEffect } from 'react';
import { useRouter } from 'next/router';

function usePrefetch(href) {
  const router = useRouter();

  useEffect(() => {
    router.prefetch(href);
  }, [router, href]);
}

// 使用
function NavLink({ href, children }) {
  usePrefetch(href);
  return <Link href={href}>{children}</Link>;
}

// 2. 组件缓存（React）
import { memo } from 'react';

const ExpensiveList = memo(({ items }) => {
  return items.map(item => <ExpensiveItem key={item.id} {...item} />);
}, (prevProps, nextProps) => {
  // 自定义比较逻辑
  return prevProps.items.length === nextProps.items.length;
});

// 3. 状态管理优化
// 使用 Context 的细粒度分离
const UserContext = createContext();
const CartContext = createContext();
const ThemeContext = createContext();

// 而不是一个大的 AppContext

// 4. 清理副作用避免内存泄漏
function Component() {
  useEffect(() => {
    const subscription = someObservable.subscribe();

    return () => {
      subscription.unsubscribe(); // 清理
    };
  }, []);
}

// 5. 使用 Intersection Observer 懒加载组件
function LazySection() {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setIsVisible(true);
        observer.disconnect();
      }
    });

    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={ref}>
      {isVisible ? <HeavyComponent /> : <Placeholder />}
    </div>
  );
}

// 6. 优化列表渲染
// 使用 key 优化
// 不好：
items.map((item, index) => <Item key={index} {...item} />)

// 好：
items.map(item => <Item key={item.id} {...item} />)

// 7. 防止过度渲染
function ParentComponent() {
  const [count, setCount] = useState(0);

  // 使用 useCallback 避免子组件重新渲染
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <ChildComponent onClick={handleClick} />
    </>
  );
}

const ChildComponent = memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click</button>;
});
```

---

## 最佳实践

### 1. 性能预算

```javascript
// 设置性能预算
const performanceBudget = {
  FCP: 1800,      // First Contentful Paint < 1.8s
  LCP: 2500,      // Largest Contentful Paint < 2.5s
  FID: 100,       // First Input Delay < 100ms
  CLS: 0.1,       // Cumulative Layout Shift < 0.1
  TBT: 300,       // Total Blocking Time < 300ms
  bundleSize: {
    js: 500 * 1024,    // JS < 500KB
    css: 100 * 1024,   // CSS < 100KB
    images: 1024 * 1024 // Images < 1MB
  }
};

// Webpack 性能预算配置
module.exports = {
  performance: {
    maxEntrypointSize: 500000,
    maxAssetSize: 300000,
    hints: 'error'
  }
};

// Lighthouse CI 配置
module.exports = {
  ci: {
    assert: {
      assertions: {
        'first-contentful-paint': ['error', {maxNumericValue: 1800}],
        'largest-contentful-paint': ['error', {maxNumericValue: 2500}],
        'cumulative-layout-shift': ['error', {maxNumericValue: 0.1}],
      }
    }
  }
};
```

### 2. 渐进式增强

```javascript
// 1. 检测功能支持
if ('IntersectionObserver' in window) {
  // 使用 Intersection Observer
} else {
  // 降级方案
  loadAllImages();
}

// 2. 现代/传统 Bundle
// 使用 module/nomodule 模式
<script type="module" src="modern-bundle.js"></script>
<script nomodule src="legacy-bundle.js"></script>

// 3. 渐进式图片加载
<img
  src="placeholder.jpg"        // 小占位图
  data-src="full-image.jpg"    // 完整图片
  loading="lazy"
/>

// 4. CSS 渐进式增强
.container {
  display: block; /* 降级方案 */
}

@supports (display: grid) {
  .container {
    display: grid; /* 现代布局 */
  }
}
```

### 3. 监控和告警

```javascript
// 1. 实时性能监控
import { getCLS, getFID, getLCP } from 'web-vitals';

function sendToAnalytics(metric) {
  const body = JSON.stringify(metric);

  if (navigator.sendBeacon) {
    navigator.sendBeacon('/analytics', body);
  } else {
    fetch('/analytics', { body, method: 'POST', keepalive: true });
  }
}

getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getLCP(sendToAnalytics);

// 2. 错误监控
window.addEventListener('error', (event) => {
  sendToAnalytics({
    type: 'error',
    message: event.message,
    stack: event.error.stack,
    url: window.location.href
  });
});

// 3. 资源加载监控
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 3000) {
      sendToAnalytics({
        type: 'slow-resource',
        name: entry.name,
        duration: entry.duration
      });
    }
  }
});
observer.observe({ entryTypes: ['resource'] });

// 4. 自定义指标
performance.mark('feature-load-start');
await loadFeature();
performance.mark('feature-load-end');
performance.measure('feature-load', 'feature-load-start', 'feature-load-end');

const measure = performance.getEntriesByName('feature-load')[0];
sendToAnalytics({
  type: 'custom-metric',
  name: 'feature-load',
  duration: measure.duration
});
```

### 4. 持续优化流程

```bash
# 1. 本地性能测试
npm run build
lighthouse http://localhost:3000 --view

# 2. CI/CD 集成
# .github/workflows/performance.yml
name: Performance Tests
on: [pull_request]
jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run Lighthouse
        uses: treosh/lighthouse-ci-action@v8
        with:
          urls: |
            https://staging.example.com
          uploadArtifacts: true

# 3. 性能回归检测
# 比较PR前后的性能指标

# 4. 定期审计
# 每周运行一次完整的性能审计
```

### 5. 团队协作

```javascript
// 1. 性能检查清单
/*
Performance Checklist:
☐ 图片已优化（WebP/AVIF格式）
☐ 代码已分割
☐ 使用了懒加载
☐ CSS已压缩并移除未使用代码
☐ JavaScript已压缩和Tree Shaking
☐ 配置了缓存策略
☐ 使用了CDN
☐ 关键资源已预加载
☐ 第三方脚本已优化
☐ 性能预算未超标
*/

// 2. 代码审查关注点
// 在 PR 中检查：
// - 新增的依赖大小
// - 是否有性能退化
// - 是否有内存泄漏风险

// 3. 性能文化
// - 将性能指标纳入团队KPI
// - 定期分享性能优化案例
// - 建立性能优化知识库
```

---

## 总结

前端性能优化是一个持续的过程，需要：

1. **建立基准**: 使用工具测量当前性能
2. **设定目标**: 制定性能预算和优化目标
3. **系统优化**: 从加载、渲染、交互多方面优化
4. **持续监控**: 实时监控生产环境性能
5. **团队协作**: 建立性能优化文化

关键原则：
- **测量优先**: 不要凭感觉优化，用数据说话
- **用户为中心**: 关注用户实际体验的指标
- **渐进增强**: 确保基本功能可用，逐步增强体验
- **自动化**: 将性能检查集成到CI/CD流程
- **持续改进**: 性能优化永无止境

记住：**过早优化是万恶之源，但忽视性能同样不可取。在开发过程中保持性能意识，在发现问题时及时优化，是最好的实践。**
