内核编程最初作为图形处理器（GPU）编程的方式而广受欢迎。随着内核编程的通用化，理解内核式编程如何影响代码在中央处理器（CPU）上的映射至关重要。  

中央处理器（CPU）历经多年发展。2005年前后，随着提升时钟频率带来的性能增益逐渐减弱，行业发生了重大转变。并行计算成为优选解决方案——CPU制造商转而采用多核芯片设计，而非继续提高时钟频率。这使得计算机在执行多任务时的效能显著提升！  

虽然多核架构曾是提升硬件性能的主流路径，但要在软件层面实现这种性能增益却需要付出巨大努力。多核处理器迫使开发者设计全新算法，才能使硬件改进的效果得以显现，而这一过程往往充满挑战。<font style="color:#DF2A3F;">随着核心数量增加，如何高效利用这些计算单元变得愈发困难</font>。SYCL正是应对这些挑战的编程语言之一，它通过丰富的编程结构帮助开发者在CPU（及其他架构）上高效利用多种形式的并行计算能力。  

本章讨论 CPU 架构的一些细节、CPU 硬件通常如何执行 SYCL 应用程序，并提供<font style="background-color:#E8F7CF;">为 CPU 平台编写 SYCL 代码时的最佳实践</font>。  

## 性能注意事项（Performance Caveats）  
SYCL为实现应用程序的并行化或从头开发并行应用程序提供了一条可移植的途径。<font style="color:#DF2A3F;background-color:#FBF5CB;">当在CPU上运行时，应用程序的性能主要取决于以下因素</font>：  

+ 内核代码启动与执行的底层性能表现 
+ 程序中并行内核运行的占比及其可扩展性 
+ CPU利用率、有效数据共享、数据局部性及负载均衡 
+ 工作项之间的同步与通信量 
+ 创建工作项执行线程时的开销（包括恢复、管理、挂起、销毁及同步），该开销受串行-并行或并行-串行转换次数影响 
+ 共享内存引发的冲突（含伪共享内存问题） 
+ 共享资源的性能限制（如内存、写合并缓冲区及内存带宽）  

此外，与任何处理器类型一样，不同厂商甚至不同代际的CPU产品之间可能存在差异。<font style="background-color:#D9DFFC;">适用于某款CPU的最佳实践未必适用于其他型号或配置的CPU</font>。  

>  <font style="color:#DF2A3F;background-color:#FBF5CB;">要在CPU上实现最佳性能，务必尽可能深入了解CPU架构的诸多特性</font>！  
>

##  多核CPU基础（The Basics of Multicore CPUs）  
多核CPU的涌现与快速发展极大地推动了共享内存并行计算平台的广泛普及。CPU在笔记本电脑、台式机和服务器层级均提供并行计算平台，使其无处不在，几乎在所有场景下都能展现性能优势。当前<font style="color:#DF2A3F;background-color:#FBF5CB;">最常见的CPU架构形式</font>是<font style="background-color:#E6DCF9;">缓存一致的非统一内存访问（cc-NUMA）</font>，其<font style="color:#DF2A3F;background-color:#FBF5CB;">特点</font>是<font style="background-color:#E8F7CF;">内存访问时间并非完全一致</font>。许多小型双插槽通用CPU系统都采用此类内存架构。随着处理器内核数量与插槽数量的持续增长，这种架构已成为主流范式。  

<font style="background-color:#E8F7CF;">在cc-NUMA架构的CPU系统中，每个处理器插槽仅连接系统内存的一部分。通过缓存一致性互连技术将所有插槽整合，为程序员提供统一的系统内存视图</font>。<font style="color:#DF2A3F;">这种内存架构具备可扩展性</font>，因为聚合内存带宽会随系统插槽数量线性增长。互连技术的优势在于应用程序能够透明访问系统中所有内存，<font style="color:#DF2A3F;">不受数据物理位置限制</font>。然而这种<font style="color:#DF2A3F;background-color:#FBF5CB;">设计存在性能代价</font>：<font style="background-color:#E8F7CF;">内存访问延迟不再具有一致性（即固定访问延迟不复存在），实际延迟取决于数据在系统中的存储位置</font>。<font style="background-color:#CEF5F7;">最佳情况下，数据来自运行代码所在插槽直连的内存</font>；<font style="background-color:#CEF5F7;">最差情况下，数据必须从系统中远端插槽连接的内存获取，由于cc-NUMA CPU系统中插槽间互连的跳数增加，此类内存访问的开销会显著上升</font>。  

图16-1展示了一个采用cc-NUMA内存架构的通用CPU设计。该简化系统架构包含了当代通用多插槽系统中的核心与内存组件。本章后续内容将借助该示意图，演示对应代码示例的映射关系。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">为实现最佳性能，我们必须充分理解特定系统的cc-NUMA架构特性</font>。以英特尔最新服务器为例，其采用网状互连架构——处理器核心、缓存及内存控制器以行列矩阵形式排布。在追求系统峰值性能时，掌握处理器与内存间的互联关系至关重要。  

![图16-1. 通用多核CPU系统](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744965588711-0cd8ed9e-805d-49a3-a3bf-1c24034f116b.png)

图16-1所示系统包含两个插槽，每个插槽配备两个内核，每个内核具有四个硬件线程。每个内核拥有专用的一级（L1）缓存。所有L1缓存均连接到共享的最后一级缓存，该缓存与插槽上的内存系统相连。同一插槽内的内存访问延迟具有均匀性，即延迟值保持一致且可精准预测。  

两个处理器插槽通过缓存一致性互连架构相连。内存分布在系统各处，但所有内存均可从系统任意位置透明访问。当访问非当前运行代码所在插槽的内存时，读写延迟呈现非均匀特性——这意味着访问远端插槽数据可能产生显著更长且不稳定的延迟。然而该互连架构的核心在于一致性保障：我们无需担忧系统中内存数据视图的不一致问题，只需专注于分布式内存访问方式对性能的影响。更高级的优化技术（如采用宽松内存序的原子操作）可实现对硬件内存一致性要求较低的操作，但当我们需要一致性时，硬件会确保其完美实现。  

<font style="color:#DF2A3F;">CPU中的硬件线程是执行载体，即执行指令流的运算单元</font>。图16-1中的硬件线程采用0至15的连续编号标注，此标记法用于简化本章示例的讨论。<font style="background-color:#E8F7CF;">除非特别说明，本章中所有关于CPU系统的描述均指向图16-1所示的参考cc-NUMA系统</font>。  

##  SIMD硬件基础（The Basics of SIMD Hardware）
1996年，x86架构上广泛部署的MMX扩展指令集成为早期大规模应用的SIMD技术范例。此后英特尔体系架构及整个行业相继推出了众多SIMD指令集扩展方案。处理器核心通过执行指令完成运算任务，其所能处理的指令类型由底层指令集（如x86、x86_64、AltiVec、NEON）及所实现的扩展指令集（如SSE、AVX、AVX-512）共同决定，其中指令集扩展新增的运算功能大多专注于SIMD并行处理。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">单指令多数据（SIMD）指令</font><font style="background-color:#E8F7CF;">通过使用比被处理数据基本单元更大的寄存器和硬件，使得单个核心上能同时执行多个计算</font>。例如，利用512位寄存器，我们可通过单条机器指令完成八次64位运算。  

图16-2所示的示例理论上可实现高达八倍的加速效果。但实际应用中，这种加速往往会有所折损——当八倍加速解决了一个性能瓶颈后，往往会暴露出新的瓶颈（如内存吞吐量限制）。一般而言，SIMD技术的性能优势因具体场景而异，<font style="background-color:#E8F7CF;">在某些情况下（如存在大量分支发散、非连续内存访问的收集/散射操作、SIMD载入存储发生缓存行分割时），其表现甚至可能逊于简单的非SIMD等效代码</font>。尽管如此，若能准确掌握SIMD技术的适用场景与应用方法（或交由编译器自动实现），当代处理器仍能获得显著性能提升。与所有性能优化手段相同，开发人员应在投入生产环境前，于典型目标机器上实测加速效果。本章后续章节将更详细探讨预期性能增益的相关细节。  

```cpp
h.parallel_for(range(1024), [=](id<1> k) { z[k] = x[k] + y[k]; });
```

![图16-2. CPU硬件线程中的SIMD执行](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744965706553-a4cf639a-6d39-4c4e-8137-139551fd7746.png)

---

> **<font style="color:#DF2A3F;">插图解释</font>**
>
> 1. **传统的标量执行**：如果你使用传统的 CPU 执行方式（标量方式），每次只能处理一个数据：
>     - 加载 `x[0]` 和 `y[0]`，执行加法，存储结果 `z[0]`；
>     - 然后加载 `x[1]` 和 `y[1]`，执行加法，存储结果 `z[1]`；
>     - 依此类推，直到完成所有的加法操作。
>
> 这种方式称为 **标量执行**，它每次处理一个数据。
>
> 1. **SIMD 执行**：在 SIMD 中，CPU 会同时处理多个数据，利用硬件上的并行处理能力：
>     - 假设 CPU 能在一次操作中处理 8 个数据（宽度为 512 位，意味着一次能加载 8 个 64 位数据），那么它会将 `x[]` 和 `y[]` 数组分成 8 个数字一组。
>     - 例如，它会将 `x[]` 和 `y[]` 数组的前 8 个元素（即 `x[0..7]` 和 `y[0..7]`）同时加载到处理器的 SIMD 寄存器中，并同时执行加法运算。
>
> 这种方式大大加快了计算速度，因为 CPU 可以在一个时钟周期内同时执行多个数据的操作。
>

---

采用SIMD单元的cc-NUMA CPU架构构成了多核处理器的基础，该架构能够以至少五种不同方式（如图16-3所示）开发从指令级并行开始的广泛并行性。  

![图16-3. 并行执行指令的五种方式](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744965747649-ceff177c-721f-40db-9523-9f814926763a.png)

在图16-3中，<font style="color:#DF2A3F;background-color:#FBF5CB;">指令级并行性</font>可<font style="background-color:#E8F7CF;">通过标量指令的乱序执行和单线程内的SIMD并行机制实现</font>。<font style="color:#DF2A3F;background-color:#FBF5CB;">线程级并行性</font>则可<font style="background-color:#E8F7CF;">通过在同一核心或多核心上执行多个线程来实现，其规模可有所不同</font>。更具体而言，线程级并行性可从以下方面体现：  

1. 现代CPU架构允许单个核心同时执行两个或多个线程的指令。 
2. 多核架构在每个处理器内包含两个或更多核心。操作系统将其每个执行核心视为独立的处理器，具备所有相关的执行资源。 
3. 处理器（芯片）级别的多任务处理可通过执行独立的代码线程实现。因此，处理器可以同时运行一个应用程序的线程和另一个操作系统的线程，也可以运行单个应用程序内的并行线程。 
4. 分布式处理可通过在计算机集群上执行由多线程组成的进程来实现，这些集群通常通过消息传递框架进行通信。  

随着多处理器计算机和多核技术日益普及，采用并行处理技术作为标准实践以提升性能变得至关重要。本章后续章节将介绍SYCL中的编码方法与性能调优技术，这些技术能帮助我们在多核CPU上实现峰值性能。  

与其他并行处理硬件（如GPU）类似，关键是要为处理器提供足够大的数据集进行处理。为了说明利用多级并行性处理大规模数据的重要性，我们以图16-4所示的简单C++ STREAM Triad程序为例进行说明。  

```cpp
// C++ STREAM Triad 基准测试  
// __restrict 关键字用于声明参数间无内存重叠  
template <typename T>  
double triad(T* __restrict VA, T* __restrict VB, T* __restrict VC, size_t array_size, const T scalar) {  
    double ts = timer_start();  
    for (size_t id = 0; id < array_size; id++) {  
        VC[id] = VA[id] + scalar * VB[id];  
    }  
    double te = timer_end();  
    return (te – ts);  
}
```

>  关于STREAM TRIAD工作负载的说明  
>
> 流三件套工作负载（www.cs.virginia.edu/stream）是CPU厂商用于展示内存带宽能力的重要且广受认可的基准测试负载。我们采用流三件套核心程序来演示并行内核的代码生成及其调度方式——通过本章所述技术，这种调度能显著提升性能。该工作负载虽相对简单，却足以清晰展示诸多优化方法。布里斯托大学开发的Babelstream项目提供了包含SYCL版本C++实现的流测试方案。  
>

STREAM Triad循环可以轻松在CPU上使用单核进行串行执行。优秀的C++编译器会通过循环向量化技术，为具备指令级SIMD并行硬件支持的CPU生成SIMD代码。例如，对于支持AVX-512指令集的英特尔至强处理器，英特尔C++编译器生成的SIMD代码如图16-5所示。关键在于，编译器通过对代码的转化（采用SIMD指令和循环展开技术），减少了循环迭代次数——每次迭代能处理更多数据量。  

```cpp
# %bb.0:
vbroadcastsd      %xmm0, %zmm0            # %entry
movq            $-32, %rax
.p2align         4, 0x30
.LBB0_1:
# %loop.19
# =>This Loop Header: Depth=1
# load 8 elements from memory to zmm1
vmovupd 256(%rdx,%rax,8), %zmm1       # zmm1 = (zmm0*zmm1)+mem
vfmadd213pd      256(%rsi,%rax,8), %zmm0, %zmm1 # zmm1 = (zmm0*zmm1)+mem
vmovupd %zmm1, 256(%rdi,%rax,8)       # store 8-element result to mem from zmm1
vmovupd 320(%rdx,%rax,8), %zmm1       # zmm1 = (zmm0*zmm1)+mem
vfmadd213pd      320(%rsi,%rax,8), %zmm0, %zmm1 # zmm1 = (zmm0*zmm1)+mem
vmovupd %zmm1, 320(%rdi,%rax,8)       # store 8-element result to mem from zmm1
vmovupd 384(%rdx,%rax,8), %zmm1       # zmm1 = (zmm0*zmm1)+mem
vfmadd213pd      384(%rsi,%rax,8), %zmm0, %zmm1 # zmm1 = (zmm0*zmm1)+mem
vmovupd %zmm1, 384(%rdi,%rax,8)       # store 8-element result to mem from zmm1
vmovupd 448(%rdx,%rax,8), %zmm1       # zmm1 = (zmm0*zmm1)+mem
vfmadd213pd      448(%rsi,%rax,8), %zmm0, %zmm1 # zmm1 = (zmm0*zmm1)+mem
vmovupd %zmm1, 448(%rdi,%rax,8)       # store 8-element result to mem from zmm1
addq            $32, %rax
cmpq            $134217696, %rax       # imm = 0x7FFFFFF0
jb              .LBB0_1
```

如图16-5所示，<font style="color:#DF2A3F;background-color:#FBF5CB;">编译器可通过两种方式利用指令级并行性</font>：其一，<font style="background-color:#E8F7CF;">采用SIMD指令开发指令级数据并行性，单条指令即可同时处理八个双精度数据元素（每指令）</font>；其二，<font style="background-color:#E8F7CF;">编译器基于硬件多路指令调度机制实施循环展开，使得这些互无依赖关系的指令能够获得乱序执行效果</font>。  

若我们尝试在CPU上执行此函数，对于较小的数组规模它或许尚能运行——但表现并不出色，因其未能利用CPU的任何多核或线程能力。然而，若试图在CPU上处理大规模数组，其性能很可能急剧恶化：单线程仅能调用单一CPU核心，当内存带宽达到该核心的饱和极限时，便会形成性能瓶颈。  

##  挖掘线程级并行性（Exploiting Thread-Level Parallelism）
为提高STREAM Triad内核的性能，我们可通过将循环转换为parallel_for内核，对可并行处理的数据元素范围进行计算。  

该STREAM Triad SYCL并行内核的主体结构与在CPU上以串行C++方式执行的STREAM Triad循环主体完全一致，如图16-6所示。  

```cpp
constexpr int num_runs = 10;  // 测试运行次数
constexpr size_t scalar = 3;   // 标量乘数

double triad(const std::vector<float>& vecA, 
            const std::vector<float>& vecB,
            std::vector<float>& vecC) 
{
    assert(vecA.size() == vecB.size() && vecB.size() == vecC.size());
    const size_t array_size = vecA.size();
    double min_time_ns = std::numeric_limits<double>::max();

    queue q{property::queue::enable_profiling{}};
    std::cout << "Running on device: " 
             << q.get_device().get_info<info::device::name>() << "\n";

    buffer<float> bufA(vecA);
    buffer<float> bufB(vecB);
    buffer<float> bufC(vecC);

    for (int i = 0; i < num_runs; i++) {
        auto Q_event = q.submit([&](handler& h) {
            accessor A{bufA, h};
            accessor B{bufB, h};
            accessor C{bufC, h};

            h.parallel_for(array_size, [=](id<1> idx) {
                C[idx] = A[idx] + B[idx] * scalar;
            });
        });
        double exec_time_ns = 
            Q_event.get_profiling_info<info::event_profiling::command_end>() -
            Q_event.get_profiling_info<info::event_profiling::command_start>();

        std::cout << "Execution time (iteration " << i << ") [sec]: " 
                 << (double)exec_time_ns * 1.0E-9 << "\n";
        min_time_ns = std::min(min_time_ns, exec_time_ns);
    }

    return min_time_ns;
}
```

尽管该并行内核与采用循环结构编写的串行C++版STREAM Triad函数极为相似，但由于`<font style="color:#DF2A3F;">parallel_for</font>`<font style="color:#DF2A3F;">使得数组的不同元素能够在多核上并行处理</font>，其运行速度显著提升。图16-7展示了该内核在CPU上的映射原理：假设系统配置为单插槽、四核心、每核双硬件线程（共八线程），且实现方案以每组32个工作项的工作组处理数据。当需要处理1024个双精度数据元素时，将生成32个工作组。<font style="color:#601BDE;">工作组调度可采用轮询机制</font>，即线程ID=工作组ID模8。实质上每个线程将执行四个工作组，每轮可并行执行八个工作组。需注意的是，此场景下的工作组是由SYCL编译器和运行时隐式构建的工作项集合。  

![图16-7. STREAM Triad并行内核的映射关系](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744966390393-9d31a817-120b-4d84-8a0f-2bb7e7642fb1.png)

需注意的是，<font style="color:#DF2A3F;">在SYCL程序中，数据元素的具体划分方式及其分配到不同处理器核心（或线程）的过程并未明确规定</font>。这使得<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL实现能够灵活选择在特定CPU上最优执行并行内核的方式</font>。尽管如此，实现方可能会为程序员提供一定程度的控制权，以支持性能调优（例如通过编译器选项或环境变量实现）。  

虽然CPU可能带来相对昂贵的线程上下文切换和同步开销，但<font style="color:#DF2A3F;">在处理器核心上驻留更多软件线程可能是有益的</font>，因为这使得每个处理器核心拥有可执行工作的选择权。当某个软件线程正在等待另一线程生成数据时，处理器核心可以切换到另一个准备就绪的软件线程，从而避免核心闲置。  

> 选择如何绑定和调度线程  
>
> 选择有效的方案来分区和调度线程之间的工作对于调整 CPU 和其他设备类型上的应用程序非常重要。后续部分将描述一些技术。  
>

###  线程亲和性洞察（Thread Affinity Insight）
<font style="color:#DF2A3F;background-color:#FBF5CB;">线程亲和性指定了特定线程执行的CPU核心</font>。<font style="background-color:#E8F7CF;">若线程在核心间频繁迁移（例如线程未在同一核心上执行），性能可能受损——当数据在不同核心间来回传递时，缓存局部性会降低效率</font>。  

DPC++编译器运行时库通过环境变量DPCPP_CPU_CU_AFFINITY、DPCPP_CPU_PLACES、DPCPP_CPU_NUM_CUS和DPCPP_CPU_SCHEDULE支持多种线程与核心绑定的方案，这些变量并非由SYCL标准定义。其他实现方案可能提供类似的环境变量配置。  

首个可调参数是环境变量DPCPP_CPU_CU_AFFINITY。通过该环境变量进行调优操作简便、成本低廉，但对众多应用程序能产生显著影响。该环境变量的具体说明如图16-8所示。  

![图16-8. DPCPP_CPU_CU_AFFINITY环境变量](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744966765584-eb911f61-eafe-4594-9ca2-6d576a042196.png)

当指定环境变量 `DPCPP_CPU_CU_AFFINITY` 时，软件线程将通过以下公式绑定到硬件线程：  

展开式绑定逻辑：boundHT = (tid mod numHT) + (tid mod numSocket) × numHT   
闭合式绑定逻辑：boundHT = tid mod (numSocket × numHT)  

 其中 

+ tid 表示软件线程标识符 
+ boundHT 表示线程 tid 所绑定的硬件线程（逻辑核心） 
+ numHT 表示每个插槽的硬件线程数量 
+ numSocket 表示系统中的插槽数量  

假设我们在一个双核双插槽系统上运行一个包含八个线程的程序——换言之，系统配备四个物理核心，共需编程管理八个线程。图16-9展示了在不同DPCPP_CPU_CU_AFFINITY设置下，线程如何映射到硬件线程与物理核心的示例。  

![图16-9. 利用硬件线程将线程映射至核心](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744966834418-21200e35-11bb-412d-bddc-92d0cf8abc35.png)

结合环境变量`DPCPP_CPU_CU_AFFINITY`，还有以下支持CPU性能调优的环境变量：  

+ DPCPP_CPU_NUM_CUS = [n]，用于设置内核执行所使用的线程数，默认值为系统中硬件线程的数量。
+ DPCPP_CPU_PLACES = [ sockets | numa_domains | cores | threads ]，用于指定亲和性设置的作用域，类似于OpenMP 5.1中的OMP_PLACES，默认设置为cores（核心）。 
+ DPCPP_CPU_SCHEDULE = [ dynamic | affinity | static ]，用于指定工作组的调度算法，默认设置dynamic（动态调度）。   
dynamic（动态调度）：启用auto_partitioner（自动分区器），通常能通过充分的任务分割实现工作线程间的负载均衡。  

affinity（亲和性）：启用affinity_partitioner，该分区器可提升缓存亲和性，并在将子范围映射至工作线程时采用比例分割策略。   
	静态：启用static_partitioner，该分区器会尽可能均匀地将迭代任务分配给工作线程。  

在使用英特尔OpenCL CPU运行时环境执行时，工作组的调度由线程构建模块（TBB）库处理。通过设置DPCPP_CPU_SCHEDULE参数可指定采用的TBB分区策略。需注意的是，<font style="color:#DF2A3F;">TBB分区器还会通过粒度参数控制任务分割，其默认粒度值为1，表示所有工作组均可独立执行</font>。更多详细信息请参阅：tinyurl.com/oneTBBpart。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">缺乏线程亲和性调优并不必然导致性能下降</font>。<font style="background-color:#E8F7CF;">实际性能往往更多取决于并行执行的线程总数，而非线程与数据的关联和绑定程度</font>。通过基准测试来验证应用程序是判断线程亲和性是否影响性能的有效方法。如图16-1所示的STREAM Triad代码，初始未设置线程亲和性时性能较低。通过控制亲和性设置并利用环境变量实现软件线程的静态调度（如下所示的Linux导出命令），性能得到了提升：  

```cpp
export DPCPP_CPU_PLACES=numa_domains  
export DPCPP_CPU_CU_AFFINITY=close
```

通过将numa_domains设为亲和性的位置参数，TBB任务竞技场被绑定至NUMA节点或CPU插槽，工作负载得以均匀分布在各个任务竞技场之间。<font style="background-color:#E8F7CF;">通常建议将环境变量DPCPP_CPU_PLACES与DPCPP_CPU_CU_AFFINITY配合使用</font>。这些环境变量设置帮助我们在配备2个插槽、每插槽28个核心、每核心2个硬件线程、主频2.5 GHz的英特尔至强服务器系统上实现约30%的性能提升。然而，我们仍可进一步优化以提升该CPU的性能表现。  

###  谨记初次接触记忆（Be Mindful of First Touch to Memory）
<font style="color:#DF2A3F;background-color:#FBF5CB;">内存存储在其首次被访问（使用）的位置</font>。在本例中，由于初始化循环由主机线程串行执行，所有内存均与主机线程运行的CPU插槽相关联。其他插槽后续访问数据时，将读取初始插槽（用于初始化的插槽）所连接的内存，这显然不利于性能表现。如图16-10所示，通过并行化初始化循环来<font style="color:#DF2A3F;">控制跨插槽的首访效应</font>，我们可在STREAM Triad内核上实现更高性能。  

```cpp
template <typename T> 
void init(queue& deviceQueue, T* VA, T* VB, T* VC, size_t array_size) {
    range<1> numOfItems{array_size};
    buffer<T, 1> bufferA(VA, numOfItems); 
    buffer<T, 1> bufferB(VB, numOfItems);
    buffer<T, 1> bufferC(VC, numOfItems);

    auto queue_event = deviceQueue.submit([&](handler& cgh) {
        auto aA = bufferA.template get_access<sycl_write>(cgh);
        auto aB = bufferB.template get_access<sycl_write>(cgh);
        auto aC = bufferC.template get_access<sycl_write>(cgh);

        cgh.parallel_for<class Init<T>>(numOfItems, [=](id<1> wi) {
            aA[wi] = 2.0;
            aB[wi] = 1.0;
            aC[wi] = 0.0;
        });
    });

    queue_event.wait();
}
```

<font style="color:#DF2A3F;">利用初始化代码中的并行性可提升内核在CPU上的运行性能</font>。此例中，我们在Intel Xeon处理器系统上实现了约2倍的性能增益。  

本章最近几节已表明，<font style="color:#DF2A3F;background-color:#FBF5CB;">通过利用线程级并行性，我们能够有效发挥CPU核心与线程的计算效能</font>。然而，若<font style="color:#DF2A3F;background-color:#FBF5CB;">要实现峰值性能，还需充分挖掘CPU核心硬件中的SIMD向量级并行计算能力</font>。  

>  SYCL并行内核能够充分利用跨核心和硬件线程的线程级并行性优势！  
>

##  CPU上的SIMD向量化（SIMD Vectorization on CPU） 
虽然一段编写良好的SYCL内核（不含工作项间依赖关系）可以在CPU上高效并行运行，但实现方案同样可对SYCL内核应用向量化技术，以利用类似于第15章所述GPU支持的SIMD硬件。本质上，<font style="background-color:#E8F7CF;">多数数据元素通常位于连续内存中，且在数据并行内核中遵循相同的控制流路径，从而使用SIMD指令</font>。例如，在包含语句a[i] = a[i] + b[i]的内核中，通过多个数据元素共享硬件逻辑并将其作为组执行，每个数据元素皆以相同的指令流（加载、加载、加法、存储）执行，这种模式可自然地映射到硬件的SIMD指令集上。具体而言，单个指令可同时处理多个数据元素。  <font style="color:#DF2A3F;">CPU处理器可通过以下事实优化内存加载、存储和运算操作</font>：多数数据元素通常位于连续内存中，且在数据并行内核中遵循相同的控制流路径，从而使用SIMD指令。例如，在包含语句a[i] = a[i] + b[i]的内核中，通过多个数据元素共享硬件逻辑并将其作为组执行，每个数据元素皆以相同的指令流（加载、加载、加法、存储）执行，这种模式可自然地映射到硬件的SIMD指令集上。具体而言，单个指令可同时处理多个数据元素。  

<font style="background-color:#E8F7CF;">一条指令同时处理的数据元素数量</font>有时被称为该指令或执行该指令的处理器的**向量长度**（或**SIMD宽度**）。在图16-11中，我们的指令流以四路SIMD执行方式运行。  

![图16-11. SIMD执行的指令流](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744986473504-4aff7256-eeff-4d08-b601-9fda152cfe39.png)

CPU处理器并非唯一实现SIMD指令集的处理器。诸如GPU等其他处理器也采用SIMD指令来提升大规模数据处理效率。与其他处理器类型相比，英特尔至强CPU处理器的关键差异在于其具有三种固定位宽的SIMD寄存器（128位XMM、256位YMM和512位ZMM），而非可变长度的SIMD位宽。<font style="background-color:#FBF5CB;">当我们使用子组或向量类型编写具有SIMD并行性的SYCL代码时（参见第11章），需特别注意硬件中的SIMD位宽及SIMD向量寄存器数量</font>。  

###  确保SIMD执行合法性（Ensure SIMD Execution Legality）
从语义上讲，<font style="background-color:#FBF5CB;">SYCL执行模型确保SIMD执行可应用于任何内核，且每个工作组中的工作项集合（即子组）可通过SIMD指令并发执行</font>。某些实现方案可能选择改用SIMD指令执行内核内的循环，但该操作仅当满足以下条件时可行：保留所有原始数据依赖性，或编译器能基于私有化与归约语义解析数据依赖性。此类实现方案通常会将子组大小报告为1。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">单个SYCL内核的执行可以通过在工作组内使用SIMD指令，从处理单一工作项转变为处理一组工作项</font>。<font style="background-color:#E8F7CF;">在ND-range模型下，编译器向量化器会选择增长最快（单位步长）的维度来生成SIMD代码</font>。<font style="background-color:#E8F7CF;">本质上，要实现给定ND-range的向量化，同一子组中任何两个工作项之间不得存在跨工作项依赖关系，否则编译器需保留同一子组内的跨工作项前向依赖关系</font>。  

当工作项（work-items）的内核执行映射到CPU线程时，细粒度同步的开销众所周知较高，且线程上下文切换的开销也很大。因此，<font style="color:#DF2A3F;background-color:#FBF5CB;">在为CPU编写SYCL内核时，消除工作组（work-group）内工作项间的依赖关系是一项重要的性能优化措施</font>。另一种有效方法是<font style="color:#DF2A3F;background-color:#FBF5CB;">将此类依赖限制在子组（sub-group）内的工作项之间</font>，如图16-12所示的写前读依赖（read-before-write dependence）。若子组在SIMD执行模型下运行，编译器可将内核中的子组屏障（sub-group barrier）视为空操作（noop），运行时不会产生实际同步开销。  

```cpp
const int n = 16, w = 16;  
queue q;  
range<2> G = {n, w};  
range<2> L = {1, w};  

int *a = malloc_shared<int>(n * (n + 1), q);  
for (int i = 0; i < n; i++)  
    for (int j = 0; j < n + 1; j++)  
        a[i * n + j] = i + j;  

q.parallel_for(  
    nd_range<2>{G, L},  
    [=](nd_item<2> it) [[sycl::reqd_sub_group_size(w)]] {  
        // 在子组内均匀分布"i"，并进行16次冗余计算  
        const int i = it.get_global_id(0);  
        sub_group sg = it.get_sub_group();  

        for (int j = sg.get_local_id()[0]; j < n; j += w) {  
            // 在更新a[i*n+j:16]前加载a[i*n+j+1:16]，以保持循环携带的前向依赖  
            auto va = a[i * n + j + 1];  
            group_barrier(sg);  
            a[i * n + j] = va + i + 2;  
        }  

        group_barrier(sg);  
    }).wait();
```

核心代码已进行向量化处理（以向量长度为8为例），其SIMD执行过程如图16-13所示。工作组设置为(1,8)的规模组，内核中的循环迭代被分配至该子组的工作项上，通过八路SIMD并行方式执行。  

![图16-13. 具有前向依赖循环的SIMD向量化](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744986657722-63a695f2-f598-4a04-b225-91ef9fa35da2.png)

在此示例中，若内核中的循环是性能瓶颈，允许在子组内实现SIMD向量化将带来显著的性能提升。

通过采用并行处理数据元素的SIMD指令，可使内核性能突破CPU核心数和硬件线程数的限制实现扩展。  

###  SIMD掩码与开销（SIMD Masking and Cost）
在实际应用中，我们可能会遇到条件语句（如if语句）、条件表达式（如a = b > a ? a : b）、迭代次数可变的循环、switch语句等结构。任何条件性结构都可能导致标量控制流执行不同的代码路径，正如GPU上的情况（参见第15章），这会降低性能。SIMD掩码是由内核中条件语句生成的一组取值为1或0的位。以数组A={1,2,3,4}和B={3,7,8,1}及比较表达式a<b为例，该比较会生成包含四个值{1,1,1,0}的掩码（可存储于硬件掩码寄存器中），用于指示后续SIMD指令中哪些通道应执行受该比较条件保护（启用）的代码。  

若内核包含条件代码，则通过掩码指令进行向量化处理。这些指令的执行取决于与每个数据元素（SIMD指令中的通道）关联的掩码位。每个数据元素的掩码位即为掩码寄存器中的对应位。  

 使用掩码可能导致性能低于对应的非掩码版本代码。这可能是由于以下原因造成的： 

+ 每次加载时额外的掩码混合操作 
+ 对目标地址的依赖性  

掩码操作会产生开销，因此仅在必要时使用。当内核为ND范围内核且在执行范围内显式分组工作项时，应谨慎选择ND范围工作组大小，通过最小化掩码开销来最大化SIMD效率。若工作组尺寸无法被处理器的SIMD宽度整除，则该工作组的局部执行过程可能需启用内核掩码机制。  

 图16-14展示了合并掩码操作如何产生对目标寄存器的依赖性：  

+ 在不使用掩码的情况下，处理器每个周期可执行两次乘法运算（vmulps）。 
+ 在使用合并掩码时，由于乘法指令（vmulps）会保留目标寄存器中的结果（如图16-17所示），处理器每四个周期才能执行两次乘法运算。 
+ 零掩码不受目标寄存器的依赖限制，因此每个周期可执行两次乘法运算（vmulps）。  

![图16-14. 内核掩码处理的三种掩码码生成方式](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744986804883-fd84b84f-f438-4762-8c2e-164cfb0e443a.png)

<font style="color:#DF2A3F;background-color:#FBF5CB;">访问缓存对齐的数据比访问非对齐数据具有更佳的性能</font>。在许多情况下，地址在编译时未知或已知但未对齐。处理循环时，可采用内存访问剥离技术——先通过掩码访问处理前几个元素直至首个对齐地址，随后利用多版本技术处理未掩码的访问及带掩码的剩余部分。该方法虽会增加代码体积，但能整体提升数据处理效率。对于并行内核编程，开发者可通过手动应用类似技术，或确保内存分配实现适当对齐来优化性能。

###  避免使用结构体数组以提升SIMD效率（Avoid Array of Struct for SIMD Efficiency）
<font style="color:#DF2A3F;background-color:#FBF5CB;">AOS（Array-of-Struct）结构会导致数据的聚集和分散操作，这不仅可能影响SIMD效率，还会为内存访问引入额外的带宽开销与延迟</font>。硬件提供的聚集-分散机制并不能消除结构重组的必要性——聚集-分散访问通常比连续加载需要显著更高的带宽和更长的延迟。假设存在结构体`struct{float x; float y; float z; float w;} a[4]`的AOS数据布局，图16-15展示了操作该数据的核函数示例。  

```cpp
cgh.parallel_for<class aos<T>>(numOfItems, [=](id<1> wi) {
    x[wi] = a[wi].x; // 导致聚集 x0, x1, x2, x3
    y[wi] = a[wi].y; // 导致聚集 y0, y1, y2, y3
    z[wi] = a[wi].z; // 导致聚集 z0, z1, z2, z3
    w[wi] = a[wi].w; // 导致聚集 w0, w1, w2, w3
});
```

当编译器沿一组工作项对内核进行向量化时，由于需要进行非连续步长的内存访问，会导致生成SIMD聚集指令。例如，访问a[0].x、a[1].x、a[2].x和a[3].x的步长为4，而非更高效的连续步长1。

![](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744986947576-bbaccd20-85ba-40cd-b4a8-77bb4cd9d8f3.png)

<font style="color:#DF2A3F;background-color:#FBF5CB;">在内核中，我们通常可以通过消除内存的聚集-分散操作来提升SIMD效率</font>。某些代码会受益于数据布局的转换——将原本采用结构体数组(AOS)表示的数据结构改为数组结构体(SOA)表示，即为每个结构体字段建立独立数组，从而在执行SIMD向量化时保持内存访问的连续性。例如，采用SOA数据布局的结构体可表示为：`struct {float x[4]; float y[4]; float z[4]; float w[4];} a;` 如下图所示：  

![](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744986967097-ea01fa05-a4a8-40a0-bbab-44bb2257b1c8.png)

核心计算单元即便在向量化状态下，仍可采用单位步长（连续）向量加载/存储操作处理数据，如图16-16所示。 

```cpp
cgh.parallel_for<class aos<T>>(numOfItems, [=](id<1> wi) {  
    x[wi] = a.x[wi]; // 生成连续步长向量加载 x[0:4]  
    y[wi] = a.y[wi]; // 生成连续步长向量加载 y[0:4]  
    z[wi] = a.z[wi]; // 生成连续步长向量加载 z[0:4]  
    w[wi] = a.w[wi]; // 生成连续步长向量加载 w[0:4]  
});
```

<font style="color:#DF2A3F;background-color:#FBF5CB;">SOA（结构体数组）数据布局有助于防止在跨数组元素访问结构体单个字段时出现数据聚集现象，同时能辅助编译器对与工作项相关联的连续数组元素实现内核向量化</font>。<font style="background-color:#E8F7CF;">需要注意的是，此类AOS（数组结构体）到SOA或AOSOA（混合布局）的数据布局转换应在程序层面（由开发者）完成，并需统筹考虑数据结构的所有使用场景</font>。<font style="background-color:#CEF5F7;">若仅在循环层面实施转换，则需要在循环前后执行代价高昂的格式转换操作</font>。当然，我们也可依赖编译器对AOS布局数据实施带一定开销的向量加载-重排优化。当SOA（或AOS）数据布局的成员具有向量类型时，编译器向量化过程会根据底层硬件特性选择水平扩展或垂直扩展策略，从而生成最优代码。  

###  数据类型对SIMD效率的影响（Data Type Impact on SIMD Efficiency）
C++程序员在确定数据适配32位有符号类型时，常习惯性地使用整型数据类型，这往往会导致如下代码模式：`int id = get_global_id(0); a[id] = b[id] + c[id];`。然而鉴于`get_global_id(0)`的返回类型为`size_t`（无符号整型，通常为64位），此类类型转换可能削弱编译器依法实施的优化能力。例如在内核代码向量化过程中，该操作可能导致编译器生成SIMD聚集/散集指令。  

+ 对 `a[get_global_id(0)]`的读取可能导致 SIMD 单元步长向量加载。 
+ 对 `a[(int)get_global_id(0)]` 的读取可能导致非单元步长的聚集指令。  

这种微妙情形源于从`size_t`到`int`（或`uint`）数据类型转换的环绕行为（C++标准中未定义行为和/或明确定义的环绕行为），这主要是基于C语言系列演进过程中遗留的历史产物。具体而言，某些转换过程中的溢出属于未定义行为，这使得编译器可以假定此类情形永不发生，从而实施更激进的优化策略。图16-17为希望了解技术细节的读者展示了若干典型案例。  

![图16-17. 整型数值回绕示例](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744987344453-d322e737-6420-4694-93de-c431ede4cdc8.png)

SIMD聚集/分散指令的执行速度低于SIMD单位步长向量加载/存储操作。<font style="color:#DF2A3F;background-color:#FBF5CB;">为实现最优的SIMD效率，无论使用何种编程语言，避免聚集/分散操作对应用程序而言都至关重要</font>。  

大多数SYCL的`get_*_id()`系列函数具有相同的特性——虽然多数情况下返回值受限于MAX_INT（例如工作组内的最大ID值），但编译器仍可合法地假设相邻工作项组的内存地址采用单位步长，从而避免聚集/分散操作。当全局ID或其派生值可能溢出导致编译器无法安全生成线性单位步长的向量内存加载/存储指令时，编译器将生成聚集/分散操作。  

在"为用户提供最佳性能"的理念指导下，DPC++编译器默认假设不会发生溢出，这种假设在实践中几乎总能成立，因此编译器能生成最优化的SIMD代码以实现出色性能。但DPC++编译器提供了-fnosycl-id-queries-fit-in-int编译选项，用以告知编译器可能存在溢出情况，且基于ID查询生成的向量化访问可能不安全。该选项会显著影响性能，应在所有不能安全假设无溢出的场景中使用。关键要点是：程序员必须确保全局ID值适配32位整型范围，否则应使用-fno-sycl-idqueries-fit-in-int编译选项来保证程序正确性——这可能以降低性能为代价。  

### SIMD执行（使用single_task）（SIMD Execution Using single_task ）
在单任务执行模型下，不存在可供向量化处理的工作项。虽然仍可能实施与向量类型及函数相关的优化，但这取决于编译器实现。编译器和运行时环境有权选择在单任务内核中启用显式SIMD执行或采用标量执行方式，最终执行效果将由具体编译器实现决定。  

C++编译器在针对CPU进行编译时，可能会将单个_task内出现的矢量类型映射为SIMD指令。矢量加载(vec load)、存储(store)和置换(swizzle)函数直接对矢量变量进行操作，这向编译器表明数据元素正在访问内存中从相同(统一)位置开始的连续数据，使我们能够请求对连续数据进行优化加载/存储。如第11章所述（参见第16章CPU编程447页），这种对vec的解释是有效的——但我们应该预期该功能最终会被弃用，转而采用更显式的矢量类型（例如std::simd）一旦该类型可用。  

```cpp
queue q;  
bool *resArray = malloc_shared<bool>(1, q);  
resArray[0] = true;  
q.single_task([=]() {  
    sycl::vec<int, 4> old_v = sycl::vec<int, 4>(0, 100, 200, 300);  
    sycl::vec<int, 4> new_v = sycl::vec<int, 4>();  
    new_v.rgba() = old_v.abgr();  
    int vals[] = {300, 200, 100, 0};  
    if (new_v.r() != vals[0] || new_v.g() != vals[1] || new_v.b() != vals[2] || new_v.a() != vals[3]) {  
        resArray[0] = false;  
    }  
}).wait();
```

在图16-18所示的示例中，单任务执行时声明了一个包含三个数据元素的向量。通过old_v.abgr()执行了混合操作（swizzle operation）。若CPU为某些混合操作提供SIMD硬件指令支持，则应用程序使用此类操作可获得一定的性能优势。  

>  SIMD向量化指南  
>
> <font style="color:#DF2A3F;background-color:#FBF5CB;">CPU处理器实现了具有不同SIMD位宽的SIMD指令集</font>。在许多情况下，这属于实现细节，对于在CPU上执行内核的应用程序是透明的——因为<font style="color:#DF2A3F;background-color:#FBF5CB;">编译器能自动确定最适合特定SIMD尺寸处理的数据元素组，而非要求开发者显式调用SIMD指令</font>。<font style="background-color:#E8F7CF;">子组（sub-group）可更直接地表达需要按SIMD方式执行的数据元素分组场景</font>。 
>
> 考虑到计算复杂度，<font style="background-color:#E8F7CF;">选择最适合向量化的代码结构和数据布局最终能带来更高性能</font>。<font style="color:#DF2A3F;background-color:#FBF5CB;">在设计数据结构时，应尽量选择能使高频计算以SIMD友好方式、以最大并行度访问内存的数据排布、对齐方式和位宽</font>，正如本章所述。  
>

## 总结（Summary）
为充分发挥CPU的线程级并行性和SIMD向量级并行性优势，需谨记以下目标： 

+ 深入理解各类SYCL并行模式及其目标CPU底层架构； 
+ 在硬件资源允许范围内精准匹配线程级并行度——既不过度也不欠缺，可借助厂商分析工具与性能剖析器指导调优； 
+ 关注线程亲和性与内存首次访问对程序性能的影响； 
+ 设计数据结构时优化内存布局、对齐方式和数据位宽，确保高频计算能以SIMD友好方式访问内存，最大化SIMD并行效率； 
+ 权衡掩码操作与条件分支的成本平衡。  
+ 采用清晰的编程风格，尽可能减少潜在的内存别名和副作用问题。 
+ 注意向量类型和接口在扩展性方面的限制。若编译器实现将其映射至硬件SIMD指令，固定长度的向量可能无法良好适配多代CPU或不同厂商CPU的SIMD寄存器位宽。  