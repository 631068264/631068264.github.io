---
Coalescedlayout:     post
rewards: false
title:  openAI Triton
categories:
    - AI
tags:
   - 大模型




---

# Torch 简介

**torch 两种模式**

- Eager Mode（即时模式）

  - PyTorch 默认的执行方式，代码按照 Python 方式逐行执行，易于调试和开发。
  - 适合研究和原型开发，但执行速度相对较慢，不能跨设备（如 C++ 部署）。
  - 由于是解释执行，每次调用 forward 都会重新计算计算图，因此不能充分利用编译优化。

- Graph Mode（图模式）
  - 计算图提前构建并优化，提升执行速度，适合部署和优化。
  - 主要依赖 TorchScript 进行计算图构建。
  - 通过 **JIT（Just-In-Time 编译）** 来优化执行流程，提高推理速度。

**TorchScript**

- 一种介于 **Eager Mode（动态图）** 和 **Graph Mode（静态图）** 之间的模式。

- 提供了一种将 PyTorch 模型转换为 **静态计算图** 的方法，适用于优化和跨平台部署。

- 支持两种方式转换：

  - **Tracing（跟踪）**
    - 通过示例输入记录模型的计算路径，并构造计算图。
    
    - 适用于大部分 forward 计算路径固定的模型，但如果 if for 语句依赖输入数据，则可能导致错误。
    
  - **Scripting（脚本化）**
    - 直接使用 @torch.jit.script 将代码转换为 TorchScript，可处理 Python 逻辑分支。
    - 需要对 Python 代码进行约束（如避免 NumPy、列表推导式等， 参数类型），支持完整的 PyTorch 语法。
  

 **PyTorch JIT**

- PyTorch 的 JIT（即时编译）技术，是 TorchScript 的核心实现，主要用于**加速模型推理和部署**。

- **JIT 作用：**

  - 解析 PyTorch 代码并转换为 TorchScript（静态计算图）。
  - 通过优化计算图（如常量折叠、算子融合）提高执行效率。
  - 允许模型序列化后在 C++ 端运行（torch.jit.load 加载）。

- JIT 包含：
  - torch.jit.trace（追踪模式）
  - torch.jit.script（脚本模式）
  - torch.jit.save/load（保存和加载 TorchScript）

**torch.compile** 

PyTorch 2.0 的 torch.compile 采用了新的 **TorchDynamo + AOTAutograd + Inductor** 组合优化方式，能够自动将 Eager Mode 代码转换为高性能计算图，并执行深度优化。

- **TorchDynamo**：这是一个动态捕获工具，能够在不改变原有 Python 代码的情况下，拦截和捕获模型的前向计算图。它通过解析 Python 字节码，识别出模型中的算子和操作，生成对应的静态计算图。
- **AOTAutograd**：
  - 可以在计算前捕获完整的前向和反向传播计算图，并基于该计算图进行优化。其优化方式包括：
  - **算子调度**：根据计算图中的依赖关系，重新安排算子的执行顺序，以提高计算效率。
  - **算子融合**：将多个小算子合并为更高效的大算子，减少内存访问和计算开销。
  - **层融合**：在可能的情况下，将多个计算层合并，以优化整个计算过程。
- **TorchInductor**：这是一个深度学习编译器，**负责将前向和后向计算图降低（lower）为高效的 GPU （Triton） 或 CPU (C++ / OpenMP)代码**。通过融合算子、优化内存访问等方式，生成高性能的计算内核，从而提升模型的执行效率。

  这三个组件的协同工作流程如下：

- **捕获前向图**：当调用被 torch.compile 装饰的模型时，TorchDynamo 拦截 Python 解释器的执行，捕获前向计算图。 
- **生成后向图**：AOTAutograd 接收前向计算图，禁用钩子并调用自动微分引擎，生成对应的后向计算图，并重写前向和后向的实现。 
- **编译优化**：TorchInductor 将前向和后向计算图降低为高效的目标设备代码，进行算子融合和内存优化，生成优化后的计算内核。 
- **执行优化后的模型**：最终，优化后的计算内核被执行，实现对原始模型的加速。

**PrimTorch**

- **Prim 算子库**：约 250 个较低级别的操作符，适合编译器开发者使用。这些操作符的低级特性意味着需要通过融合等方式来提升性能。 

- **ATen 算子库**：约 750 个规范化的操作符，适合直接导出，适用于已经在 ATen 层集成的后端，或那些无法通过编译器优化低级操作符性能的后端。  (Triton 算子库 flaggems 平替)





# Triton在PyTorch



```python
torhch.compile(model: Optional[Callable] = None, *,
            fullgraph: builtins.bool = False,
            dynamic: Optional[builtins.bool] = None,
            backend: Union[str, Callable] = "inductor",
            mode: Union[str, None] = None,
            options: Optional[Dict[str, Union[str, builtins.int, builtins.bool]]] = None,
            disable: builtins.bool = False) -> Callable:
  
  
# TORCH_COMPILE_DEBUG = 1 python xx.py

# 缓存清理
rm -rf /tmp/torchinductor_xx
```

- **mode (str)**：编译模式，可选值：

  - "default"（默认）：性能和编译开销之间取得平衡。
  - "reduce-overhead"：减少 Python 运行时开销，适用于小 batch 计算，但可能会增加内存占用（通过 CUDA graphs 预分配计算所需的内存）。
  - "max-autotune"：启用 Triton 优化的矩阵乘法（matmul）和卷积（convolution），默认启用 CUDA graphs。
  - "max-autotune-no-cudagraphs"：类似 "max-autotune"，但不使用 CUDA graphs。
  - 可用 torch._inductor.list_mode_options() 查看各模式的具体配置。

  

- **options (dict)**：用于配置后端的参数，常见选项包括：

  

  - "epilogue_fusion"：将逐点（pointwise）操作融合到模板中（需启用 "max_autotune"）。
  - "max_autotune"：自动分析最优的矩阵乘法配置。
  - "fallback_random"：用于调试精度问题。
  - "shape_padding"：对矩阵形状进行填充，以优化 GPU 访问（尤其是 Tensor Cores）。
  - "triton.cudagraphs"：减少 Python 运行时开销，使用 CUDA graphs。
  - "trace.enabled"：开启调试模式，记录编译过程。
  - "trace.graph_diagram"：生成优化后计算图的可视化图片。
  - 可使用 torch._inductor.list_options() 查看 Inductor 相关的所有选项。

  

- **disable (bool)**：如果设置为 True，则 torch.compile() 不执行任何优化，仅用于测试。







# triton算子开发

![image-20250404125503246](https://cdn.jsdelivr.net/gh/631068264/img/202504041310355.png)

[Triton算子关键参数优化](https://www.bilibili.com/video/BV1miotY3E2D?spm_id_from=333.788.videopod.sections&vd_source=d591262dc9ce1bba22682d1cba1ca930)

[Triton-Puzzles](https://github.com/srush/Triton-Puzzles)

[Triton-Puzzles-Lite](https://github.com/SiriusNEO/Triton-Puzzles-Lite)

[Triton算子开发](https://www.bilibili.com/video/BV193fFYkE7P?spm_id_from=333.788.videopod.sections&vd_source=d591262dc9ce1bba22682d1cba1ca930)



## 向量加法

```python
@triton.jit
def add_kernel(x_ptr,  # *Pointer* to first input vector.
               y_ptr,  # *Pointer* to second input vector.
               output_ptr,  # *Pointer* to output vector.
               n_elements,  # Size of the vector.
               BLOCK_SIZE: tl.constexpr,  # Number of elements each program should process.
               # NOTE: `constexpr` so it can be used as a shape value.
               ):
    # There are multiple 'programs' processing different data. We identify which program
    # we are here:
    pid = tl.program_id(axis=0)  # We use a 1D launch grid so axis is 0.
    # This program will process inputs that are offset from the initial data.
    # For instance, if you had a vector of length 256 and block_size of 64, the programs
    # would each access the elements [0:64, 64:128, 128:192, 192:256].
    # Note that offsets is a list of pointers:
    block_start = pid * BLOCK_SIZE
    offsets = block_start + tl.arange(0, BLOCK_SIZE)
    # Create a mask to guard memory operations against out-of-bounds accesses.
    mask = offsets < n_elements
    # Load x and y from DRAM, masking out any extra elements in case the input is not a
    # multiple of the block size.
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)
    output = x + y
    # Write x + y back to DRAM.
    tl.store(output_ptr + offsets, output, mask=mask)
    
def add(x: torch.Tensor, y: torch.Tensor):
    # We need to preallocate the output.
    output = torch.empty_like(x)
    assert x.device == DEVICE and y.device == DEVICE and output.device == DEVICE
    n_elements = output.numel()
    # The SPMD launch grid denotes the number of kernel instances that run in parallel.
    # It is analogous to CUDA launch grids. It can be either Tuple[int], or Callable(metaparameters) -> Tuple[int].
    # In this case, we use a 1D grid where the size is the number of blocks:
    grid = lambda meta: (triton.cdiv(n_elements, meta['BLOCK_SIZE']), )
    # NOTE:
    #  - Each torch.tensor object is implicitly converted into a pointer to its first element.
    #  - `triton.jit`'ed functions can be indexed with a launch grid to obtain a callable GPU kernel.
    #  - Don't forget to pass meta-parameters as keywords arguments.
    add_kernel[grid](x, y, output, n_elements, BLOCK_SIZE=1024)
    # We return a handle to z but, since `torch.cuda.synchronize()` hasn't been called, the kernel is still
    # running asynchronously at this point.
    return output

torch.manual_seed(0)
size = 98432
x = torch.rand(size, device=DEVICE)
y = torch.rand(size, device=DEVICE)
output_torch = x + y
output_triton = add(x, y)
```



1. **tl.program_id(axis=0)** 是 **当前程序的 ID**，类似 CUDA blockIdx.x，用于 **并行计算时划分数据范围**。
2. **grid 定义有多少个 program 并行执行**，计算方式类似 CUDA gridDim.x = ceil(n_elements / BLOCK_SIZE)。
3. **当数据量小**，可以不使用 program_id，但当数据量大时，需要 program_id 进行划分并行计算。

| **CUDA 术语** | **Triton 术语**          |
| ------------- | ------------------------ |
| blockIdx.x    | tl.program_id(axis=0)    |
| gridDim.x     | grid = (num_blocks,)     |
| blockDim.x    | BLOCK_SIZE               |
| threadIdx.x   | tl.arange(0, BLOCK_SIZE) |

在 **CUDA** 里，**block** 里包含 **多个线程**，而 **Triton** 里没有“线程”的概念，直接用 **block** 处理一段数据，**多个 programs 并行执行**

**对比cuda kernel函数， 每个线程只计算一个元素。**

```c
__global__ void add_kernel(float* x, float* y, float* output, int n) {
    int tid = threadIdx.x + blockIdx.x * blockDim.x;
    if (tid < n) {
        output[tid] = x[tid] + y[tid];
    }
}
```



## 共享内存

```python
@triton.jit
def softmax_kernel(output_ptr, input_ptr, input_row_stride, output_row_stride, 
                   n_rows, n_cols, BLOCK_SIZE: tl.constexpr, num_stages: tl.constexpr):
    row_start = tl.program_id(0)        # 计算当前 row 索引
    row_step = tl.num_programs(0)       # 总 program 数，每个处理一部分行

    for row_idx in tl.range(row_start, n_rows, row_step, num_stages=num_stages):
        row_start_ptr = input_ptr + row_idx * input_row_stride  # 计算行起始地址
        col_offsets = tl.arange(0, BLOCK_SIZE)                 # 计算列索引
        input_ptrs = row_start_ptr + col_offsets               # 计算当前 block 需要加载的地址

        mask = col_offsets < n_cols  # 避免越界（因为 BLOCK_SIZE 可能比 n_cols 大）
        row = tl.load(input_ptrs, mask=mask, other=-float('inf'))  # 读取数据，越界填充 -inf

        row_minus_max = row - tl.max(row, axis=0)   # 计算 max，并归一化
        numerator = tl.exp(row_minus_max)          # 计算 e^x
        denominator = tl.sum(numerator, axis=0)    # 计算 sum(e^x)
        softmax_output = numerator / denominator   # 计算 softmax
        
        output_row_start_ptr = output_ptr + row_idx * output_row_stride  # 计算输出地址
        output_ptrs = output_row_start_ptr + col_offsets
        tl.store(output_ptrs, softmax_output, mask=mask)  # 存回结果
        
def softmax(x):
    n_rows, n_cols = x.shape

    # The block size of each loop iteration is the smallest power of two greater than the number of columns in `x`
    # BLOCK_SIZE 必须是 2 的幂，因为 Triton 只能处理 power-of-2 block。
    # 比如 n_cols=781，那么 BLOCK_SIZE=1024。
    BLOCK_SIZE = triton.next_power_of_2(n_cols)

    # Another trick we can use is to ask the compiler to use more threads per row by
    # increasing the number of warps (`num_warps`) over which each row is distributed.
    # You will see in the next tutorial how to auto-tune this value in a more natural
    # way so you don't have to come up with manual heuristics yourself.
    num_warps = 8 # 一个 program 内部用 8 个 warp 处理 row, 并行线程数.

    # Number of software pipelining stages. 
    # 计算 num_stages（软件流水线）
    num_stages = 4 if SIZE_SMEM > 200000 else 2

    # Allocate output
    y = torch.empty_like(x)

    # pre-compile kernel to get register usage and compute thread occupancy.
    kernel = softmax_kernel.warmup(y, x, x.stride(0), y.stride(0), n_rows, n_cols, BLOCK_SIZE=BLOCK_SIZE,
                                   num_stages=num_stages, num_warps=num_warps, grid=(1, ))
    kernel._init_handles()
    n_regs = kernel.n_regs
    size_smem = kernel.metadata.shared
    if is_hip():
        # NUM_REGS represents the number of regular purpose registers. On CDNA architectures this is half of all registers available.
        # However, this is not always the case. In most cases all registers can be used as regular purpose registers.
        # ISA SECTION (3.6.4 for CDNA3)
        # VGPRs are allocated out of two pools: regular VGPRs and accumulation VGPRs. Accumulation VGPRs are used
        # with matrix VALU instructions, and can also be loaded directly from memory. A wave may have up to 512 total
        # VGPRs, 256 of each type. When a wave has fewer than 512 total VGPRs, the number of each type is flexible - it is
        # not required to be equal numbers of both types.
        if is_cdna():
            NUM_GPRS = NUM_REGS * 2

        # MAX_NUM_THREADS represents maximum number of resident threads per multi-processor.
        # When we divide this number with WARP_SIZE we get maximum number of waves that can
        # execute on a CU (multi-processor)  in parallel.
        MAX_NUM_THREADS = properties["max_threads_per_sm"]
        max_num_waves = MAX_NUM_THREADS // WARP_SIZE
        occupancy = min(NUM_GPRS // WARP_SIZE // n_regs, max_num_waves) // num_warps
    else:
        occupancy = NUM_REGS // (n_regs * WARP_SIZE * num_warps)
    occupancy = min(occupancy, SIZE_SMEM // size_smem)
    
    num_programs = NUM_SM * occupancy # 根据 SM 数量 & 线程占用计算并行度 (根据寄存器 & 共享内存计算每个 SM 能跑多少 program)
    # 并行执行 num_programs 个 program，每个处理一行
    num_programs = min(num_programs, n_rows) # 限制不超过行数

    # Create a number of persistent programs.
    kernel[(num_programs, 1, 1)](y, x, x.stride(0), y.stride(0), n_rows, n_cols, BLOCK_SIZE, num_stages)
    return y
```





```c
// CUDA kernel: normalize each row of the input by its max value
__global__ void row_normalize_kernel(float* input, float* output, int rows, int cols) {
    extern __shared__ float sdata[];  // shared memory (动态分配)

    int row = blockIdx.x;             // 每个 block 处理一行
    int tid = threadIdx.x;            // 每个线程处理一列
    int index = row * cols + tid;

    // Step 1: load data into shared memory
    if (tid < cols) {
        sdata[tid] = input[index];
    }
    __syncthreads();

    // Step 2: compute max in shared memory (简单并行归约可扩展，但这里只用线程 0 做)
    float row_max = -FLT_MAX;
    if (tid == 0) {
        for (int i = 0; i < cols; ++i) {
            row_max = fmaxf(row_max, sdata[i]);
        }
        sdata[cols] = row_max;  // 把 max 存在 shared memory 尾部
    }
    __syncthreads();

    // Step 3: normalize each element
    if (tid < cols) {
        output[index] = sdata[tid] / sdata[cols];
    }
}


int rows = 512, cols = 128;
float *d_input, *d_output;
size_t size = rows * cols * sizeof(float);

cudaMalloc(&d_input, size);
cudaMalloc(&d_output, size);

// 填充 d_input 省略...

dim3 blockDim(cols);  // 每行一个 block，每列一个线程
dim3 gridDim(rows);   // 每个 block 处理一行
size_t sharedMemSize = (cols + 1) * sizeof(float);  // +1 为存 max

row_normalize_kernel<<<gridDim, blockDim, sharedMemSize>>>(d_input, d_output, rows, cols);
```

