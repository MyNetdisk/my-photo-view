# react-photo-view 详细实现步骤

**目标**：小白也能按照步骤从零实现完整功能  
**特点**：每个步骤都有完整代码，照抄就能运行

---

## 阶段一：最小可行性版本（MVP）

### Step 1: 创建项目并安装依赖

```bash
# 1. 创建 React + TypeScript 项目
npm create vite@latest my-photo-view -- --template react-ts

# 2. 进入目录
cd my-photo-view

# 3. 安装 less（用于样式）
npm install -D less

# 4. 安装项目依赖
npm install
```

### Step 2: 创建类型定义文件 `src/types.ts`

这是第一个要创建的文件，定义所有数据类型。

```typescript
// src/types.ts

// 图片资源类型
export interface PhotoItem {
  // 唯一标识
  key: string | number;
  // 图片地址
  src?: string;
  // 自定义渲染
  render?: () => React.ReactNode;
  // 自定义覆盖节点
  overlay?: React.ReactNode;
  // 宽度
  width?: number;
  // 高度
  height?: number;
}

// Context 类型
export interface PhotoContextType {
  // 图片列表
  images: PhotoItem[];
  // 是否可见
  visible: boolean;
  // 当前索引
  index: number;
  // 显示预览
  show: (key: string | number) => void;
  // 隐藏预览
  hide: () => void;
  // 更新图片
  update: (item: PhotoItem) => void;
  // 移除图片
  remove: (key: string | number) => void;
  // 生成唯一 ID
  nextId: () => number;
}
```

### Step 3: 创建 Context 文件 `src/photo-context.ts`

创建 React Context，用于组件间通信。

```typescript
// src/photo-context.ts
import { createContext } from 'react';
import { PhotoContextType } from './types';

// 创建 Context，初始值为 undefined
const PhotoContext = createContext<PhotoContextType | undefined>(undefined!);

export default PhotoContext;
```

### Step 4: 创建 useSetState Hook `src/hooks/useSetState.ts`

一个简单的状态管理 Hook，类似 useState 但支持函数式更新。

```typescript
// src/hooks/useSetState.ts
import { useReducer } from 'react';

/**
 * 类似 setState 的 useReducer 封装
 * 支持两种用法：
 * 1. setState({ count: 1 })
 * 2. setState(prev => ({ count: prev.count + 1 }))
 */
export default function useSetState<S extends Record<string, any>>(initialState: S) {
  return useReducer(
    // reducer 函数
    (state: S, action: Partial<S> | ((state: S) => Partial<S>)) => ({
      ...state,
      // 如果 action 是函数，执行它；否则直接合并
      ...(typeof action === 'function' ? action(state) : action),
    }),
    initialState,
  );
}
```

### Step 5: 创建 useMethods Hook `src/hooks/useMethods.ts`

这个 Hook 的作用是：让方法引用保持稳定，避免子组件不必要的重渲染。

```typescript
// src/hooks/useMethods.ts
import { useRef } from 'react';

/**
 * 持久化方法引用
 *
 * 解决的问题：每次组件渲染，方法引用都会变化
 * 用法：const methods = useMethods({ fn: () => {} })
 * 优势：返回的方法对象引用始终不变
 */
export default function useMethods<T extends Record<string, (...args: any[]) => any>>(fn: T) {
  // 使用 useRef 持久化存储
  const { current } = useRef({
    fn, // 最新的 fn（来自 props）
    curr: undefined as T | undefined, // 缓存的稳定方法对象
  });

  // 每次渲染更新 fn（保持最新）
  current.fn = fn;

  // 首次创建时，生成稳定的方法对象
  if (!current.curr) {
    const curr = Object.create(null); // 创建纯净对象（无原型链）

    // 为每个方法创建包装函数
    Object.keys(fn).forEach((key) => {
      // 关键：调用时始终使用最新的 fn
      curr[key] = (...args: unknown[]) => current.fn[key].call(current.fn, ...args);
    });

    current.curr = curr;
  }

  // 返回稳定的方法对象
  return current.curr as T;
}
```

### Step 6: 创建 useInitial Hook `src/hooks/useInitial.ts`

只执行一次的 Hook，类似于 class 组件的 constructor。

```typescript
// src/hooks/useInitial.ts
import { useRef } from 'react';

/**
 * 只执行一次的 Hook
 * 类似 class 组件的 constructor
 *
 * 用法：const value = useInitial(() => computeExpensiveValue())
 */
export default function useInitial<T extends (...args: any) => any>(callback: T) {
  const { current } = useRef({
    sign: false, // 是否已执行
    value: undefined as ReturnType<T>, // 缓存的值
  });

  // 首次执行，调用 callback 并缓存结果
  if (!current.sign) {
    current.sign = true;
    current.value = callback();
  }

  // 返回缓存的值
  return current.value;
}
```

### Step 7: 创建 PhotoProvider `src/PhotoProvider.tsx`

这是整个组件的状态管理中心。

```typescript
// src/PhotoProvider.tsx
import React, { useMemo, useRef } from 'react';
import PhotoContext from './photo-context';
import { PhotoItem } from './types';
import useSetState from './hooks/useSetState';
import useMethods from './hooks/useMethods';

// 初始状态
interface State {
  images: PhotoItem[];
  visible: boolean;
  index: number;
}

const initialState: State = {
  images: [],
  visible: false,
  index: 0,
};

// Props 类型
interface PhotoProviderProps {
  children: React.ReactNode;
}

export default function PhotoProvider({ children }: PhotoProviderProps) {
  // 状态管理
  const [state, updateState] = useSetState(initialState);

  // 生成唯一 ID
  const uniqueIdRef = useRef(0);

  const { images, visible, index } = state;

  // 核心方法
  const methods = useMethods({
    // 生成下一个唯一 ID
    nextId() {
      return (uniqueIdRef.current += 1);
    },

    // 更新图片信息
    update(item: PhotoItem) {
      const currentIndex = images.findIndex(n => n.key === item.key);

      if (currentIndex > -1) {
        // 已存在，更新
        const nextImages = images.slice();
        nextImages[currentIndex] = item;
        updateState({ images: nextImages });
      } else {
        // 不存在，添加
        updateState(prev => ({
          images: [...prev.images, item],
        }));
      }
    },

    // 移除图片
    remove(key: string | number) {
      updateState(prev => {
        const nextImages = prev.images.filter(item => item.key !== key);
        return {
          images: nextImages,
          // 防止索引越界
          index: Math.min(prev.index, Math.max(0, nextImages.length - 1)),
        };
      });
    },

    // 显示预览
    show(key: string | number) {
      const currentIndex = images.findIndex(item => item.key === key);
      if (currentIndex > -1) {
        updateState({
          visible: true,
          index: currentIndex,
        });
      }
    },
  });

  // 关闭和切换方法
  const fn = useMethods({
    // 关闭预览
    close() {
      updateState({ visible: false });
    },

    // 切换图片
    changeIndex(nextIndex: number) {
      // 边界检查
      if (nextIndex < 0 || nextIndex >= images.length) return;
      updateState({ index: nextIndex });
    },
  });

  // 合并状态和方法
  const value = useMemo(() => ({
    ...state,
    ...methods,
    hide: fn.close,
  }), [state, methods, fn]);

  return (
    <PhotoContext.Provider value={value}>
      {children}
    </PhotoContext.Provider>
  );
}
```

### Step 8: 创建 PhotoView `src/PhotoView.tsx`

包裹图片的组件，点击触发预览。

```typescript
// src/PhotoView.tsx
import React, { useContext, useEffect, useRef, Children, cloneElement } from 'react';
import PhotoContext from './photo-context';
import { PhotoItem } from './types';
import useInitial from './hooks/useInitial';
import useMethods from './hooks/useMethods';

interface PhotoViewProps {
  src?: string;
  children?: React.ReactElement;
}

export default function PhotoView({ src, children }: PhotoViewProps) {
  // 使用 Context
  const context = useContext(PhotoContext);

  if (!context) {
    throw new Error('PhotoView must be used within PhotoProvider');
  }

  const { images, update, remove, show, nextId } = context;

  // 生成唯一 key（只执行一次）
  const key = useInitial(() => nextId());

  // 记录原始元素引用
  const originRef = useRef<HTMLElement>(null);

  // 组件卸载时移除
  useEffect(() => {
    return () => {
      remove(key);
    };
  }, [key, remove]);

  // 注册到 Provider
  useEffect(() => {
    if (src) {
      update({
        key,
        src,
      });
    }
  }, [src, key, update]);

  // 点击显示预览
  const handleClick = () => {
    show(key);
  };

  // 如果有子元素，克隆并添加点击事件
  if (children) {
    return cloneElement(children, {
      onClick: handleClick,
      ref: originRef,
    });
  }

  // 如果没有子元素，返回 null
  return null;
}
```

### Step 9: 创建简单样式 `src/index.css`

添加最基础的样式。

```css
/* src/index.css */
.photo-view-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.9);
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
}

.photo-view-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.photo-view-close {
  position: absolute;
  top: 20px;
  right: 20px;
  color: white;
  font-size: 30px;
  cursor: pointer;
  z-index: 10000;
}

.photo-view-counter {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  color: white;
  font-size: 16px;
}
```

### Step 10: 创建 PhotoSlider `src/PhotoSlider.tsx`（基础版）

这是预览的主组件，先创建一个简化版本。

```typescript
// src/PhotoSlider.tsx
import React, { useEffect } from 'react';
import PhotoContext from './photo-context';
import { useContext } from 'react';
import { PhotoItem } from './types';
import './index.css';

export default function PhotoSlider() {
  const context = useContext(PhotoContext);

  if (!context) return null;

  const { images, visible, index, hide, changeIndex } = context;

  // 不可见时不渲染
  if (!visible) return null;

  const currentImage = images[index];

  return (
    <div className="photo-view-overlay" onClick={hide}>
      {/* 阻止冒泡，点击图片不关闭 */}
      <img
        className="photo-view-image"
        src={currentImage?.src}
        alt=""
        onClick={(e) => e.stopPropagation()}
      />

      {/* 关闭按钮 */}
      <div className="photo-view-close" onClick={hide}>×</div>

      {/* 计数器 */}
      <div className="photo-view-counter">
        {index + 1} / {images.length}
      </div>

      {/* 上一张 */}
      {index > 0 && (
        <button
          style={{ position: 'absolute', left: 20, color: 'white', fontSize: 30 }}
          onClick={(e) => {
            e.stopPropagation();
            changeIndex(index - 1);
          }}
        >
          ‹
        </button>
      )}

      {/* 下一张 */}
      {index < images.length - 1 && (
        <button
          style={{ position: 'absolute', right: 20, color: 'white', fontSize: 30 }}
          onClick={(e) => {
            e.stopPropagation();
            changeIndex(index + 1);
          }}
        >
          ›
        </button>
      )}
    </div>
  );
}
```

### Step 11: 修改 App.tsx 测试

```tsx
// src/App.tsx
import React from 'react';
import PhotoProvider from './PhotoProvider';
import PhotoView from './PhotoView';
import PhotoSlider from './PhotoSlider';
import './index.css';

function App() {
  const images = [
    'https://picsum.photos/800/600?random=1',
    'https://picsum.photos/800/600?random=2',
    'https://picsum.photos/800/600?random=3',
  ];

  return (
    <PhotoProvider>
      <div style={{ padding: 20 }}>
        <h1>图片预览示例</h1>
        <div style={{ display: 'flex', gap: 10 }}>
          {images.map((src, i) => (
            <PhotoView key={i} src={src}>
              <img src={src} alt="" style={{ width: 200, cursor: 'pointer' }} />
            </PhotoView>
          ))}
        </div>
      </div>
      <PhotoSlider />
    </PhotoProvider>
  );
}

export default App;
```

### Step 12: 运行测试

```bash
npm run dev
```

打开浏览器，点击图片，应该能看到预览效果了！

---

## 阶段二：添加手势滑动切换

### Step 13: 添加滑动支持 `src/hooks/useSwipe.ts`

```typescript
// src/hooks/useSwipe.ts
import { useState, useRef } from 'react';

interface SwipeOptions {
  onSwipeLeft?: () => void;
  onSwipeRight?: () => void;
  threshold?: number; // 触发滑动的最小距离
}

export default function useSwipe({ onSwipeLeft, onSwipeRight, threshold = 50 }: SwipeOptions) {
  const [startX, setStartX] = useState(0);
  const [startY, setStartY] = useState(0);
  const [isSwiping, setIsSwiping] = useState(false);

  const handleTouchStart = (e: React.TouchEvent) => {
    const touch = e.touches[0];
    setStartX(touch.clientX);
    setStartY(touch.clientY);
    setIsSwiping(true);
  };

  const handleTouchMove = (e: React.TouchEvent) => {
    if (!isSwiping) return;

    const touch = e.touches[0];
    const deltaX = touch.clientX - startX;
    const deltaY = touch.clientY - startY;

    // 判断是水平滑动还是垂直滑动
    // 水平距离 > 垂直距离 → 水平滑动
    if (Math.abs(deltaX) > Math.abs(deltaY) && Math.abs(deltaX) > threshold) {
      if (deltaX < 0 && onSwipeLeft) {
        // 向左滑 → 下一张
        onSwipeLeft();
        setIsSwiping(false);
      } else if (deltaX > 0 && onSwipeRight) {
        // 向右滑 → 上一张
        onSwipeRight();
        setIsSwiping(false);
      }
    }
  };

  const handleTouchEnd = () => {
    setIsSwiping(false);
  };

  return {
    handleTouchStart,
    handleTouchMove,
    handleTouchEnd,
  };
}
```

### Step 14: 在 PhotoSlider 中使用滑动

修改 `src/PhotoSlider.tsx`：

```tsx
import React, { useEffect } from 'react';
import PhotoContext from './photo-context';
import { useContext } from 'react';
import useSwipe from './hooks/useSwipe';
import './index.css';

export default function PhotoSlider() {
  const context = useContext(PhotoContext);

  if (!context) return null;

  const { images, visible, index, hide, changeIndex } = context;

  if (!visible) return null;

  const currentImage = images[index];

  // 添加滑动支持
  const { handleTouchStart, handleTouchMove, handleTouchEnd } = useSwipe({
    onSwipeLeft: () => {
      if (index < images.length - 1) changeIndex(index + 1);
    },
    onSwipeRight: () => {
      if (index > 0) changeIndex(index - 1);
    },
  });

  return (
    <div
      className="photo-view-overlay"
      onClick={hide}
      onTouchStart={handleTouchStart}
      onTouchMove={handleTouchMove}
      onTouchEnd={handleTouchEnd}
    >
      <img className="photo-view-image" src={currentImage?.src} alt="" onClick={(e) => e.stopPropagation()} />

      <div className="photo-view-close" onClick={hide}>
        ×
      </div>
      <div className="photo-view-counter">
        {index + 1} / {images.length}
      </div>
    </div>
  );
}
```

---

## 阶段三：实现长图滚动预览（重点！）

### Step 15: 检测图片类型 `src/utils/detectImageType.ts`

```typescript
// src/utils/detectImageType.ts

/**
 * 检测图片类型：长图 or 普通图
 *
 * 判断逻辑：
 * - 长图：图片高度 > 视口高度
 * - 普通图：图片高度 <= 视口高度
 */
export function detectImageType(imageHeight: number, viewportHeight: number): 'long' | 'normal' {
  return imageHeight > viewportHeight ? 'long' : 'normal';
}

/**
 * 判断是否需要滚动
 */
export function needsScroll(imageHeight: number, viewportHeight: number): boolean {
  return imageHeight > viewportHeight;
}
```

### Step 16: 创建长图滚动 Hook `src/hooks/useLongImageScroll.ts`

这是实现长图滚动的核心代码！

```typescript
// src/hooks/useLongImageScroll.ts
import { useState, useRef, useCallback } from 'react';

interface ScrollState {
  translateY: number; // Y轴偏移
  isScrolling: boolean; // 是否正在滚动
}

export default function useLongImageScroll(imageHeight: number, viewportHeight: number) {
  const [state, setState] = useState<ScrollState>({
    translateY: 0,
    isScrolling: false,
  });

  const startY = useRef(0);
  const startTranslateY = useRef(0);

  // 是否需要滚动
  const shouldScroll = imageHeight > viewportHeight;

  const handleTouchStart = useCallback(
    (clientY: number) => {
      if (!shouldScroll) return;

      startY.current = clientY;
      startTranslateY.current = state.translateY;
      setState((s) => ({ ...s, isScrolling: true }));
    },
    [shouldScroll, state.translateY],
  );

  const handleTouchMove = useCallback(
    (clientY: number) => {
      if (!shouldScroll || !state.isScrolling) return;

      const deltaY = clientY - startY.current;
      const newTranslateY = startTranslateY.current + deltaY;

      // 计算边界
      const maxTranslateY = 0;
      const minTranslateY = -(imageHeight - viewportHeight);

      let finalTranslateY = newTranslateY;

      // 顶部边界：允许拖动但有阻力
      if (newTranslateY > maxTranslateY) {
        finalTranslateY = newTranslateY * 0.3;
      }
      // 底部边界：允许拖动但有阻力
      else if (newTranslateY < minTranslateY) {
        finalTranslateY = minTranslateY + (newTranslateY - minTranslateY) * 0.3;
      }

      setState((s) => ({ ...s, translateY: finalTranslateY }));
    },
    [shouldScroll, state.isScrolling, imageHeight, viewportHeight],
  );

  const handleTouchEnd = useCallback(() => {
    if (!shouldScroll) return;

    // 边界吸附
    const maxTranslateY = 0;
    const minTranslateY = -(imageHeight - viewportHeight);

    let finalTranslateY = state.translateY;

    // 超过边界时吸附
    if (finalTranslateY > maxTranslateY) {
      finalTranslateY = maxTranslateY;
    } else if (finalTranslateY < minTranslateY) {
      finalTranslateY = minTranslateY;
    }

    setState((s) => ({
      ...s,
      translateY: finalTranslateY,
      isScrolling: false,
    }));
  }, [shouldScroll, state.translateY, imageHeight, viewportHeight]);

  return {
    translateY: state.translateY,
    isScrolling: state.isScrolling,
    shouldScroll,
    handleTouchStart,
    handleTouchMove,
    handleTouchEnd,
  };
}
```

### Step 17: 更新 PhotoSlider 支持长图

```tsx
// src/PhotoSlider.tsx（长图版）
import React, { useState, useEffect, useRef } from 'react';
import PhotoContext from './photo-context';
import { useContext } from 'react';
import useSwipe from './hooks/useSwipe';
import useLongImageScroll from './hooks/useLongImageScroll';
import './index.css';

export default function PhotoSlider() {
  const context = useContext(PhotoContext);
  if (!context) return null;

  const { images, visible, index, hide, changeIndex } = context;

  if (!visible) return null;

  const currentImage = images[index];
  const [imageHeight, setImageHeight] = useState(0);
  const [isLoaded, setIsLoaded] = useState(false);

  // 图片加载后获取尺寸
  const handleImageLoad = (e: React.SyntheticEvent<HTMLImageElement>) => {
    const img = e.target as HTMLImageElement;
    setImageHeight(img.offsetHeight);
    setIsLoaded(true);
  };

  // 长图滚动
  const viewportHeight = window.innerHeight;
  const {
    translateY,
    shouldScroll,
    handleTouchStart: handleScrollTouchStart,
    handleTouchMove: handleScrollTouchMove,
    handleTouchEnd: handleScrollTouchEnd,
  } = useLongImageScroll(imageHeight, viewportHeight);

  // 滑动切换
  const { handleTouchStart, handleTouchMove, handleTouchEnd } = useSwipe({
    onSwipeLeft: () => {
      if (!shouldScroll || Math.abs(translateY) < 10) {
        if (index < images.length - 1) changeIndex(index + 1);
      }
    },
    onSwipeRight: () => {
      if (!shouldScroll || Math.abs(translateY) < 10) {
        if (index > 0) changeIndex(index - 1);
      }
    },
  });

  // 统一触摸事件处理
  const handleTouchStart = (e: React.TouchEvent) => {
    if (shouldScroll) {
      handleScrollTouchStart(e.touches[0].clientY);
    } else {
      handleTouchStart(e);
    }
  };

  const handleTouchMove = (e: React.TouchEvent) => {
    if (shouldScroll) {
      handleScrollTouchMove(e.touches[0].clientY);
    } else {
      handleTouchMove(e);
    }
  };

  const handleTouchEnd = () => {
    if (shouldScroll) {
      handleScrollTouchEnd();
    } else {
      handleTouchEnd();
    }
  };

  return (
    <div
      className="photo-view-overlay"
      onClick={hide}
      onTouchStart={handleTouchStart}
      onTouchMove={handleTouchMove}
      onTouchEnd={handleTouchEnd}
    >
      <img
        className="photo-view-image"
        src={currentImage?.src}
        alt=""
        onClick={(e) => e.stopPropagation()}
        onLoad={handleImageLoad}
        style={{
          // 长图时添加 transform
          transform: shouldScroll ? `translateY(${translateY}px)` : undefined,
          // 滚动时移除过渡动画，实现实时跟随
          transition: isLoaded ? 'transform 0s' : undefined,
        }}
      />

      <div className="photo-view-close" onClick={hide}>
        ×
      </div>
      <div className="photo-view-counter">
        {index + 1} / {images.length}
      </div>
    </div>
  );
}
```

---

## 阶段四：惯性滚动（让滚动更自然）

### Step 18: 创建惯性滚动 Hook `src/hooks/useInertialScroll.ts`

```typescript
// src/hooks/useInertialScroll.ts
import { useRef, useCallback } from 'react';

interface Velocity {
  x: number;
  y: number;
}

export default function useInertialScroll(
  currentTranslateY: number,
  setTranslateY: (y: number) => void,
  minY: number,
  maxY: number,
) {
  const lastY = useRef(0);
  const lastTime = useRef(0);
  const rafId = useRef<number | null>(null);

  // 计算速度
  const calculateVelocity = useCallback((clientY: number, timestamp: number): Velocity => {
    const dy = clientY - lastY.current;
    const dt = timestamp - lastTime.current;

    lastY.current = clientY;
    lastTime.current = timestamp;

    return {
      x: 0,
      y: dt > 0 ? dy / dt : 0,
    };
  }, []);

  // 惯性动画
  const startInertialScroll = useCallback(
    (velocity: Velocity) => {
      const { y: velocityY } = velocity;

      // 速度太小时不触发惯性
      if (Math.abs(velocityY) < 0.1) return;

      let currentVelocity = velocityY * 15; // 速度系数
      const friction = 0.95; // 摩擦系数

      const animate = () => {
        if (Math.abs(currentVelocity) < 0.1) {
          // 动画结束，吸附到边界
          if (currentTranslateY > maxY) {
            setTranslateY(maxY);
          } else if (currentTranslateY < minY) {
            setTranslateY(minY);
          }
          return;
        }

        const newY = currentTranslateY + currentVelocity;

        // 边缘回弹效果
        if (newY > maxY) {
          setTranslateY(maxY + (newY - maxY) * 0.3);
          currentVelocity *= -0.5; // 反弹
        } else if (newY < minY) {
          setTranslateY(minY + (newY - minY) * 0.3);
          currentVelocity *= -0.5; // 反弹
        } else {
          setTranslateY(newY);
        }

        currentVelocity *= friction;
        rafId.current = requestAnimationFrame(animate);
      };

      rafId.current = requestAnimationFrame(animate);
    },
    [currentTranslateY, setTranslateY, minY, maxY],
  );

  // 停止惯性滚动
  const stopInertialScroll = useCallback(() => {
    if (rafId.current) {
      cancelAnimationFrame(rafId.current);
      rafId.current = null;
    }
  }, []);

  return {
    calculateVelocity,
    startInertialScroll,
    stopInertialScroll,
  };
}
```

---

## 阶段五：添加动画效果

### Step 19: 创建动画 Hook `src/hooks/useAnimation.ts`

```typescript
// src/hooks/useAnimation.ts
import { useState, useCallback } from 'react';

type AnimationType = 'fade' | 'scale' | 'slide';

interface AnimationState {
  isAnimating: boolean;
  animationType: AnimationType | null;
}

export default function useAnimation(duration: number = 300) {
  const [state, setState] = useState<AnimationState>({
    isAnimating: false,
    animationType: null,
  });

  // 开始动画
  const startAnimation = useCallback((type: AnimationType) => {
    setState({ isAnimating: true, animationType: type });
  }, []);

  // 动画结束
  const endAnimation = useCallback(() => {
    setState({ isAnimating: false, animationType: null });
  }, []);

  return {
    ...state,
    startAnimation,
    endAnimation,
    duration,
  };
}
```

### Step 20: 添加 CSS 动画

更新 `src/index.css`：

```css
/* 入场动画 */
@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.8);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* 退场动画 */
@keyframes fadeOut {
  from {
    opacity: 1;
  }
  to {
    opacity: 0;
  }
}

.photo-view-overlay {
  animation: fadeIn 0.3s ease-out;
}

.photo-view-overlay.closing {
  animation: fadeOut 0.3s ease-out;
}

.photo-view-image {
  animation: scaleIn 0.3s ease-out;
}
```

---

## 阶段六：完善与优化

### Step 21: 添加 Portal 支持

```typescript
// src/components/PhotoPortal.tsx
import React from 'react';
import { createPortal } from 'react-dom';

interface PortalProps {
  children: React.ReactNode;
  container?: HTMLElement;
}

export default function PhotoPortal({ children, container }: PortalProps) {
  const mountNode = container || document.body;
  return createPortal(children, mountNode);
}
```

### Step 22: 导出入口文件 `src/index.ts`

```typescript
// src/index.ts
export { default as PhotoProvider } from './PhotoProvider';
export { default as PhotoView } from './PhotoView';
export { default as PhotoSlider } from './PhotoSlider';
export type { PhotoItem } from './types';
```

---

## 最终项目结构

完成所有步骤后，你的项目结构如下：

```
my-photo-view/
├── src/
│   ├── index.ts                    # 导出入口
│   ├── index.css                   # 样式
│   ├── types.ts                    # 类型定义
│   ├── photo-context.ts            # Context
│   │
│   ├── PhotoProvider.tsx           # 状态管理 ⭐
│   ├── PhotoView.tsx              # 单图组件 ⭐
│   ├── PhotoSlider.tsx            # 预览主组件 ⭐
│   │
│   ├── components/
│   │   └── PhotoPortal.tsx        # Portal 容器
│   │
│   ├── hooks/
│   │   ├── useMethods.ts           # 持久化方法 ⭐
│   │   ├── useSetState.ts          # 类 setState ⭐
│   │   ├── useInitial.ts           # 初始化一次 ⭐
│   │   ├── useSwipe.ts             # 滑动切换
│   │   ├── useLongImageScroll.ts   # 长图滚动 ⭐⭐
│   │   ├── useInertialScroll.ts    # 惯性滚动
│   │   └── useAnimation.ts          # 动画
│   │
│   └── utils/
│       └── detectImageType.ts      # 图片类型检测
│
├── package.json
└── tsconfig.json
```

---

## 快速回顾

| 步骤  | 文件                       | 核心功能        |
| ----- | -------------------------- | --------------- |
| 1-2   | types.ts, photo-context.ts | 基础类型定义    |
| 3-6   | hooks/                     | 基础工具 Hooks  |
| 7-8   | PhotoProvider, PhotoView   | 核心组件        |
| 9-12  | PhotoSlider 基础版         | 基础预览功能    |
| 13-14 | useSwipe                   | 手势滑动        |
| 15-17 | useLongImageScroll         | **长图滚动** ⭐ |
| 18    | useInertialScroll          | 惯性滚动        |
| 19-20 | useAnimation               | 动画效果        |
| 21-22 | Portal, index.ts           | 完善与导出      |

---

按这个顺序一步步来，你一定能实现完整的功能！加油！🚀
