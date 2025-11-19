# React 和 Next.js 性能优化指南

## 目录
1. [React DevTools Profiler 使用指南](#react-devtools-profiler-使用指南)
2. [React 性能优化技巧](#react-性能优化技巧)
3. [Next.js 性能优化](#nextjs-性能优化)
4. [Bundle 分析与优化](#bundle-分析与优化)
5. [实战案例](#实战案例)
6. [性能测试与监控](#性能测试与监控)

---

## React DevTools Profiler 使用指南

### 安装 React DevTools

```bash
# Chrome 扩展
# https://chrome.google.com/webstore/detail/react-developer-tools/

# 或使用独立应用
npm install -g react-devtools
react-devtools
```

### Profiler 面板概览

```
┌─────────────────────────────────────────────────────┐
│  ⚫ Start Profiling  |  ⚙️ Settings  |  💾 Load      │
├─────────────────────────────────────────────────────┤
│  Commits: ◀ 1 / 5 ▶                                │
│                                                      │
│  📊 Flamegraph | 📈 Ranked | 🔍 Component           │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Flamegraph View (火焰图):                          │
│  ┌──────────────────────────────────────────┐      │
│  │ <App> 45.2ms (45.2ms total)              │      │
│  └┬─────────────────────────────────────────┘      │
│   ├─ <Header> 2.1ms (2.1ms)                        │
│   ├─ <Sidebar> 1.8ms (1.8ms)                       │
│   └─ <ProductList> 41.3ms (41.3ms)                 │
│      └─ <ProductCard> 0.8ms × 50                   │
│                                                      │
│  Rendered at: 12:34:56.789                          │
│  Duration: 45.2ms                                   │
│  Why did this render? Props changed                 │
└─────────────────────────────────────────────────────┘
```

### Profiler 的两种视图

#### 1. Flamegraph（火焰图）视图

**特点**:
- 显示组件层级结构
- 横向宽度 = 渲染时间
- 纵向高度 = 组件嵌套深度

**颜色含义**:
```javascript
🟦 蓝色/绿色: 渲染速度快（< 5ms）
🟨 黄色: 渲染速度中等（5-15ms）
🟧 橙色: 渲染速度慢（15-50ms）
🟥 红色: 渲染速度很慢（> 50ms）
⬜ 灰色: 这次没有渲染
```

**示例火焰图分析**:
```javascript
// 火焰图显示
┌─────────────────────────────────────────────────────┐
│ <App> 156ms / 156ms                  🟥            │  ← 根组件很慢
└┬────────────────────────────────────────────────────┘
 ├─ <Header> 2ms / 2ms           🟦                    ← 快速
 ├─ <Sidebar> 3ms / 3ms          🟦                    ← 快速
 └─ <Main> 151ms / 151ms               🟥             ← 瓶颈！
    └─ <DataTable> 148ms / 148ms       🟥             ← 主要问题
       ├─ <Row> 2.5ms × 60           🟨🟨🟨           ← 渲染60行

分析:
1. 总渲染时间: 156ms
2. 瓶颈: <DataTable> 占了 95% 的时间
3. 问题: 渲染了 60 个 <Row> 组件
4. 优化方向: 使用虚拟滚动或分页
```

#### 2. Ranked（排序）视图

**特点**:
- 按渲染时间从长到短排序
- 快速找到最慢的组件
- 显示每个组件的渲染次数

**示例**:
```javascript
┌──────────────────────────────────────────────────┐
│ Component          │ Duration │ Renders │ Self   │
├──────────────────────────────────────────────────┤
│ DataTable          │ 148ms    │ 1       │ 148ms  │  ← 最慢
│ ProductList        │ 45ms     │ 1       │ 5ms    │
│ Row                │ 2.5ms    │ 60      │ 2.5ms  │  ← 渲染60次
│ ProductCard        │ 0.8ms    │ 50      │ 0.8ms  │
│ Header             │ 2ms      │ 1       │ 2ms    │
└──────────────────────────────────────────────────┘

Self Time: 组件自身的渲染时间（不包括子组件）
Total Time: 包括子组件的总时间
```

### Profiler 设置选项

```javascript
// ⚙️ Settings 菜单

✓ Record why each component rendered
  // 记录组件为什么重新渲染
  // Props changed, State changed, Hooks changed, Parent rendered

✓ Hide commits below X ms
  // 隐藏渲染时间低于 X ms 的 commits
  // 有助于过滤噪音，聚焦问题

□ Profiler Throttling (6x slowdown)
  // CPU 节流，模拟低端设备
```

### 使用 Profiler 的步骤

#### 步骤 1: 开始录制

```javascript
1. 打开 React DevTools -> Profiler 标签
2. 点击蓝色录制按钮 ⚫
3. 执行需要分析的操作（点击按钮、滚动、输入等）
4. 点击停止按钮 ⏹
```

#### 步骤 2: 分析 Commits

```javascript
// Commit = React 的一次更新渲染

查看 Commits:
- 使用 ◀ ▶ 浏览不同的 commit
- 每个 commit 代表一次状态更新

关注指标:
- Commit duration: 这次更新花费的时间
- Rendered components: 重新渲染了多少组件
```

#### 步骤 3: 识别问题组件

```javascript
在 Flamegraph 视图:
1. 找到红色/橙色的组件（慢）
2. 查看宽度最宽的组件（耗时最长）
3. 点击组件查看详细信息

在 Ranked 视图:
1. 直接看排在最上面的组件
2. 注意 "Renders" 列，高频渲染也是问题
```

#### 步骤 4: 查看渲染原因

```javascript
点击组件后，右侧面板显示:

Why did this render?
✓ Props changed
  - prop "data" changed

✓ Hooks changed
  - useState hook 1 changed

Parent component rendered
  - 父组件重新渲染导致子组件也渲染
```

### 实际示例：分析 React 应用

```javascript
// 问题应用
function App() {
  const [count, setCount] = useState(0);
  const [users, setUsers] = useState([]);

  // ❌ 问题：每次渲染都创建新对象
  const config = {
    theme: 'dark',
    layout: 'grid'
  };

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>

      {/* ❌ 即使 count 变化，这个组件也会重新渲染 */}
      <ExpensiveComponent config={config} users={users} />
    </div>
  );
}

function ExpensiveComponent({ config, users }) {
  // 模拟复杂计算
  const processedData = users.map(u => heavyTransform(u));

  return <div>{/* 渲染 */}</div>;
}
```

**Profiler 会显示**:
```javascript
Commit #1 (点击按钮，count: 0 → 1)
┌────────────────────────────────────────┐
│ <App> 158ms / 158ms           🟥      │
└┬───────────────────────────────────────┘
 └─ <ExpensiveComponent> 155ms / 155ms 🟥

点击 <ExpensiveComponent>:
Why did this render?
✓ Props changed
  - prop "config" changed

问题诊断:
1. config 每次都是新对象 (引用不同)
2. 导致 ExpensiveComponent 不必要的重新渲染
3. 每次重新计算 processedData
```

**优化后**:
```javascript
function App() {
  const [count, setCount] = useState(0);
  const [users, setUsers] = useState([]);

  // ✅ 使用 useMemo 缓存对象
  const config = useMemo(() => ({
    theme: 'dark',
    layout: 'grid'
  }), []);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>

      {/* ✅ 使用 memo 包裹 */}
      <ExpensiveComponent config={config} users={users} />
    </div>
  );
}

// ✅ memo 阻止不必要的重新渲染
const ExpensiveComponent = memo(({ config, users }) => {
  // ✅ useMemo 缓存计算结果
  const processedData = useMemo(
    () => users.map(u => heavyTransform(u)),
    [users]
  );

  return <div>{/* 渲染 */}</div>;
});
```

**优化后的 Profiler**:
```javascript
Commit #1 (点击按钮，count: 0 → 1)
┌────────────────────────────────┐
│ <App> 0.8ms / 0.8ms    🟦     │  ← 只渲染 App
└────────────────────────────────┘
  <ExpensiveComponent>  (灰色)     ← 没有重新渲染！

性能提升:
- 渲染时间: 158ms → 0.8ms (99.5% 减少)
- ExpensiveComponent 没有重新渲染
```

---

## React 性能优化技巧

### 1. 使用 React.memo 避免不必要的重新渲染

```javascript
// ❌ 问题：父组件更新时，子组件总是重新渲染
function ParentComponent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <ChildComponent name="John" />  {/* 即使 props 没变也重新渲染 */}
    </div>
  );
}

function ChildComponent({ name }) {
  console.log('ChildComponent rendered');
  return <div>Hello {name}</div>;
}

// ✅ 解决方案：使用 memo
const ChildComponent = memo(({ name }) => {
  console.log('ChildComponent rendered');
  return <div>Hello {name}</div>;
});

// 或自定义比较函数
const ChildComponent = memo(
  ({ name, data }) => {
    return <div>Hello {name}</div>;
  },
  (prevProps, nextProps) => {
    // 返回 true 表示不需要重新渲染
    return prevProps.name === nextProps.name &&
           prevProps.data.id === nextProps.data.id;
  }
);
```

### 2. 使用 useMemo 缓存计算结果

```javascript
// ❌ 问题：每次渲染都重新计算
function ProductList({ products, filter }) {
  // 即使 products 和 filter 没变，也会重新过滤
  const filteredProducts = products.filter(p =>
    p.name.includes(filter)
  );

  const sortedProducts = filteredProducts.sort((a, b) =>
    a.price - b.price
  );

  return <div>{/* 渲染 sortedProducts */}</div>;
}

// ✅ 解决方案：使用 useMemo
function ProductList({ products, filter }) {
  const filteredProducts = useMemo(() =>
    products.filter(p => p.name.includes(filter)),
    [products, filter]  // 只有依赖变化时才重新计算
  );

  const sortedProducts = useMemo(() =>
    [...filteredProducts].sort((a, b) => a.price - b.price),
    [filteredProducts]
  );

  return <div>{/* 渲染 sortedProducts */}</div>;
}
```

**何时使用 useMemo**:
```javascript
✅ 适合使用:
- 复杂计算（排序、过滤大数组）
- 创建对象/数组作为 props
- 依赖数组稳定，不频繁变化

❌ 不需要使用:
- 简单计算（加法、字符串拼接）
- 原始值（字符串、数字、布尔值）
- 计算成本低于 useMemo 本身的开销
```

### 3. 使用 useCallback 缓存函数引用

```javascript
// ❌ 问题：每次渲染创建新函数，导致子组件重新渲染
function ParentComponent() {
  const [count, setCount] = useState(0);

  // 每次渲染都是新函数
  const handleClick = () => {
    console.log('Clicked');
  };

  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <ChildButton onClick={handleClick} />  {/* 每次都重新渲染 */}
    </>
  );
}

const ChildButton = memo(({ onClick }) => {
  console.log('ChildButton rendered');
  return <button onClick={onClick}>Click me</button>;
});

// ✅ 解决方案：使用 useCallback
function ParentComponent() {
  const [count, setCount] = useState(0);

  // 函数引用保持不变
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);  // 依赖数组为空，函数永不变化

  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <ChildButton onClick={handleClick} />  {/* 不会重新渲染 */}
    </>
  );
}

// 带依赖的例子
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleSearch = useCallback(async () => {
    const data = await fetch(`/api/search?q=${query}`);
    setResults(data);
  }, [query]);  // query 变化时，函数才变化

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchButton onSearch={handleSearch} />
    </>
  );
}
```

### 4. 虚拟化长列表

```javascript
// ❌ 问题：渲染1000个项目
function ProductList({ products }) {
  return (
    <div>
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

// 性能问题:
// - 初始渲染: 2500ms
// - 内存: 大量DOM节点
// - 滚动: 卡顿

// ✅ 解决方案：react-window (轻量) 或 react-virtualized (功能多)
import { FixedSizeList } from 'react-window';

function ProductList({ products }) {
  return (
    <FixedSizeList
      height={600}              // 可见区域高度
      itemCount={products.length}
      itemSize={100}            // 每项高度
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          <ProductCard product={products[index]} />
        </div>
      )}
    </FixedSizeList>
  );
}

// 动态高度的列表
import { VariableSizeList } from 'react-window';

function ProductList({ products }) {
  const listRef = useRef();

  const getItemSize = (index) => {
    // 根据内容动态计算高度
    return products[index].featured ? 200 : 100;
  };

  return (
    <VariableSizeList
      ref={listRef}
      height={600}
      itemCount={products.length}
      itemSize={getItemSize}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          <ProductCard product={products[index]} />
        </div>
      )}
    </VariableSizeList>
  );
}

// 性能提升:
// - 初始渲染: 2500ms → 80ms (97% 减少)
// - 只渲染可见的 6-8 个项目
// - 滚动流畅 60 FPS
```

### 5. 代码分割与懒加载

```javascript
// ❌ 问题：一次加载所有路由组件
import Home from './pages/Home';
import About from './pages/About';
import Dashboard from './pages/Dashboard';
import Settings from './pages/Settings';

function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/about" element={<About />} />
      <Route path="/dashboard" element={<Dashboard />} />
      <Route path="/settings" element={<Settings />} />
    </Routes>
  );
}

// Bundle 大小: 2.5MB
// 首屏加载: 慢

// ✅ 解决方案：使用 React.lazy 和 Suspense
import { lazy, Suspense } from 'react';

const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}

// 改进:
// - 首屏只加载 Home: 450KB
// - 其他页面按需加载
// - 首屏加载: 快 80%

// 带预加载的高级方案
const DashboardLazy = lazy(() => import('./pages/Dashboard'));

function App() {
  // 鼠标悬停时预加载
  const handleMouseEnter = () => {
    import('./pages/Dashboard');
  };

  return (
    <div>
      <Link to="/dashboard" onMouseEnter={handleMouseEnter}>
        Dashboard
      </Link>

      <Suspense fallback={<Spinner />}>
        <Routes>
          <Route path="/dashboard" element={<DashboardLazy />} />
        </Routes>
      </Suspense>
    </div>
  );
}
```

### 6. 优化 Context 使用

```javascript
// ❌ 问题：Context 更新导致所有消费者重新渲染
const AppContext = createContext();

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [settings, setSettings] = useState({});

  // 所有状态在一个 Context
  const value = {
    user, setUser,
    theme, setTheme,
    settings, setSettings
  };

  return (
    <AppContext.Provider value={value}>
      {children}
    </AppContext.Provider>
  );
}

// 问题：theme 变化时，所有使用 user 的组件也重新渲染

// ✅ 解决方案1：拆分 Context
const UserContext = createContext();
const ThemeContext = createContext();
const SettingsContext = createContext();

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [settings, setSettings] = useState({});

  return (
    <UserContext.Provider value={{ user, setUser }}>
      <ThemeContext.Provider value={{ theme, setTheme }}>
        <SettingsContext.Provider value={{ settings, setSettings }}>
          {children}
        </SettingsContext.Provider>
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}

// 组件只订阅需要的 Context
function UserProfile() {
  const { user } = useContext(UserContext);  // 只关心 user
  return <div>{user.name}</div>;
}

function ThemeToggle() {
  const { theme, setTheme } = useContext(ThemeContext);  // 只关心 theme
  return <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
    Toggle
  </button>;
}

// ✅ 解决方案2：使用 useMemo 优化 Context value
function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  // 缓存 value 对象
  const userValue = useMemo(() => ({ user, setUser }), [user]);
  const themeValue = useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <UserContext.Provider value={userValue}>
      <ThemeContext.Provider value={themeValue}>
        {children}
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}
```

### 7. 使用 Key 优化列表渲染

```javascript
// ❌ 问题：使用 index 作为 key
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map((todo, index) => (
        <li key={index}>{todo.text}</li>  // 问题！
      ))}
    </ul>
  );
}

// 问题场景：删除第一项
// 之前: [0: "Buy milk", 1: "Walk dog", 2: "Code"]
// 之后: [0: "Walk dog", 1: "Code"]
// React 会重新渲染所有项，因为 key 对应的内容都变了

// ✅ 解决方案：使用稳定的唯一 ID
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.text}</li>  // 使用 ID
      ))}
    </ul>
  );
}

// React 会正确识别哪些项被删除、添加或移动
// 只重新渲染必要的项
```

### 8. 防抖和节流事件处理

```javascript
// ❌ 问题：搜索框每次输入都触发搜索
function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = async (e) => {
    const value = e.target.value;
    setQuery(value);

    // 每次输入都发送请求
    const data = await fetch(`/api/search?q=${value}`);
    setResults(data);
  };

  return <input value={query} onChange={handleChange} />;
}

// ✅ 解决方案：防抖
import { useState, useCallback } from 'react';
import { debounce } from 'lodash';

function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const performSearch = async (value) => {
    const data = await fetch(`/api/search?q=${value}`);
    setResults(data);
  };

  // 防抖：300ms 后才执行
  const debouncedSearch = useCallback(
    debounce((value) => {
      if (value.trim()) {
        performSearch(value);
      }
    }, 300),
    []
  );

  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value);
    debouncedSearch(value);
  };

  return <input value={query} onChange={handleChange} />;
}

// 节流示例：滚动事件
function InfiniteScroll() {
  const [items, setItems] = useState([]);

  const handleScroll = useCallback(
    throttle(() => {
      const { scrollTop, scrollHeight, clientHeight } = document.documentElement;

      if (scrollTop + clientHeight >= scrollHeight - 100) {
        loadMore();
      }
    }, 200),  // 200ms 内最多执行一次
    []
  );

  useEffect(() => {
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, [handleScroll]);

  return <div>{/* 渲染 items */}</div>;
}
```

### 9. 优化图片加载

```javascript
// ❌ 问题：加载大图片
function ProductCard({ product }) {
  return (
    <div>
      <img src={product.imageUrl} alt={product.name} />
    </div>
  );
}

// ✅ 解决方案：响应式图片 + 懒加载
function ProductCard({ product }) {
  return (
    <div>
      <img
        srcSet={`
          ${product.image_small} 400w,
          ${product.image_medium} 800w,
          ${product.image_large} 1200w
        `}
        sizes="(max-width: 600px) 400px, (max-width: 1000px) 800px, 1200px"
        src={product.image_medium}
        alt={product.name}
        loading="lazy"  // 懒加载
      />
    </div>
  );
}

// 使用 Intersection Observer 的自定义懒加载
function LazyImage({ src, alt }) {
  const [imageSrc, setImageSrc] = useState(null);
  const [imageRef, setImageRef] = useState(null);

  useEffect(() => {
    let observer;

    if (imageRef && imageSrc !== src) {
      observer = new IntersectionObserver((entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            setImageSrc(src);
            observer.unobserve(imageRef);
          }
        });
      });

      observer.observe(imageRef);
    }

    return () => {
      if (observer && imageRef) {
        observer.unobserve(imageRef);
      }
    };
  }, [imageRef, src, imageSrc]);

  return (
    <img
      ref={setImageRef}
      src={imageSrc || 'placeholder.jpg'}
      alt={alt}
    />
  );
}
```

---

## Next.js 性能优化

### 1. 图片优化 (next/image)

```javascript
// ❌ 传统 img 标签
function Hero() {
  return (
    <div>
      <img src="/hero.jpg" alt="Hero" width={1200} height={600} />
    </div>
  );
}

// 问题:
// - 没有自动优化
// - 不支持现代格式 (WebP, AVIF)
// - 没有懒加载
// - LCP 很慢

// ✅ 使用 next/image
import Image from 'next/image';

function Hero() {
  return (
    <div>
      <Image
        src="/hero.jpg"
        alt="Hero"
        width={1200}
        height={600}
        priority  // 预加载（对 LCP 图片使用）
        placeholder="blur"  // 模糊占位符
        blurDataURL="data:image/..."  // 可选：自定义占位符
      />
    </div>
  );
}

// 自动优化:
// ✓ 自动转换为 WebP/AVIF
// ✓ 响应式图片（不同尺寸）
// ✓ 懒加载（默认）
// ✓ 防止布局偏移（CLS）

// 外部图片
function ProductCard({ product }) {
  return (
    <Image
      src={product.imageUrl}  // 外部 URL
      alt={product.name}
      width={400}
      height={300}
      loading="lazy"
    />
  );
}

// next.config.js 配置
module.exports = {
  images: {
    domains: ['cdn.example.com', 'images.unsplash.com'],
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
};
```

### 2. 字体优化 (next/font)

```javascript
// ❌ 传统字体加载
// pages/_document.js
<link
  href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap"
  rel="stylesheet"
/>

// 问题:
// - 网络请求延迟
// - 字体闪烁 (FOIT/FOUT)
// - 影响 FCP/LCP

// ✅ Next.js 13+ 使用 next/font
import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
});

const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-roboto-mono',
});

export default function RootLayout({ children }) {
  return (
    <html lang="en" className={`${inter.variable} ${robotoMono.variable}`}>
      <body>{children}</body>
    </html>
  );
}

// CSS 中使用
// globals.css
body {
  font-family: var(--font-inter);
}

code {
  font-family: var(--font-roboto-mono);
}

// 自定义字体
import localFont from 'next/font/local';

const myFont = localFont({
  src: './my-font.woff2',
  display: 'swap',
  variable: '--font-my-font',
});

// 优势:
// ✓ 零布局偏移
// ✓ 自托管字体（无外部请求）
// ✓ 自动优化
// ✓ 内置 font-display: swap
```

### 3. Script 优化 (next/script)

```javascript
// ❌ 传统 script 标签
// 会阻塞页面渲染
<script src="https://analytics.example.com/script.js"></script>

// ✅ 使用 next/script
import Script from 'next/script';

export default function MyApp({ Component, pageProps }) {
  return (
    <>
      {/* Google Analytics - afterInteractive 策略 */}
      <Script
        src="https://www.googletagmanager.com/gtag/js?id=GA_MEASUREMENT_ID"
        strategy="afterInteractive"  // 页面可交互后加载
      />

      <Script id="google-analytics" strategy="afterInteractive">
        {`
          window.dataLayer = window.dataLayer || [];
          function gtag(){dataLayer.push(arguments);}
          gtag('js', new Date());
          gtag('config', 'GA_MEASUREMENT_ID');
        `}
      </Script>

      {/* 非关键脚本 - lazyOnload 策略 */}
      <Script
        src="https://widget.example.com/widget.js"
        strategy="lazyOnload"  // 空闲时加载
      />

      {/* 关键脚本 - beforeInteractive 策略 */}
      <Script
        src="/polyfills.js"
        strategy="beforeInteractive"  // 页面可交互前加载
      />

      <Component {...pageProps} />
    </>
  );
}

// 策略说明:
// - beforeInteractive: 关键脚本，在页面可交互前加载
// - afterInteractive: 页面可交互后立即加载（默认）
// - lazyOnload: 空闲时加载
// - worker: 在 Web Worker 中加载（实验性）
```

### 4. 动态导入

```javascript
// ✅ 客户端组件懒加载
import dynamic from 'next/dynamic';

const DynamicComponent = dynamic(() => import('../components/HeavyComponent'), {
  loading: () => <p>Loading...</p>,
  ssr: false,  // 禁用 SSR（仅客户端渲染）
});

function Page() {
  return (
    <div>
      <h1>My Page</h1>
      <DynamicComponent />
    </div>
  );
}

// 条件加载
function Page() {
  const [showModal, setShowModal] = useState(false);
  const [ModalComponent, setModalComponent] = useState(null);

  const handleOpenModal = async () => {
    if (!ModalComponent) {
      const module = await import('../components/Modal');
      setModalComponent(() => module.default);
    }
    setShowModal(true);
  };

  return (
    <div>
      <button onClick={handleOpenModal}>Open Modal</button>
      {showModal && ModalComponent && <ModalComponent />}
    </div>
  );
}

// 命名导出
const ClientOnlyComponent = dynamic(
  () => import('../components/ClientComponent').then(mod => mod.ClientComponent),
  { ssr: false }
);
```

### 5. 服务端组件 (App Router - Next.js 13+)

```javascript
// app/page.js - 默认是服务端组件
async function Page() {
  // 在服务端获取数据
  const data = await fetch('https://api.example.com/data', {
    cache: 'force-cache'  // 缓存
  }).then(res => res.json());

  return (
    <div>
      <h1>Products</h1>
      {data.map(item => (
        <ProductCard key={item.id} product={item} />
      ))}
    </div>
  );
}

// app/components/AddToCart.js - 客户端组件
'use client';  // 标记为客户端组件

import { useState } from 'react';

export function AddToCart({ productId }) {
  const [loading, setLoading] = useState(false);

  const handleClick = async () => {
    setLoading(true);
    await fetch('/api/cart', {
      method: 'POST',
      body: JSON.stringify({ productId })
    });
    setLoading(false);
  };

  return (
    <button onClick={handleClick} disabled={loading}>
      {loading ? 'Adding...' : 'Add to Cart'}
    </button>
  );
}

// app/page.js - 混合使用
import { AddToCart } from './components/AddToCart';

async function Page() {
  const products = await getProducts();

  return (
    <div>
      {products.map(product => (
        <div key={product.id}>
          <h2>{product.name}</h2>
          {/* 服务端渲染静态内容 */}
          <p>{product.description}</p>

          {/* 客户端组件处理交互 */}
          <AddToCart productId={product.id} />
        </div>
      ))}
    </div>
  );
}

// 优势:
// ✓ 减少客户端 JavaScript
// ✓ 更好的 SEO
// ✓ 更快的首屏加载
// ✓ 服务端数据获取
```

### 6. 路由预取

```javascript
// Next.js 自动预取可见的 Link
import Link from 'next/link';

function Navigation() {
  return (
    <nav>
      {/* 默认会预取 */}
      <Link href="/about">About</Link>

      {/* 禁用预取 */}
      <Link href="/contact" prefetch={false}>Contact</Link>

      {/* 条件预取 */}
      <Link
        href="/dashboard"
        prefetch={user.isPremium}  // 只为高级用户预取
      >
        Dashboard
      </Link>
    </nav>
  );
}

// 编程式预取
import { useRouter } from 'next/navigation';

function ProductCard({ product }) {
  const router = useRouter();

  const handleMouseEnter = () => {
    router.prefetch(`/products/${product.id}`);
  };

  return (
    <div onMouseEnter={handleMouseEnter}>
      <Link href={`/products/${product.id}`}>
        {product.name}
      </Link>
    </div>
  );
}
```

---

## Bundle 分析与优化

### 安装和配置 Bundle Analyzer

```bash
# 安装
npm install --save-dev @next/bundle-analyzer

# 或
yarn add -D @next/bundle-analyzer
```

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  // 你的 Next.js 配置
  reactStrictMode: true,
  swcMinify: true,
});

// package.json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "analyze": "ANALYZE=true next build"
  }
}
```

### 运行分析

```bash
npm run analyze
```

这会生成两个可视化页面：
1. **Client bundle** - 客户端 JavaScript
2. **Server bundle** - 服务端代码

### 分析结果解读

```
Bundle Analyzer 显示:

┌──────────────────────────────────────────┐
│ 📦 Total: 2.5 MB                         │
├──────────────────────────────────────────┤
│ ▓▓▓▓▓▓▓▓▓▓ lodash (890 KB) 35.6%       │  ← 最大依赖
│ ▓▓▓▓▓▓▓ moment (420 KB) 16.8%          │
│ ▓▓▓▓▓ @mui/material (350 KB) 14%       │
│ ▓▓▓▓ react-dom (280 KB) 11.2%          │
│ ▓▓▓ your-code (250 KB) 10%             │
│ ▓▓ other packages (310 KB) 12.4%       │
└──────────────────────────────────────────┘

问题识别:
1. lodash 890KB - 可能导入了整个库
2. moment 420KB - 可以换成更轻量的日期库
3. @mui/material - 检查是否tree-shaking正常
```

### 优化大型依赖

```javascript
// ❌ 问题：导入整个 lodash
import _ from 'lodash';

const result = _.debounce(myFunction, 300);

// Bundle: +890 KB

// ✅ 解决方案1：只导入需要的函数
import debounce from 'lodash/debounce';

const result = debounce(myFunction, 300);

// Bundle: +15 KB (减少 98%)

// ✅ 解决方案2：使用 lodash-es (支持 tree-shaking)
import { debounce } from 'lodash-es';

// ❌ 问题：Moment.js 太大
import moment from 'moment';
const date = moment().format('YYYY-MM-DD');

// Bundle: +420 KB

// ✅ 解决方案：使用 date-fns
import { format } from 'date-fns';
const date = format(new Date(), 'yyyy-MM-dd');

// Bundle: +13 KB (减少 97%)

// 或使用原生 Intl
const date = new Intl.DateTimeFormat('en-US').format(new Date());
// Bundle: 0 KB (原生 API)
```

### Next.js 特定优化

```javascript
// next.config.js

module.exports = {
  // 1. 压缩
  swcMinify: true,  // 使用 SWC 压缩（比 Terser 快）

  // 2. 包导入优化
  experimental: {
    optimizePackageImports: ['@mui/material', '@mui/icons-material'],
    // 自动优化这些包的导入
  },

  // 3. 输出独立服务器（减小部署体积）
  output: 'standalone',

  // 4. webpack 优化
  webpack: (config, { isServer }) => {
    // 客户端bundle优化
    if (!isServer) {
      config.optimization = {
        ...config.optimization,
        splitChunks: {
          chunks: 'all',
          cacheGroups: {
            // 提取第三方库
            vendor: {
              test: /[\\/]node_modules[\\/]/,
              name(module) {
                const packageName = module.context.match(
                  /[\\/]node_modules[\\/](.*?)([\\/]|$)/
                )[1];
                return `vendor.${packageName.replace('@', '')}`;
              },
              priority: 10,
            },
            // 提取公共代码
            common: {
              minChunks: 2,
              priority: 5,
              reuseExistingChunk: true,
            },
          },
        },
      };
    }

    return config;
  },
};
```

---

## 实战案例

### 案例：电商产品列表优化

#### 初始状态

```javascript
// pages/products.js
function ProductsPage() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>Loading...</div>;

  return (
    <div>
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

function ProductCard({ product }) {
  return (
    <div>
      <img src={product.imageUrl} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button>Add to Cart</button>
    </div>
  );
}
```

**性能问题**:
- 初始加载: 3.2s
- LCP: 2.8s
- 渲染1000个产品
- 图片未优化
- 没有使用 SSR

#### 优化后

```javascript
// app/products/page.js (Next.js 13+ App Router)
import { Suspense } from 'react';
import Image from 'next/image';
import { FixedSizeList } from 'react-window';

// 服务端组件 - SSR
async function ProductsPage() {
  // 服务端获取数据
  const products = await fetch('https://api.example.com/products', {
    next: { revalidate: 3600 }  // ISR: 1小时重新验证
  }).then(res => res.json());

  return (
    <div>
      <h1>Products</h1>
      <Suspense fallback={<ProductsSkeleton />}>
        <ProductList products={products} />
      </Suspense>
    </div>
  );
}

// 客户端组件 - 虚拟滚动
'use client';

function ProductList({ products }) {
  return (
    <FixedSizeList
      height={800}
      itemCount={products.length}
      itemSize={250}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          <ProductCard product={products[index]} />
        </div>
      )}
    </FixedSizeList>
  );
}

// memo 优化
const ProductCard = memo(({ product }) => {
  return (
    <div className="product-card">
      {/* next/image 优化 */}
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={300}
        height={300}
        loading="lazy"
        placeholder="blur"
        blurDataURL={product.blurDataURL}
      />
      <h3>{product.name}</h3>
      <p>${product.price}</p>

      {/* 客户端交互 */}
      <AddToCartButton productId={product.id} />
    </div>
  );
});

// 骨架屏
function ProductsSkeleton() {
  return (
    <div className="grid">
      {Array.from({ length: 12 }).map((_, i) => (
        <div key={i} className="skeleton-card" />
      ))}
    </div>
  );
}

export default ProductsPage;
```

**性能提升**:
- 初始加载: 3.2s → 0.8s (75% 减少)
- LCP: 2.8s → 1.2s (57% 减少)
- FCP: 1.8s → 0.6s (67% 减少)
- 只渲染可见的 4-6 个产品
- 图片自动优化为 WebP
- SEO 友好（SSR）

---

## 性能测试与监控

### 1. 使用 Lighthouse CI

```bash
# 安装
npm install -g @lhci/cli

# lighthouserc.js
module.exports = {
  ci: {
    collect: {
      startServerCommand: 'npm run start',
      url: ['http://localhost:3000/'],
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};

# 运行
lhci autorun
```

### 2. 生产环境监控

```javascript
// pages/_app.js
import { useReportWebVitals } from 'next/web-vitals';

export function reportWebVitals(metric) {
  // 发送到分析服务
  const body = JSON.stringify(metric);
  const url = '/api/analytics';

  if (navigator.sendBeacon) {
    navigator.sendBeacon(url, body);
  } else {
    fetch(url, { body, method: 'POST', keepalive: true });
  }
}

// 或使用 Web Vitals 库
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    id: metric.id,
    page: window.location.pathname,
  });

  fetch('/api/analytics', { method: 'POST', body });
}

getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getFCP(sendToAnalytics);
getLCP(sendToAnalytics);
getTTFB(sendToAnalytics);
```

### 3. 性能预算

```javascript
// webpack.config.js (Next.js 通过 next.config.js 配置)
module.exports = {
  performance: {
    maxEntrypointSize: 500000,  // 500KB
    maxAssetSize: 300000,       // 300KB
    hints: 'error',
  },
};

// 构建时如果超过预算会报错
```

---

## 总结与最佳实践

### React 性能优化检查清单

```javascript
✅ 使用 React DevTools Profiler 识别慢组件
✅ 使用 memo 避免不必要的重新渲染
✅ 使用 useMemo 缓存昂贵的计算
✅ 使用 useCallback 缓存函数引用
✅ 使用稳定的 key（不要用 index）
✅ 虚拟化长列表（react-window）
✅ 代码分割（React.lazy）
✅ 拆分 Context，避免过度渲染
✅ 防抖/节流高频事件
✅ 优化图片加载
```

### Next.js 性能优化检查清单

```javascript
✅ 使用 next/image 优化图片
✅ 使用 next/font 优化字体
✅ 使用 next/script 优化第三方脚本
✅ 使用服务端组件减少客户端 JS
✅ 启用 ISR (增量静态再生)
✅ 配置适当的缓存策略
✅ 使用 Bundle Analyzer 分析包大小
✅ 优化第三方依赖
✅ 启用 SWC 压缩
✅ 使用 Lighthouse CI 监控性能
```

### 何时优化

```javascript
1. 测量优先
   - 使用 Profiler 和 Performance 工具
   - 不要盲目优化

2. 优先优化最大瓶颈
   - 20% 的代码导致 80% 的性能问题
   - 找到并优化那 20%

3. 平衡开发体验和性能
   - 不要过度优化
   - 代码可读性也很重要

4. 持续监控
   - 集成到 CI/CD
   - 设置性能预算
   - 监控真实用户指标
```

通过系统地应用这些技巧和工具，你可以显著提升 React 和 Next.js 应用的性能！
