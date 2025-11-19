# Web Performance 前端性能优化研究

## 简介

本目录包含前端Web性能优化的系统性研究文档，涵盖性能调试常见问题、工具使用、解决方案和最佳实践。

## 文档列表

### 1. [网页性能调试问题与解决方案](./web-performance-debugging.md)

全面研究前端开发者在优化网页性能时遇到的各种问题及其解决方案。

**主要内容**:
- 加载性能问题（首屏加载、网络请求、第三方脚本）
- 渲染性能问题（重排重绘、JavaScript执行、内存泄漏）
- 资源优化问题（图片、CSS、JavaScript）
- 调试工具详解（Chrome DevTools、Lighthouse等）
- 实战案例和最佳实践

### 2. [Chrome DevTools 火焰图深度指南](./chrome-devtools-flame-chart-guide.md)

深入讲解如何使用 Chrome DevTools 的火焰图（Flame Chart）进行性能分析。

**主要内容**:
- 火焰图基础知识和阅读方法
- Chrome DevTools Performance 面板详解
- 识别常见性能模式（长任务、布局抖动、强制同步布局）
- **真实优化案例**：
  - Deepdesk 文本差异算法优化（1850ms → 85ms）
  - isOnScreen 检查优化（FPS 12 → 60）
  - React 列表渲染优化（2500ms → 95ms）
  - 事件处理器防抖优化
- 2024 年新功能（CPU 节流校准、20x 减速）
- 高级调试技巧和最佳实践

### 3. [React 和 Next.js 性能优化指南](./react-nextjs-performance-optimization.md)

专注于 React 和 Next.js 应用的性能优化技巧和工具。

**主要内容**:
- **React DevTools Profiler** 详细使用指南
  - Flamegraph 和 Ranked 视图解析
  - 识别渲染瓶颈和不必要的重新渲染
- **React 性能优化技巧**：
  - React.memo、useMemo、useCallback 使用场景
  - 虚拟化长列表（react-window）
  - 代码分割与懒加载
  - Context 优化策略
  - 防抖节流处理
- **Next.js 专项优化**：
  - next/image 图片优化
  - next/font 字体优化
  - next/script 第三方脚本优化
  - 服务端组件 vs 客户端组件
  - 路由预取策略
- **Bundle 分析与优化**：
  - @next/bundle-analyzer 使用
  - 依赖优化（lodash → lodash-es, moment → date-fns）
  - webpack 配置优化
- 电商产品列表优化实战案例

### 4. [Chrome DevTools 完整性能调试指南](./chrome-devtools-complete-guide.md)

Chrome DevTools 所有性能相关面板的详细使用说明。

**主要内容**:
- **Performance 面板深度使用**：
  - 录制模式和设置详解
  - Overview、Main 线程、Summary、Bottom-Up、Call Tree 分析
  - 火焰图颜色编码和事件类型
- **Network 面板详解**：
  - 请求 Timing 详解（Queueing、Stalled、DNS、TTFB）
  - 瀑布图分析和优化
  - 请求阻塞分析
  - HAR 文件导出
- **Memory 面板使用**：
  - Heap Snapshot 堆快照对比
  - Allocation Timeline 内存分配时间线
  - 内存泄漏模式识别（事件监听器、分离 DOM、闭包陷阱）
- **Coverage 面板**：未使用代码检测
- **Rendering 面板**：Paint flashing、Layout Shift、Layer borders
- **Layers 面板**：合成层 3D 视图
- **Performance Monitor**：实时性能指标监控
- **2024 新功能**：CPU 节流校准、Performance Insights
- 实用技巧和快捷键

### 5. [性能术语表](./GLOSSARY.md)

完整的前端性能优化术语中英文对照表，包含 90+ 专业术语的详细解释。

## 核心性能指标

- **FCP** (First Contentful Paint): 首次内容绘制
- **LCP** (Largest Contentful Paint): 最大内容绘制
- **FID** (First Input Delay): 首次输入延迟
- **CLS** (Cumulative Layout Shift): 累积布局偏移
- **TTI** (Time to Interactive): 可交互时间
- **TBT** (Total Blocking Time): 总阻塞时间

## 快速开始

### 性能检测工具

```bash
# 安装 Lighthouse
npm install -g lighthouse

# 运行性能测试
lighthouse https://your-website.com --view

# 使用 Web Vitals
npm install web-vitals
```

### 基本优化清单

- [ ] 启用 Gzip/Brotli 压缩
- [ ] 配置浏览器缓存
- [ ] 优化图片（WebP/AVIF格式）
- [ ] 实现代码分割
- [ ] 使用 CDN
- [ ] 懒加载非关键资源
- [ ] 移除未使用的代码
- [ ] 压缩 CSS 和 JavaScript

## 常用调试命令

```bash
# 分析包体积
npm install --save-dev webpack-bundle-analyzer

# 检查未使用的依赖
npx depcheck

# 运行性能审计
npm run build
npx lighthouse http://localhost:3000
```

## 学习路径

### 初学者
1. 先阅读 [网页性能调试问题与解决方案](./web-performance-debugging.md)
2. 学习 [Chrome DevTools 完整性能调试指南](./chrome-devtools-complete-guide.md) 的基础部分
3. 查阅 [性能术语表](./GLOSSARY.md) 理解专业术语

### 中级开发者
1. 深入学习 [Chrome DevTools 火焰图深度指南](./chrome-devtools-flame-chart-guide.md)
2. 实践文档中的真实优化案例
3. 使用 Performance 面板分析自己的项目

### React/Next.js 开发者
1. 重点阅读 [React 和 Next.js 性能优化指南](./react-nextjs-performance-optimization.md)
2. 学习使用 React DevTools Profiler
3. 实践 Bundle 分析和优化
4. 应用 Next.js 专项优化技巧

### 高级优化
1. 掌握所有 Chrome DevTools 面板的使用
2. 建立性能监控体系
3. 集成 Lighthouse CI 到开发流程
4. 设置和维护性能预算

## 工具对比

### 性能分析工具

| 工具 | 用途 | 优点 | 缺点 |
|------|------|------|------|
| **Chrome DevTools** | 实时性能分析 | 功能强大、免费、实时 | 需要手动分析 |
| **Lighthouse** | 自动化审计 | 全面评分、建议明确 | 只是实验室数据 |
| **WebPageTest** | 真实设备测试 | 真实环境、详细报告 | 速度较慢 |
| **React DevTools Profiler** | React 组件分析 | React 专用、精准 | 仅限 React |
| **Bundle Analyzer** | 包体积分析 | 可视化清晰 | 需要构建 |

### 监控服务

| 服务 | 类型 | 特点 |
|------|------|------|
| **Chrome User Experience Report (CrUX)** | 真实用户监控 (RUM) | 免费、真实数据、覆盖广 |
| **Sentry Performance** | RUM + APM | 集成错误追踪、详细追踪 |
| **New Relic Browser** | RUM | 企业级、功能全面 |
| **Datadog RUM** | RUM | 可观测性平台、集成度高 |

## 性能优化速查表

### 快速诊断
```
页面加载慢？
├─ TTFB > 600ms → 服务器优化、CDN、缓存
├─ 资源体积大 → 压缩、代码分割、tree-shaking
└─ 请求数量多 → HTTP/2、合并资源、懒加载

交互卡顿？
├─ 长任务 >50ms → 代码分割、Web Worker、时间切片
├─ FPS < 60 → 减少布局抖动、优化动画
└─ 内存增长 → 检查内存泄漏

渲染慢？
├─ LCP > 2.5s → 优化关键资源、预加载
├─ CLS > 0.1 → 设置图片尺寸、避免动态插入
└─ FID > 100ms → 减少主线程阻塞
```

### React 优化决策树
```
组件渲染慢？
├─ 频繁重新渲染？
│  ├─ Props 变化 → React.memo
│  ├─ Context 变化 → 拆分 Context
│  └─ 父组件渲染 → memo + useCallback
├─ 计算耗时？
│  └─ useMemo 缓存结果
└─ 列表很长？
   └─ react-window 虚拟滚动
```

### Next.js 优化检查表
```
- [ ] 使用 next/image 替代 <img>
- [ ] 使用 next/font 优化字体加载
- [ ] 使用 next/script 控制第三方脚本
- [ ] 服务端组件优先，客户端组件按需
- [ ] 启用 ISR 或 SSG
- [ ] 配置 experimental.optimizePackageImports
- [ ] 运行 Bundle Analyzer 检查包大小
- [ ] 设置适当的 Cache-Control 头
```

## 推荐资源

### 官方文档
- [Web.dev Performance](https://web.dev/performance/)
- [MDN Web Performance](https://developer.mozilla.org/en-US/docs/Web/Performance)
- [Chrome DevTools Documentation](https://developer.chrome.com/docs/devtools/)
- [Web Vitals](https://web.dev/vitals/)
- [Next.js Performance](https://nextjs.org/docs/app/building-your-application/optimizing)

### 工具和库
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [web-vitals](https://github.com/GoogleChrome/web-vitals)
- [react-window](https://github.com/bvaughn/react-window)
- [next/bundle-analyzer](https://www.npmjs.com/package/@next/bundle-analyzer)
- [Bundle Buddy](https://bundle-buddy.com/)

### 学习资源
- [Performance Calendar](https://calendar.perfplanet.com/)
- [High Performance Browser Networking](https://hpbn.co/)
- [Frontend Masters - Web Performance](https://frontendmasters.com/courses/web-performance/)

## 贡献

欢迎补充和完善性能优化相关的研究内容。

## 更新日志

**2024-11 更新**:
- ✨ 新增 Chrome DevTools 火焰图深度指南
- ✨ 新增 React 和 Next.js 性能优化指南
- ✨ 新增 Chrome DevTools 完整性能调试指南
- 📝 包含多个真实世界优化案例
- 🆕 涵盖 Chrome DevTools 2024 新功能（CPU 节流校准、20x 减速）
- 🔥 详细的火焰图分析和实战案例
