# my-photo-view 项目结构参考

学习时，建议按照以下结构创建项目，**与原项目保持一致**，这样方便对照学习：

---

## 完整结构（对照原项目）

```
my-photo-view/
├── src/
│   ├── index.ts                    # 导出入口
│   ├── types.ts                    # 类型定义
│   ├── photo-context.ts            # Context 定义
│   ├── variables.ts                # 常量配置
│   │
│   ├── PhotoProvider.tsx           # 状态管理组件
│   ├── PhotoView.tsx              # 单图组件
│   ├── PhotoSlider.tsx            # 预览主组件
│   ├── Photo.tsx                  # 图片组件（可选）
│   ├── PhotoBox.tsx               # 图片容器（可选）
│   │
│   ├── components/                 # 基础 UI 组件
│   │   ├── CloseIcon.tsx          # 关闭按钮
│   │   ├── ArrowLeft.tsx          # 左箭头
│   │   ├── ArrowRight.tsx         # 右箭头
│   │   ├── Spinner.tsx            # 加载中
│   │   ├── SlidePortal.tsx        # 弹窗容器
│   │   └── PreventScroll.tsx      # 阻止滚动
│   │
│   ├── hooks/                     # 自定义 Hooks
│   │   ├── useMethods.ts          # 持久化方法 ⭐
│   │   ├── useSetState.ts         # 类 setState
│   │   ├── useInitial.ts          # 初始化一次
│   │   ├── useEventListener.ts    # 事件监听
│   │   ├── useScrollPosition.ts   # 滚动位置 ⭐
│   │   ├── useTargetScale.ts     # 缩放计算
│   │   ├── useContinuousTap.ts    # 连续点击
│   │   ├── useDebounceCallback.ts # 防抖
│   │   ├── useAnimationVisible.tsx # 显示动画
│   │   ├── useAnimationPosition.ts # 位置动画
│   │   ├── useAnimationOrigin.tsx  # 来源动画
│   │   ├── useAdjacentImages.ts   # 邻近图片
│   │   ├── useForkedVariable.ts  # 分叉变量
│   │   ├── useMountedRef.ts       # 挂载引用
│   │   └── useIsomorphicLayoutEffect.ts # SSR 兼容
│   │
│   ├── utils/                     # 工具函数
│   │   ├── edgeHandle.ts          # 边缘检测
│   │   ├── getPositionOnMoveOrScale.ts # 位置计算 ⭐
│   │   ├── getRotateSize.ts       # 旋转尺寸
│   │   ├── getSuitableImageSize.ts # 自适应尺寸
│   │   ├── getMultipleTouchPosition.ts # 多指位置
│   │   ├── isTouchDevice.ts       # 触摸检测
│   │   └── limitTarget.ts         # 数值限制
│   │
│   └── styles/                    # 样式文件
│       ├── PhotoSlider.less
│       ├── PhotoBox.less
│       ├── Photo.less
│       └── components/
│           ├── SlidePortal.less
│           └── Spinner.less
│
├── package.json
├── tsconfig.json
└── vite.config.ts (或 webpack.config.js)
```

---

## 精简结构（学习阶段推荐）

学习时不需要一下创建所有文件，可以按课程进度逐步添加：

### 阶段 1：基础架构

```
my-photo-view/
├── src/
│   ├── index.ts
│   ├── types.ts
│   ├── photo-context.ts
│   │
│   ├── PhotoProvider.tsx      # 第 5 课
│   ├── PhotoView.tsx         # 第 6 课
│   └── PhotoSlider.tsx       # 第 7 课
│
│   └── index.css              # 简单样式
├── package.json
└── App.tsx                    # 测试入口
```

### 阶段 2：添加 Hooks

```
my-photo-view/
├── src/
│   ├── ...                    # 上面的文件
│   │
│   └── hooks/
│       ├── useMethods.ts      # 必选
│       ├── useSetState.ts     # 必选
│       ├── useInitial.ts      # 可选
│       └── useEventListener.ts
```

### 阶段 3：手势交互

```
my-photo-view/
├── src/
│   ├── ...                    # 上面的文件
│   │
│   ├── hooks/
│   │   ├── useScrollPosition.ts   # ⭐ 核心
│   │   ├── useTargetScale.ts     # ⭐ 缩放
│   │   └── useContinuousTap.ts   # 双击
│   │
│   └── utils/
│       ├── edgeHandle.ts          # ⭐ 边缘检测
│       ├── getPositionOnMoveOrScale.ts
│       └── getRotateSize.ts
```

### 阶段 4：动画效果

```
my-photo-view/
├── src/
│   ├── hooks/
│   │   ├── useAnimationVisible.tsx
│   │   ├── useAnimationPosition.ts
│   │   └── useAnimationOrigin.tsx
│   │
│   └── components/
│       └── SlidePortal.tsx
```

### 最终阶段：完整结构

所有文件都创建好，与原项目一致。

---

## 文件创建顺序

建议按照以下顺序创建文件：

| 顺序 | 文件                    | 课程        |
| ---- | ----------------------- | ----------- |
| 1    | `types.ts`              | 第 3 课     |
| 2    | `photo-context.ts`      | 第 4 课     |
| 3    | `PhotoProvider.tsx`     | 第 5 课     |
| 4    | `PhotoView.tsx`         | 第 6 课     |
| 5    | `PhotoSlider.tsx`       | 第 7 课     |
| 6    | `useSwipe.ts` (新建)    | 第 10 课    |
| 7    | `useScroll.ts` (新建)   | 第 12 课    |
| 8    | `useInertial.ts` (新建) | 第 13 课    |
| 9    | `useZoom.ts` (新建)     | 第 14 课    |
| 10   | 动画相关 hooks          | 第 15-16 课 |

---

## 简化版 vs 完整版

### 如果只想快速实现功能

可以创建一个更简单的版本：

```
my-photo-view/
├── src/
│   ├── index.ts
│   ├── PhotoProvider.tsx       # 包含状态管理
│   ├── PhotoView.tsx          # 包含点击逻辑
│   ├── PhotoSlider.tsx        # 包含所有交互
│   │
│   └── hooks/                 # 必要的 hooks
│       ├── useSwipe.ts
│       ├── useScroll.ts
│       └── useZoom.ts
│
│   └── styles/
│       └── index.css
└── package.json
```

### 如果想完整学习

建议完全按照原项目结构，一个文件一个文件对照学习。

---

## 安装依赖

```bash
# 创建项目
npm create vite@latest my-photo-view -- --template react-ts

# 进入目录
cd my-photo-view

# 安装开发依赖
npm install -D less typescript @types/react @types/react-dom

# 安装（用于参考学习）
npm install react-photo-view
```

---

## 对照学习技巧

1. **先看结构**：了解每个文件的职责
2. **先简化实现**：用自己的方式实现基础功能
3. **对照优化**：再看原项目代码，对比差异
4. **逐步深入**：从简单到复杂，从基础到进阶

---

## 总结

| 方案             | 优点               | 缺点               |
| ---------------- | ------------------ | ------------------ |
| **完全照搬**     | 方便对照，不易出错 | 文件多，初学压力大 |
| **精简版**       | 快速上手，聚焦核心 | 需要自己补充功能   |
| **推荐：渐进式** | 循序渐进，课程配套 | 需要按顺序学习     |

建议按照课程大纲的进度，逐步添加文件，这样学习效果最好！
