# C++ 工程师转型 AI Infra 两个月突击计划 (终极版 v3)

> **核心策略**：利用 9 年 C++ 功力攻克底层系统，用 Python 补齐业务逻辑。
> **差异化定位**：不做只会调 API 的应用工程师，做能修改源码、榨干硬件性能的 **AI 系统工程师**。
> **目标岗位**：AI System Engineer、LLM 推理优化工程师、AI Infra 工程师
> **保底岗位**：LLM 应用开发工程师（需准备独立简历）

---

## 你的核心优势 (The "Hook")

```
✓ C++ (9年) → 能直接修改 llama.cpp/vLLM 底层源码 (90% 转行者做不到)
✓ 硬件 (Mac M2 + 3060) → 覆盖 ARM NEON 和 NVIDIA CUDA 双架构优化
✓ 深度理解 → 不仅仅会用，还能通过 Profiling 告诉面试官"为什么慢"、"怎么变快"
```

---

## ⚠️ 开始前必做：确认硬件配置

```bash
# 在 Windows 上运行
nvidia-smi

# 查看显存大小
# 12GB → 台式机 3060，可以微调 7B/8B 模型
# 6GB  → 笔记本 3060，只能微调 3B 以下
```

**本计划假设你是 12GB 版本**（台式机）。如果是 6GB，Week 6 微调部分需要调整模型大小。

---

## 硬件分工

| 设备 | 用途 | 关键任务 |
|-----|------|------|
| **Mac M2 (16GB)** | 源码研读、SIMD 优化 | llama.cpp 探针修改、NEON 向量加速 |
| **Windows 3060 12GB (WSL2)** | GPU 编程、服务部署 | CUDA/Triton 算子、vLLM 调优、7B 微调 |

**WSL2 环境准备清单**（Week 4 之前完成）：
```bash
1. WSL2 安装 Ubuntu 22.04
2. 安装 CUDA Toolkit 12.x
3. 安装 nvidia-container-toolkit
4. 验证：nvidia-smi 和 nvcc --version 正常输出
```

---

## 阶段总览

| 阶段 | 周期 | 核心任务 | 关键产出 | 求职节点 |
|-----|------|------|------|------|
| **P1: 底层引擎魔改** | Week 1-2 | 深入 llama.cpp 源码，植入性能探针 | 带算子级监控的推理引擎 | - |
| **P2: 业务层 + SIMD** | Week 3 | RAG 系统 + **C++ SIMD 扩展** | 高性能 RAG + C++ 加速库 | **简历 B 投应用岗** |
| **P3: GPU 编程与系统** | Week 4-6 | CUDA/Triton + vLLM + 7B 微调 | Triton 算子 + 高并发服务 | **简历 A 投 Infra 岗** |
| **P4: 闭环与交付** | Week 7-8 | 项目完善 + 简历重构 + 面试 | 系统级简历 + Offer | **全面冲刺** |

---

## Phase 1: 底层引擎魔改 (Week 1-2)

### Week 1: 深入 llama.cpp (Mac 为主)
**目标**：脱离 Python，建立对 LLM 推理链路的 C++ 级认知。

| Day | 任务 | 硬核产出 |
|-----|------|------|
| Day 1 | 编译与环境 | Mac (Metal) + WSL2 (CUDA) 双端编译通过 |
| Day 2 | **GGUF 逆向分析** | 画出 GGUF 文件二进制结构图（Header, KV Pairs, Tensor Offsets） |
| Day 3 | `llama_decode` 追踪 | 绘制 C++ 调用堆栈图，定位到 GEMM 发生的具体行数 |
| Day 4 | **量化底层逻辑** | 阅读 `ggml-quants.c`，理解 Q4_K_M 的 block-wise 存储结构 |
| Day 5 | KV Cache 探秘 | 计算并打印当前 Context 使用的 KV Cache 显存大小 |
| Day 6 | **examples/simple 精读** | 逐行注释 simple.cpp，理解最小推理流程 |
| Day 7 | 总结与笔记 | 整理 Week 1 技术笔记：*《C++ 视角下的 LLM 推理链路解析》* |

**Week 1 必须能回答的问题**：
```
- [ ] llama_model 和 llama_context 有什么区别？
- [ ] GGUF 文件的内存映射 (mmap) 是怎么工作的？
- [ ] Q4_K_M 量化的 block size 是多少？
- [ ] KV Cache 的 Shape 是什么？
```

---

### Week 2: 源码修改与性能探针 (关键差异化)
**目标**：不做简单的 API Wrapper，而是做**算子级监控**。

| Day | 任务 | 硬核产出 |
|-----|------|------|
| Day 1 | **植入性能探针** | 修改 `src/llama.cpp`，在 Attention 和 FFN 模块前后插入计时代码 (`std::chrono`) |
| Day 2 | **层级耗时分析** | 统计并打印每一层 Transformer 的 Attention vs FFN 耗时占比 |
| Day 3 | 采样器 (Sampler) 分析 | 阅读 `llama_sampler` 相关代码，理解 top_k/top_p 实现 |
| Day 4 | Python Binding 原理 | 理解 `ctypes`/`cffi` 如何跨语言传递指针，跑通 llama-cpp-python |
| Day 5-6 | **[项目 1] 可观测推理引擎** | C++ Core + Python Wrapper |
| Day 7 | 项目完善 + 文档 | README + 性能数据 |

**项目 1 规格**：
```
可观测推理引擎
├── C++ Core：修改 llama.cpp，添加算子计时
├── Python API：简单的 HTTP 接口
├── 性能指标：返回 X-Latency-Attn、X-Latency-FFN header
├── 文档：算子耗时分析报告
└── 亮点：能说出 "7B 模型在 M2 上 Attention 占比约 XX%"
```

**项目 1 面试话术**：
> "我不仅仅部署了模型，我修改了 llama.cpp 源码，实现了细粒度的算子性能监控，发现 7B 模型在 M2 上 Attention 计算占比约为 XX%，FFN 占比约为 YY%。"

---

## Phase 2: 业务层 + SIMD 加速 (Week 3)

### Week 3: RAG 系统 + C++ SIMD 扩展（核心差异化）
**目标**：快速掌握应用层，同时用 C++ 扩展展示降维打击能力。

> ⚠️ **重要**：C++ 扩展是**必做项**，不是可选。这是你区别于 Python 工程师的核武器。

| Day | 任务 | 产出 |
|-----|------|------|
| Day 1 | RAG 原理 + LangChain 快速入门 | 跑通基础 RAG demo |
| Day 2 | 向量数据库 + Embedding | Chroma 搭建 + bge-m3 |
| Day 3 | **SIMD 基础复习** | AVX2 (x86) / NEON (ARM) 基础概念 |
| Day 4 | **C++ 扩展开发（必做）** | 用 pybind11 + xsimd 写向量点积/余弦相似度 |
| Day 5 | **性能 Benchmark** | 对比 Python Loop vs Numpy vs 你的 C++ 扩展 |
| Day 6 | **[项目 2] 高性能 RAG** | 集成 C++ 扩展 + 完整 RAG 系统 |
| Day 7 | **简历 B + 投递应用岗** | 准备保底简历并开始投递 |

**C++ 扩展实现方案**：
```
使用 xsimd 库（推荐）：
- 跨平台 SIMD 抽象库
- 写起来像正常 C++，自动适配 AVX2/NEON
- 比手写 intrinsics 简单 10 倍

示例代码结构：
├── src/
│   └── simd_ops.cpp      ← xsimd 实现的点积/余弦相似度
├── python/
│   └── bindings.cpp      ← pybind11 绑定
├── benchmark.py          ← 性能对比脚本
└── CMakeLists.txt
```

**项目 2 规格**：
```
高性能 RAG 知识库
├── 文档解析：PDF/Word/Markdown
├── 向量存储：Chroma
├── 检索：向量检索 + 你的 C++ SIMD 加速
├── 生成：支持 OpenAI / 本地 llama.cpp 切换
├── UI：Streamlit
├── **核心亮点**：C++ 扩展 + Benchmark 对比图
└── 性能数据：Python vs Numpy vs C++ SIMD
```

**项目 2 面试话术（价值千金）**：
> "Designed a custom C++ extension using pybind11 and xsimd (AVX2/NEON), accelerating vector similarity computation by **50x** compared to pure Python and **3-5x** compared to NumPy."

**Week 3 检查点**：
```
- [ ] C++ SIMD 扩展完成（必做！）
- [ ] RAG 项目完成
- [ ] Benchmark 对比数据
- [ ] 简历 B（应用版）完成
- [ ] 开始投递应用岗（保底）
```

---

## Phase 3: GPU 编程与系统优化 (Week 4-6)

### Week 4: CUDA 与 Triton (3060 为主)
**目标**：掌握 GPU 编程，Triton 是 Infra 岗的"显学"。

| Day | 任务 | 硬核产出 |
|-----|------|------|
| Day 1 | GPU 架构基础 | 理解 SM、Warp、HBM、Shared Memory，运行 `deviceQuery` |
| Day 2 | **Hello CUDA** | 手写 CUDA C++ 向量加法 (Vector Add) |
| Day 3 | **手写矩阵乘法** | 实现 Naive GEMM，理解为什么需要 Tiling |
| Day 4 | **Triton 入门** | 学习 Triton 语法，用 Triton 重写 Vector Add |
| Day 5 | **Triton 进阶** | 用 Triton 实现 Softmax |
| Day 6 | Profiling | 使用 Nsight Systems 分析你的 Kernel |
| Day 7 | 技术博客 | 撰写 *《从 CUDA 到 Triton：C++ 工程师的算子开发之路》* |

**CUDA 核心概念清单**：
```
必须理解：
- [ ] Grid → Block → Thread 层级
- [ ] Global Memory vs Shared Memory vs Register
- [ ] Memory Coalescing（合并访问）
- [ ] Warp Divergence（分支发散）
- [ ] Bank Conflict（Shared Memory）

能回答：
- 3060 有多少 SM？每个 SM 多少 CUDA Core？
- 为什么 GEMM 需要 Tiling？
- Triton 相比 CUDA C++ 的优势是什么？
```

---

### Week 5: vLLM 与高并发服务
**目标**：掌握企业级推理系统，这是 Infra 岗高频考点。

| Day | 任务 | 硬核产出 |
|-----|------|------|
| Day 1 | vLLM 部署 | 在 3060 上部署 vLLM，跑通基础推理 |
| Day 2 | **PagedAttention 论文** | 精读论文，理解 KV Cache 分页管理 |
| Day 3 | **Continuous Batching** | 编写脚本观察动态批处理行为 |
| Day 4 | **源码分析：scheduler.py** | 阅读调度器核心逻辑 |
| Day 5 | **3060 极限调优** | 调整 `gpu-memory-utilization`、`max-model-len` |
| Day 6-7 | **[项目 3] 高性能推理服务** | vLLM 服务 + 压测报告 |

**项目 3 规格**：
```
高性能推理服务
├── 推理后端：vLLM（3060 12GB 调优版）
├── API：OpenAI Compatible
├── 压测：QPS vs Latency 曲线
├── 文档：调优过程 + 参数选择理由
└── 对比：vLLM vs llama.cpp 性能对比
```

**vLLM 必须能回答的问题**：
```
- [ ] PagedAttention 解决什么问题？
- [ ] Continuous Batching vs Static Batching 区别？
- [ ] KV Cache 显存怎么计算？公式是什么？
- [ ] gpu-memory-utilization 设多少合适？为什么？
```

**Week 5 检查点**：
```
- [ ] vLLM 部署成功
- [ ] 能解释 PagedAttention
- [ ] 项目 3 完成
- [ ] 简历 A（Infra 版）完成
- [ ] 开始投递 Infra 岗（主攻）
```

---

### Week 6: 微调全链路（7B/8B 模型）
**目标**：在主流模型尺寸上完成微调闭环。

> ⚠️ **重要**：必须在 7B/8B 这个主流尺寸上微调，而不是 1.5B/3B 玩具模型。12GB 显存 + QLoRA 完全够用。

| Day | 任务 | 产出 |
|-----|------|------|
| Day 1 | LoRA/QLoRA 原理 | 理解低秩分解 + 量化训练 |
| Day 2 | **Unsloth 源码赏析** | 看 Triton Kernel 实现（RoPE、RMSNorm），连接 Week 4 |
| Day 3 | **数据准备** | 收集/清洗垂直领域数据 500-1000 条 |
| Day 4-5 | **微调实战** | Unsloth + QLoRA 微调 **Qwen2.5-7B** 或 **Llama-3-8B** |
| Day 6 | **导出 GGUF** | LoRA 融合 + GGUF 导出 + llama.cpp 验证 |
| Day 7 | **[项目 4] 评估报告** | Base vs Fine-tuned 对比 + 完整文档 |

**3060 12GB 微调配置参考**：
```python
# Unsloth + QLoRA 配置
model = FastLanguageModel.from_pretrained(
    model_name="unsloth/Qwen2.5-7B",
    max_seq_length=2048,
    load_in_4bit=True,  # QLoRA 关键
)

# LoRA 配置
model = FastLanguageModel.get_peft_model(
    model,
    r=16,              # LoRA rank
    lora_alpha=16,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
)

# 训练配置
training_args = TrainingArguments(
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,
    # ... 
)
```

**项目 4 规格**：
```
垂直领域微调模型（7B 级别）
├── 数据：500-1000 条领域数据
├── 基座：Qwen2.5-7B 或 Llama-3-8B
├── 方法：Unsloth + QLoRA（4bit）
├── 硬件：RTX 3060 12GB
├── 导出：GGUF Q4_K_M
├── 部署：llama.cpp 推理
├── 评估：Base vs Fine-tuned 对比数据
└── 文档：显存占用、训练速度、参数选择
```

**项目 4 面试话术**：
> "使用 Unsloth + QLoRA 在消费级显卡（RTX 3060 12GB）上成功微调 Qwen2.5-7B 模型。通过 4bit 量化和梯度累积，在有限显存下完成了主流尺寸模型的全链路训练，领域任务准确率提升 XX%。"

---

## Phase 4: 简历与面试冲刺 (Week 7-8)

### Week 7: 项目完善与复盘

| Day | 任务 | 产出 |
|-----|------|------|
| Day 1-2 | 项目代码重构 + 完善 README | 4 个项目全部整理完成 |
| Day 3 | **源码复盘**：llama_decode 逐行注释 | 源码分析文档 |
| Day 4 | **源码复盘**：vLLM scheduler.py 逐行注释 | 源码分析文档 |
| Day 5 | System Design 练习 | 设计文档 |
| Day 6-7 | 技术博客 2-3 篇 + 简历定稿 | 最终版简历 A 和 B |

**System Design 必练题**：
```
1. 如何设计支持百万用户的 ChatGPT 系统？
   关键词：负载均衡、KV Cache Offloading、Speculative Decoding

2. 如何在单卡 3060 上运行 70B 模型？
   关键词：量化、CPU Offloading、模型切分

3. 设计一个企业级 RAG 系统
   关键词：索引更新、检索优化、缓存策略
```

---

### Week 8: 全面投递

| Day | 任务 | 产出 |
|-----|------|------|
| Day 1-2 | 大量投递（简历 A 为主） | 目标 50+ 岗位 |
| Day 3-4 | 面试准备 + 模拟面试 | 常见问题整理 |
| Day 5-7 | 面试 + 持续投递 | **拿到 Offer！** |

---

## 双简历策略（关键！）

> ⚠️ **重要**：不要用 Infra 简历投应用岗，会被认为"过于资深/不稳定"

### 简历 A：Infra 专家版
**Title**：AI System Engineer | C++ High Performance Computing

**投递岗位**：AI Infra、推理优化、系统工程师（40-70K）

**技能栏**：
```
Languages: C++ (Expert, 9 yrs), Python (Proficient), CUDA (Basic)
AI Infra: llama.cpp, vLLM, Triton, GGUF, ggml, pybind11
LLM: Transformer, Attention, KV Cache, Quantization, LoRA
Optimization: SIMD (AVX2/NEON), Memory Management, Profiling
```

**项目描述**：
1. **可观测推理引擎**：深入研究 llama.cpp 源码，修改 `llama_decode` 核心路径，实现微秒级算子性能探针...
2. **高性能 RAG + C++ 加速**：Designed a custom C++ extension using pybind11 and xsimd, accelerating vector operations by 50x...
3. **企业级推理服务**：基于 vLLM 构建高并发服务，深入理解 PagedAttention...
4. **7B 模型微调**：在消费级显卡上微调主流尺寸模型...

---

### 简历 B：全栈应用版
**Title**：LLM Application Engineer | Full Stack AI Developer

**投递岗位**：LLM 应用开发、AI 后端（20-35K，保底）

**技能栏**：
```
Languages: Python (Proficient), C++ (Expert)
LLM: LangChain, LlamaIndex, RAG, Prompt Engineering
Backend: FastAPI, Docker, REST API
Database: Vector DB (Chroma), PostgreSQL
```

**项目描述**（隐藏底层细节，强调业务）：
1. **企业知识库问答系统**：基于 LangChain + Chroma 构建，支持多格式文档，实现毫秒级检索...（不提 C++ 扩展细节）
2. **LLM 推理服务**：部署和优化 LLM 推理服务，支持高并发访问...（不提源码修改）
3. **垂直领域对话模型**：微调领域模型，提升业务场景准确率...（简化描述）

**关键差异**：
- 隐藏 CUDA/Triton/汇编细节
- 强调业务理解和产品落地能力
- 把"源码魔改"说成"解决复杂性能问题"

---

## 投递策略

### Phase 1（Week 3-5）：保底先行
```
简历 B → 应用开发岗 → 先拿面试机会练手
目标：拿到 1-2 个保底 Offer
```

### Phase 2（Week 5-8）：主攻 Infra
```
简历 A → Infra/推理优化岗 → 争取高薪岗位
目标：40K+ Offer
```

### 投递顺序建议
```
Week 3: 简历 B 投 10-20 个应用岗（练手）
Week 5: 简历 A 投 20-30 个 Infra 岗（主攻）
Week 6+: 根据反馈调整，两个简历并行投递
```

---

## 面试准备清单

### C++ 基础（你的基本盘）
```
- 智能指针：shared_ptr/unique_ptr/weak_ptr 区别和实现
- 内存管理：RAII、内存池、mmap
- 多线程：mutex、condition_variable、atomic、lock-free
- 现代 C++：move 语义、完美转发、SFINAE
```

### LLM 原理
```
- Transformer 完整结构描述
- Self-Attention 计算过程和复杂度 O(n²d)
- KV Cache 作用和显存计算公式
- 量化原理：FP16 → INT4 的过程
- LoRA 原理：为什么低秩分解有效
```

### CUDA/GPU
```
- Thread/Block/Grid 映射关系
- Global vs Shared vs Register Memory
- Memory Coalescing 原理
- Warp Divergence 问题
- Triton vs CUDA C++ 的优劣
```

### 系统设计
```
- PagedAttention 解决什么问题？
- Continuous Batching vs Static Batching？
- 如何处理长上下文？
- 模型并行 (TP) vs 流水线并行 (PP)？
- Speculative Decoding 原理？
```

---

## 每周检查清单

### Week 1 ✓
- [ ] llama.cpp Mac + WSL2 双端编译通过
- [ ] GGUF 结构图绘制完成
- [ ] 能画出 llama_decode 调用链
- [ ] 能解释 Q4_K_M 量化原理

### Week 2 ✓
- [ ] 性能探针代码完成
- [ ] 项目 1 完成并上传 GitHub
- [ ] 能说出 Attention vs FFN 耗时占比

### Week 3 ✓
- [ ] **C++ SIMD 扩展完成（必做！）**
- [ ] RAG 项目完成 + Benchmark 数据
- [ ] 简历 B 完成
- [ ] 开始投递应用岗

### Week 4 ✓
- [ ] CUDA Vector Add + GEMM 跑通
- [ ] Triton Vector Add + Softmax 跑通
- [ ] 能解释 GPU 内存层次

### Week 5 ✓
- [ ] vLLM 部署成功
- [ ] 能解释 PagedAttention
- [ ] 项目 3 完成
- [ ] 简历 A 完成
- [ ] 开始投递 Infra 岗

### Week 6 ✓
- [ ] 微调数据准备完成
- [ ] **7B/8B 模型微调成功**
- [ ] GGUF 导出成功
- [ ] 项目 4 完成

### Week 7 ✓
- [ ] 4 个项目 README 全部完善
- [ ] 技术博客 2-3 篇
- [ ] 简历 A/B 定稿

### Week 8 ✓
- [ ] 投递 50+ 岗位
- [ ] 完成模拟面试
- [ ] **拿到 Offer！**

---

## 学习资源汇总

### 必读
```
书：《PMPP》前 5 章（CUDA 编程圣经）
论文：vLLM (PagedAttention)、LoRA
代码：llama.cpp、vLLM、Unsloth
```

### 推荐
```
视频：Andrej Karpathy "Let's build GPT"
博客：PyTorch Blog、vLLM Blog、OpenAI Triton Blog
教程：https://triton-lang.org/main/getting-started/tutorials/
```

### 重点阅读文件
```
llama.cpp:
- examples/simple/simple.cpp（入口）
- src/llama.cpp（llama_decode 函数）
- ggml/src/ggml-quants.c（量化实现）

vLLM:
- vllm/core/scheduler.py（调度器）
- vllm/attention/backends/（Attention 实现）

Unsloth:
- unsloth/kernels/（Triton Kernel）
```

### SIMD 相关
```
xsimd 文档：https://xsimd.readthedocs.io/
pybind11 文档：https://pybind11.readthedocs.io/
```

---

祝你转型顺利！C++ + Infra 双刷子，AI 时代不可替代。
