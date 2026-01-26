# Day 3: llama_decode 调用堆栈追踪

> 学习日期：2026-01-26
> 目标：绘制 C++ 调用堆栈图，定位到 GEMM 发生的具体行数

---

## 一、调用堆栈总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        llama_decode 调用链                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  examples/simple/simple.cpp                                             │
│  └── Line 170: llama_decode(ctx, batch)                                 │
│                     │                                                   │
│                     ▼                                                   │
│  src/llama-context.cpp                                                  │
│  └── Line 3527: llama_decode() → ctx->decode(batch)                     │
│                     │                                                   │
│                     ▼                                                   │
│  src/llama-context.cpp                                                  │
│  └── Line 1459: llama_context::decode()                                 │
│                     │                                                   │
│                     ▼                                                   │
│  src/llama-context.cpp                                                  │
│  └── Line 1618: process_ubatch()                                        │
│           ├── Line 1150: 图构建 (build_graph)                            │
│           └── Line 1176: 图计算 (graph_compute)                          │
│                     │                                                   │
│                     ▼                                                   │
│  src/llama-model.cpp                                                    │
│  └── Line 7623: llama_model::build_graph()                              │
│           └── 根据模型架构选择具体的 builder                               │
│               └── llm_build_qwen2 / llm_build_llama / ...               │
│                     │                                                   │
│                     ▼                                                   │
│  src/llama-graph.cpp                                                    │
│  └── build_attn_mha() ← Attention 计算核心                               │
│           ├── Line 1655: kq = ggml_mul_mat(k, q)     ← GEMM: K×Q        │
│           ├── Line 1689: soft_cap / softmax                             │
│           └── Line 1699: kqv = ggml_mul_mat(v, kq)   ← GEMM: V×KQ       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、关键文件与行号

请填写你今天追踪到的具体行号：

| 步骤 | 文件 | 行号 | 函数/操作 |
|------|------|------|-----------|
| 1 | examples/simple/simple.cpp | 170 | llama_decode() 调用 |
| 2 | src/llama-context.cpp | 3527 | llama_decode() 定义 |
| 3 | src/llama-context.cpp | 1459 | llama_context::decode() |
| 4 | src/llama-context.cpp | 1618 | process_ubatch() |
| 5 | src/llama-context.cpp | 1150 | 图构建调用 |
| 6 | src/llama-context.cpp | 1176 | 图计算调用 |
| 7 | src/llama-model.cpp | 7623 | llama_model::build_graph() |
| 8 | src/llama-graph.cpp | 1655 | GEMM: K×Q |
| 9 | src/llama-graph.cpp | 1699 | GEMM: V×KQ |

---

## 三、核心数据结构

### 3.1 llama_batch

```cpp
// 管理batch,token数量，位置，嵌入向量值
struct llama_batch {
    // ...
};
```

### 3.2 llama_context

```cpp
// 是一个抽象对象，用来管理当前模型，当前LLM的各种状态值
```

### 3.3 ubatch (micro-batch)

为什么要把 batch 拆分成 ubatch？

因为为了控制批处理大小，由于内存显存处理的空间的限制，将batch拆分成多个小batch，然后并行处理，可以高效执行。


---

## 四、Attention 计算流程

### 4.1 Self-Attention 公式

```
Attention(Q, K, V) = softmax(Q × K^T / √d_k) × V
```

### 4.2 在代码中的对应

| 数学公式 | 代码位置 | 函数调用 |
|----------|----------|----------|
| Q × K^T | llama-graph.cpp Line 1655 | ggml_mul_mat(k, q) |
| / √d_k | 通过 kq_scale 参数 | |
| softmax | llama-graph.cpp Line 1689 | ggml_soft_max_ext() |
| × V | llama-graph.cpp Line 1699 | ggml_mul_mat(v, kq) |

### 4.3 Soft-Capping（可选特性）

```
output = cap × tanh(input / cap)
```

作用: 数据软截断，避免数值过大导致计算偏差过大

## 五、计算图执行

### 5.1 图构建 vs 图执行

| 阶段 | 描述 | 关键函数 |
|------|------|----------|
| 构建 | 定义计算操作（不执行） | build_graph() |
| 执行 | 实际运行计算 | ggml_backend_graph_compute() |

### 5.2 这种设计的优势

请写出你理解的优势：

1. 便于并行处理计算
2. 利于统一管理计算资源
3. 一次配置，多次使用，加快计算速度

---

## 六、思考问题

1. **为什么 llama.cpp 使用计算图而不是直接计算？**

   你的回答：详见5.2


2. **GEMM 为什么是性能热点？**

   你的回答：因为涉及到大量的矩阵乘法计算


3. **如果要优化推理性能，你会从哪里入手？**

   你的回答：优化矩阵乘法计算方式


---

## 七、下一步

根据学习计划，Day 4 将学习：
- **量化底层逻辑**：阅读 `ggml-quants.c`，理解 Q4_K_M 的 block-wise 存储结构

---

## 八、深入理解：计算图原理

### 8.1 什么是计算图？

计算图是一种**有向无环图 (DAG)**，用来表示计算过程：

```
示例：y = (a + b) * c

         ┌───┐
    a ───┤   │
         │ + ├─── tmp ───┐
    b ───┤   │           │    ┌───┐
         └───┘           ├────┤   │
                         │    │ * ├─── y (输出)
    c ───────────────────┘    │   │
                              └───┘

节点 (Node): 操作 (+, *)
边 (Edge): 数据流向
```

### 8.2 核心数据结构

```c
// ggml_tensor 既是数据也是节点
struct ggml_tensor {
    enum ggml_op op;                         // 操作类型
    struct ggml_tensor * src[GGML_MAX_SRC];  // 输入依赖
    void * data;                             // 数据指针
};

// ggml_cgraph 是计算图
struct ggml_cgraph {
    struct ggml_tensor ** nodes;  // 所有计算节点
    int n_nodes;                  // 节点数量
};
```

### 8.3 两阶段执行

| 阶段 | 函数 | 作用 |
|------|------|------|
| 构建 | `ggml_mul_mat(...)` | 创建节点，设置 op 类型，不计算 |
| 执行 | `ggml_backend_graph_compute()` | 遍历图，调用每个 op 的 kernel |

### 8.4 cb 回调函数

`cb(tensor, "name", il)` 只是给张量命名，方便调试，**不执行计算**。

---

## 九、算子 (Operator)

### 9.1 定义

**算子 = 一个基本的计算操作**，作用于张量。

### 9.2 常见算子

| 算子名 | 数学表示 | 说明 |
|--------|----------|------|
| ADD | a + b | 加法 |
| MUL | a * b | 逐元素乘法 |
| **MUL_MAT** | A × B | **矩阵乘法 (GEMM)** |
| SOFT_MAX | softmax(x) | Softmax |
| ROPE | 旋转位置编码 | RoPE |

### 9.3 LLM 推理中的算子耗时

| 算子 | 耗时占比 | 说明 |
|------|---------|------|
| **MUL_MAT (GEMM)** | 80-90% | 性能瓶颈！ |
| SOFT_MAX | 5-10% | Attention 中的 Softmax |
| 其他 | <5% | ROPE, ADD, MUL 等 |

### 9.4 算子开发

**算子开发 = 将公式在特定硬件上高效实现**

```
数学公式 → Naive 实现 → SIMD 优化 → CUDA/GPU 优化
```

---

## 十、Graph Plan（执行计划）

### 10.1 作用

`plan` 是执行前的**预处理**，包含：
- 工作内存分配
- 线程任务分配
- 依赖关系分析

### 10.2 优势

| 优势 | 说明 |
|------|------|
| 减少延迟 | 推理时不需要重复分析 |
| 内存复用 | 临时缓冲区跨调用复用 |
| 优化机会 | 首次创建时做深度优化 |

### 10.3 使用方式

```cpp
// 首次：创建 plan
plan = ggml_backend_graph_plan_create(backend, graph);

// 后续：复用 plan 执行（快速）
for (int i = 0; i < num_tokens; i++) {
    ggml_backend_graph_plan_compute(backend, plan);
}
```

---

## 十一、异构计算

### 11.1 概念

**异构 = 不同类型处理器协同工作** (CPU + GPU + NPU)

### 11.2 核心挑战

| 挑战 | 说明 |
|------|------|
| 内存隔离 | CPU 和 GPU 有独立的内存空间 |
| 数据传输 | PCIe 带宽是瓶颈 (~32 GB/s vs GPU 内部 ~1000 GB/s) |
| 任务调度 | 决定哪些操作交给哪个设备 |

### 11.3 ggml 的实现

```
┌─────────────────────────────────────────────────────────────┐
│                      Backend 抽象层                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   backend_cpu   ←───┐                                        │
│   backend_cuda  ←───┼─── 统一接口: graph_compute()          │
│   backend_metal ←───┘                                        │
│                                                              │
│   Scheduler (调度器):                                        │
│   - 分析每个算子适合哪个后端                                  │
│   - 自动插入数据传输                                         │
│   - 优化调度减少数据搬运                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 11.4 典型调度

```
Token Embedding  → CPU (从磁盘加载)
        ↓ 传输到 GPU
Transformer      → GPU (大量 GEMM)
        ↓ 传输回 CPU
采样             → CPU (小操作)
```

### 11.5 关键优化

- 减少传输：尽量让数据留在 GPU
- 异步传输：传输和计算重叠
- 算子融合：减少中间结果传输

---

## 十二、总结

### 核心概念

1. **计算图**：声明式表示计算，先构建后执行
2. **算子**：基本计算操作，GEMM 是性能关键
3. **Graph Plan**：执行计划预计算，提高复用效率
4. **异构计算**：多设备协同，调度器自动管理

### ggml 设计亮点

- 统一的 Backend 接口，支持多硬件
- 计算图先构建后执行，支持优化
- Graph Plan 复用减少开销
- Scheduler 自动处理异构调度

---

## 参考资料

- `examples/simple/simple.cpp` - 最简推理示例
- `src/llama-context.cpp` - 上下文和推理入口
- `src/llama-model.cpp` - 模型定义和图构建
- `src/llama-graph.cpp` - Attention 和 FFN 实现
- `ggml/src/ggml-backend.cpp` - 后端抽象和调度器
