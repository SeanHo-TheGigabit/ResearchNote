# Web Performance 前端性能优化研究

## 简介

本目录包含前端Web性能优化的系统性研究文档，涵盖性能调试常见问题、工具使用、解决方案和最佳实践。

## 文档列表

### [网页性能调试问题与解决方案](./web-performance-debugging.md)

全面研究前端开发者在优化网页性能时遇到的各种问题及其解决方案。

**主要内容**:
- 加载性能问题（首屏加载、网络请求、第三方脚本）
- 渲染性能问题（重排重绘、JavaScript执行、内存泄漏）
- 资源优化问题（图片、CSS、JavaScript）
- 调试工具详解（Chrome DevTools、Lighthouse等）
- 实战案例和最佳实践

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

## 推荐资源

- [Web.dev Performance](https://web.dev/performance/)
- [MDN Web Performance](https://developer.mozilla.org/en-US/docs/Web/Performance)
- [Chrome DevTools Documentation](https://developer.chrome.com/docs/devtools/)
- [Web Vitals](https://web.dev/vitals/)

## 贡献

欢迎补充和完善性能优化相关的研究内容。
