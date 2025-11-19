# 前端性能优化术语表

## 性能指标

### FCP (First Contentful Paint)
**首次内容绘制**
- 浏览器首次渲染任何文本、图像、非白色canvas或SVG的时间点
- 良好: < 1.8秒
- 需要改进: 1.8-3秒
- 差: > 3秒

### LCP (Largest Contentful Paint)
**最大内容绘制**
- 视口中最大的内容元素（图像、视频或文本块）完成渲染的时间
- 良好: < 2.5秒
- 需要改进: 2.5-4秒
- 差: > 4秒

### FID (First Input Delay)
**首次输入延迟**
- 用户首次与页面交互到浏览器实际响应的时间
- 良好: < 100毫秒
- 需要改进: 100-300毫秒
- 差: > 300毫秒

### CLS (Cumulative Layout Shift)
**累积布局偏移**
- 测量视觉稳定性，量化页面元素意外移动的程度
- 良好: < 0.1
- 需要改进: 0.1-0.25
- 差: > 0.25

### TTI (Time to Interactive)
**可交互时间**
- 页面完全可交互的时间点
- 良好: < 3.8秒

### TBT (Total Blocking Time)
**总阻塞时间**
- FCP和TTI之间主线程被阻塞的总时间
- 良好: < 300毫秒

### TTFB (Time to First Byte)
**首字节时间**
- 浏览器接收到服务器响应的第一个字节的时间
- 良好: < 600毫秒

### SI (Speed Index)
**速度指数**
- 页面内容可见填充的速度
- 良好: < 3.4秒

---

## 优化技术

### Code Splitting
**代码分割**
- 将应用代码拆分成多个bundle，按需加载
- 减少初始加载的JavaScript大小

### Lazy Loading
**懒加载**
- 延迟加载非关键资源，直到需要时才加载
- 常用于图片、组件、路由

### Tree Shaking
**摇树优化**
- 移除JavaScript中未使用的代码
- 需要ES6模块语法支持

### Minification
**压缩**
- 移除代码中的空格、注释、缩短变量名
- 减小文件体积

### Bundling
**打包**
- 将多个文件合并成少数几个文件
- 减少HTTP请求数量

### Critical CSS
**关键CSS**
- 首屏渲染所需的最小CSS集合
- 内联到HTML中，其余CSS异步加载

### Prefetch
**预取**
- 低优先级地预加载未来可能需要的资源
- 在浏览器空闲时进行

### Preload
**预加载**
- 高优先级地预加载当前页面需要的资源
- 立即开始加载

### Preconnect
**预连接**
- 提前建立与目标域的连接（DNS、TCP、TLS）
- 减少后续请求的延迟

### DNS Prefetch
**DNS预解析**
- 提前解析域名
- 减少DNS查询时间

---

## 性能问题

### Reflow (Layout)
**重排/回流**
- 浏览器重新计算元素的几何属性
- 非常耗性能的操作
- 触发因素: 改变元素尺寸、位置、内容

### Repaint
**重绘**
- 浏览器重新绘制元素的视觉样式
- 不涉及布局改变
- 比重排性能好，但仍有开销

### Layout Thrashing
**布局抖动**
- 快速连续地读写DOM导致的强制同步布局
- 严重影响性能

### Memory Leak
**内存泄漏**
- 应用持续占用内存，无法被垃圾回收
- 最终导致性能下降或崩溃

### Long Task
**长任务**
- 执行时间超过50毫秒的任务
- 阻塞主线程，影响交互响应

### JavaScript Heap
**JavaScript堆**
- JavaScript运行时的内存空间
- 包含对象、闭包等

### Main Thread
**主线程**
- 浏览器处理用户事件、JavaScript执行、样式计算、布局等的线程
- 是性能瓶颈所在

---

## 缓存策略

### Cache-Control
**缓存控制**
- HTTP头，控制资源的缓存行为
- 常用值: public, private, no-cache, no-store, max-age

### ETag
**实体标签**
- 资源的唯一标识符
- 用于验证缓存是否有效

### Service Worker
**服务工作线程**
- 运行在浏览器后台的脚本
- 可以拦截网络请求，实现离线缓存

### HTTP/2 Server Push
**HTTP/2服务器推送**
- 服务器主动推送资源到客户端
- 无需等待客户端请求

---

## 图片优化

### WebP
- Google开发的现代图片格式
- 比JPEG/PNG体积小25-35%

### AVIF
- 新一代图片格式
- 压缩率优于WebP

### Responsive Images
**响应式图片**
- 根据设备屏幕尺寸加载不同大小的图片
- 使用srcset和sizes属性

### Image Sprites
**图片精灵**
- 将多个小图标合并到一张图片
- 减少HTTP请求

---

## 网络优化

### CDN (Content Delivery Network)
**内容分发网络**
- 将资源部署到全球多个节点
- 用户从最近的节点获取资源

### Compression
**压缩**
- Gzip: 传统压缩算法，兼容性好
- Brotli: 新一代压缩算法，压缩率更高

### HTTP/2
- 支持多路复用、头部压缩、服务器推送
- 提升网络传输效率

### HTTP/3
- 基于QUIC协议
- 进一步提升性能和安全性

---

## 调试工具

### Chrome DevTools
- 浏览器内置的开发者工具
- 包含Performance、Network、Memory等面板

### Lighthouse
- Google的自动化性能审计工具
- 提供性能、可访问性、SEO等评分

### WebPageTest
- 在线性能测试工具
- 提供详细的加载瀑布图和性能报告

### Bundle Analyzer
- 分析Webpack打包结果的可视化工具
- 查看各模块的大小占比

---

## 渲染优化

### Virtual DOM
**虚拟DOM**
- 内存中的DOM表示
- 通过diff算法减少实际DOM操作

### Virtual Scrolling
**虚拟滚动**
- 只渲染可见区域的元素
- 处理大列表的性能优化技术

### Debounce
**防抖**
- 延迟执行，只执行最后一次
- 适用于搜索输入等场景

### Throttle
**节流**
- 固定时间间隔执行
- 适用于滚动、窗口调整等高频事件

### RequestAnimationFrame
**请求动画帧**
- 在浏览器下一次重绘前执行回调
- 实现流畅动画的最佳方式

### Web Worker
**Web工作线程**
- 在后台线程运行JavaScript
- 不阻塞主线程

### Composite Layer
**合成层**
- GPU加速的渲染层
- 使用transform和opacity触发

---

## 度量和监控

### RUM (Real User Monitoring)
**真实用户监控**
- 收集真实用户的性能数据
- 反映实际使用场景

### Synthetic Monitoring
**合成监控**
- 模拟用户访问进行监控
- 可以预设测试场景

### Performance Budget
**性能预算**
- 设定性能指标的阈值
- 超过预算时发出警告

### Core Web Vitals
**核心Web指标**
- Google定义的关键用户体验指标
- 包括LCP、FID、CLS

---

## 其他术语

### Above the Fold
**首屏**
- 页面加载后无需滚动即可看到的区域

### Critical Rendering Path
**关键渲染路径**
- 浏览器将HTML、CSS、JavaScript转换为屏幕像素的步骤

### Render Blocking
**渲染阻塞**
- 阻止页面渲染的资源（如CSS、同步JavaScript）

### Parser Blocking
**解析阻塞**
- 阻止HTML解析的资源（如同步JavaScript）

### Bandwidth
**带宽**
- 网络连接的数据传输速率

### Latency
**延迟**
- 数据从源到目的地的传输时间

### Round Trip Time (RTT)
**往返时间**
- 请求发送到接收响应的时间

### Waterfall Chart
**瀑布图**
- 可视化显示资源加载时序的图表

### Jank
**卡顿**
- 页面渲染不流畅，FPS低于60

### Frame Rate
**帧率**
- 每秒渲染的帧数
- 60 FPS是流畅体验的目标
