本章汇集了一系列在SYCL环境下进行C++编程时卓有成效的实用信息、技巧建议与技术方法。所涉内容皆非详尽论述，旨在启发认知并鼓励读者按需深入研习。  

## 获取代码示例与编译器（Getting the Code Samples and a Compiler）
第一章介绍了如何获取SYCL编译器（例如oneapi.com的实现或github.com/intel/llvm）以及本书代码示例的获取位置（github.com/Apress/data-parallel-CPP）。此处再次强调<font style="color:#DF2A3F;background-color:#FBF5CB;">动手实践示例（包括进行修改！）对于积累实战经验的重要性</font>。加入那些真正了解图1-1代码输出内容的行列吧！  

##  在线资源（Online Resources）  
关键的在线资源包括：

+ 大量资源请访问 sycl.tech/
+ 官方 SYCL 主页位于 khronos.org/sycl/，其中列出了丰富的资源，详见 khronos.org/sycl/resources
+ Resources to help migrate from CUDA to C++ with SYCL at tinyurl.com/cuda2sycl   
（CUDA迁移至SYCL C++的辅助资源，详见：tinyurl.com/cuda2sycl） 
+ Migration tool GitHub home github.com/oneapi-src/SYCLomatic   
（迁移工具GitHub主页：github.com/oneapi-src/SYCLomatic）  

##  平台模型（Platform Model）
支持SYCL的C++编译器被设计得与我们以往使用的任何C++编译器在操作体验上别无二致。值得从宏观层面理解其内部机制——正是这些机制使得支持SYCL的编译器能够同时为主机（如CPU）和设备生成可执行代码。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL采用的平台模型（图13-1）规定了负责协调控制设备端计算工作的主机架构</font>。第二章阐述任务分配至设备的机制，第四章深入探讨设备编程方法。第十二章则从不同细化层级解析该平台模型的应用。  

正如我们在第2章所讨论的，在使用经过适当配置的SYCL运行时和兼容硬件的系统中，必须始终存在一个可运行的设备。这使得设备代码的编写可以默认至少有一个可用设备。具体选择在哪些设备上运行设备代码由程序控制——<font style="background-color:#E8F7CF;">作为程序员，我们完全有权决定是否以及如何在特定设备上执行代码</font>（设备选择选项将在第12章详细讨论）。  

![图13-1. 平台模型：可抽象使用或具体化应用](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744545239460-1c85128c-743f-4099-846b-3ab244a50f09.png)

### 多架构二进制文件（Multiarchitecture Binaries）  
鉴于我们的目标是通过单一源代码支持异构机器，自然而然地希望最终能生成一个单一的可执行文件。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">多架构二进制文件</font>（又称胖二进制文件）是一种经过扩展的单一可执行文件，其内<font style="background-color:#E8F7CF;">包含异构计算设备所需的所有编译代码与中间代码</font>。这种文件与常规的a.out或a.exe无异，但<font style="background-color:#E8F7CF;">囊括了异构系统的完整运行要素，能自动适配不同设备的代码执行需求</font>。如后文所述，胖二进制文件中的设备代码可采用中间格式，将最终指令的生成延迟至运行时完成。  

###  编译模型（Compilation Model）
<font style="color:#DF2A3F;">SYCL的单源性（single-source nature）使其编译过程与常规C++编译行为无异</font>。我们<font style="background-color:#E8F7CF;">无需为设备调用额外的编译步骤，也无需手动捆绑设备与主机代码——这些均由编译器自动处理</font>。当然，理解底层运行机制至关重要：当我们需要更精准地针对特定架构优化时，这类知识极具价值；当编译过程中出现故障需要调试时，这些原理同样不可或缺。 

我们将回顾<font style="color:#DF2A3F;background-color:#FBF5CB;">编译模型</font>，以便在需要相关知识时有所准备。由于<font style="color:#601BDE;">该编译模型支持同时在主机和多个设备上执行的代码，编译器、链接器及其他支持工具发出的指令比我们熟悉的（仅针对单一架构的）C++编译过程更为复杂</font>。欢迎来到异构计算的世界！  

这种异构的复杂性被编译器故意隐藏起来，并且“正常工作”。  

编译器可以生成类似于传统 C++ 编译器的特定于目标的可执行代码（提前（AOT）编译，有时称为离线内核编译），也可以生成可以即时的中间表示（ JIT）在运行时编译为特定目标。  

> ** 编译可以是“提前”(aOt) 或“即时”(Jit)。（Compilation can be “ahead-of-time” (aOt) or “just-in-time” (Jit).）**
>
> 若要在程序编译阶段实现提前编译（AOT），必须预先确定目标设备架构。<font style="color:#DF2A3F;">采用即时编译（JIT）技术可增强编译后程序的跨平台移植性，但要求编译器与运行时环境在应用程序运行期间执行额外工作</font>。    
对于大多数设备（包括GPU），最常见的做法是依赖即时编译（JIT compilation）。某些设备（如FPGA）的编译过程可能异常缓慢，因此这类设备通常采用预先编译（AOT compilation）方案。  
>



> **<font style="color:#DF2A3F;">除非您知道使用 aOt 代码有必要（例如 FpGa）或有好处，否则请使用 Jit。（Use Jit unless you know there is a need (e.g., FpGa) or benefit to using aOt code.）</font>**
>
> 默认情况下，当我们为大多数设备编译代码时，设备代码的输出会以中间形式存储。在运行时，系统上的设备驱动程序会即时将中间形式编译为可在设备上运行的代码，以匹配系统当前的可用资源。  
>



> 与 aOt 代码不同，Jit 代码的目标是能够在运行时编译以使用系统上的任何设备。这可能包括程序最初编译为 Jit 代码时不存在的设备。  
>

我们可以要求编译器为特定设备或设备类别进行提前编译。这种方式的优势在于节省运行时开销，但缺点在于增加了编译时间并生成更臃肿的二进制文件！提前编译的代码不像即时编译那样具有可移植性，因为它无法在运行时适配可用硬件。我们可以在二进制文件中同时包含两种编译方式，从而兼得AOT（提前编译）和JIT（即时编译）的优势。  

> 为了最大限度地提高可移植性，即使包含一些 aOt 代码，我们也喜欢在二进制文件中包含 Jit 代码。  
>

预先为特定设备进行编译还有助于我们在构建时检查程序是否能在该设备上正常运行。若采用即时编译，程序有可能在运行时编译失败（可通过第5章介绍的机制捕获此类错误）。本章后续“调试”小节将提供一些调试技巧，而第5章会详细说明如何在运行时捕获这些错误，以避免应用程序被迫中止。  

图13-2展示了从源代码到胖二进制文件（可执行文件）的编译流程。无论我们选择何种组合方式，最终都会合并生成一个胖二进制文件。运行时环境将在应用程序执行时调用该胖二进制文件（这也是我们在主机上直接执行的二进制文件！）。有时我们可能需要针对特定设备单独编译设备代码，并希望将此类独立编译的结果最终合并到胖二进制文件中。这对于FPGA开发尤为重要——由于完整编译（执行综合、布局与布线全流程）耗时极长，且FPGA开发的实际需求也要求避免在运行时系统安装综合工具。图13-3展示了为满足此类需求所支持的捆绑/解绑操作流程。虽然我们可以选择一次性编译所有内容，但在开发过程中，分步编译的方案往往极具实用价值。  

每个支持SYCL的C++编译器都采用具有相同目标的编译模型，但具体实现细节会有所差异。本文展示的特定示意图由DPC++编译器工具链实现团队提供。  

![图13-2. 编译过程：预先编译与即时编译选项](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744545621593-bb430ef8-46ba-4600-9f8f-e1ec82d0b52f.png)

![图13-3. 编译流程：传输捆绑器/解绑器](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744545644637-20e400d9-4878-458a-b70f-6fe50c9a707c.png)

##  上下文：关键须知（Contexts: Important Things to Know）
如第6章所述，<font style="color:#DF2A3F;background-color:#FBF5CB;">上下文（context）</font><font style="background-color:#E8F7CF;">代表一个或多个可执行内核（kernel）的设备集合</font>。我们可以将上下文视作运行时环境存储其工作状态的便捷载体。在大多数SYCL程序中，程序员除了传递上下文对象外，通常不会直接与之交互。  

设备可进一步划分为<font style="color:#DF2A3F;background-color:#FBF5CB;">子设备</font>。这种划分方式有助于问题的分解。由于子设备的处理方式与设备<font style="background-color:#E8F7CF;">完全相同（采用相同的C++类型），所有关于设备分组的论述同样适用于子设备</font>。 

<font style="color:#DF2A3F;">SYCL在抽象层面将设备视为按平台分组</font>。同一平台内的设备可通过共享内存等方式进行交互。<font style="color:#DF2A3F;">属于同一上下文(context)的设备必须能够通过某种机制访问彼此的全局内存</font>。<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL统一共享内存</font><font style="color:#DF2A3F;">(USM，第6章)仅能在同一上下文内的设备间共享</font>。<font style="color:#DF2A3F;background-color:#FBF5CB;">USM内存分配</font><font style="color:#DF2A3F;">绑定于上下文而非设备，因此某上下文内的USM分配对其他上下文不可见</font>。故而USM分配仅限于单个上下文内使用——可能只是设备集群的子集。  

<font style="color:#DF2A3F;">上下文不会抽象化硬件无法支持的功能</font>。例如，我们无法创建一个包含两个无法共享内存的GPU的上下文。同一平台暴露的所有设备并不都需要能够被归入同一上下文中。  

创建队列时，我们可以指定其所属的上下文环境。<font style="color:#601BDE;">默认情况下，DPC++编译器会为每个平台实现默认上下文，并自动将新队列分配至该默认上下文</font>。其他SYCL编译器虽可遵循相同机制，但标准并未强制要求如此实施。  

>  **<font style="color:#DF2A3F;">创建上下文的成本很高</font>**——<font style="background-color:#E8F7CF;">上下文越少，我们的应用程序就越高效</font>。  
>

<font style="color:#DF2A3F;">将给定平台的所有设备始终放置在同一上下文中具有</font><font style="color:#DF2A3F;background-color:#FBF5CB;">两个优点</font>：<font style="background-color:#E8F7CF;">（1）由于创建上下文的成本很高，因此我们的应用程序更加高效</font>；<font style="background-color:#E8F7CF;"> (2)允许硬件支持的最大共享（例如USM）</font>。  

## 将 SYCL 添加到现有 C++ 程序中（Adding SYCL to Existing C++ Programs）
在现有C++程序中合理运用并行性是使用SYCL的第一步。若某个C++应用已实现并行执行，这可能是优势也可能是隐患——因为<font style="color:#DF2A3F;">任务划分方式会极大影响后续开发空间</font>。程序员谈及<font style="color:#DF2A3F;background-color:#FBF5CB;">重构并行程序</font>时，所指的正是<font style="background-color:#E8F7CF;">通过</font><font style="color:#DF2A3F;background-color:#E8F7CF;">调整执行流程</font><font style="background-color:#E8F7CF;">与</font><font style="color:#DF2A3F;background-color:#E8F7CF;">数据布局</font><font style="background-color:#E8F7CF;">来适配并行化需求</font>。这是个需要深入探讨的复杂课题，本文仅作简要说明。虽然不存在放之四海皆准的并行化改造方案，但仍有若干要点值得关注。  

在为C++应用程序引入并行化时，一种值得考虑的简便方法是<font style="color:#DF2A3F;background-color:#FBF5CB;">寻找程序中隔离性最强、并行潜力最大的节点</font>。我们可以从此处着手修改，再根据需求逐步向其他区域扩展并行处理。需注意的是，<font style="color:#DF2A3F;background-color:#E8F7CF;">程序重构</font><font style="background-color:#E8F7CF;">（即调整控制流与重新设计数据结构）可能进一步提升并行化潜力，这会使实施过程复杂化</font>。  

一旦在程序中找到并行化机会最大的孤立点，我们就需要考虑如何在该程序点使用SYCL技术。这正是本书后续章节所要传授的内容。  

从<font style="color:#DF2A3F;background-color:#FBF5CB;">高层次来看，引入并行性的关键步骤包括以下内容</font>： 

1. <font style="color:#DF2A3F;">并发安全</font>（在传统 CPU 编程中通常称为线程安全）：调整所有共享可变数据（可以更改并可能对其执行操作的数据）的使用同时）以防止数据竞争。请参阅第 19 章。 

2. <font style="color:#DF2A3F;">介绍并发和/或并行性</font>。 

3. <font style="color:#DF2A3F;">并行性调整</font>（最佳扩展、吞吐量或延迟优化）。  

首先考虑第一步至关重要。许多应用程序已为并发性进行了重构，但仍有大量应用尚未完成。<font style="color:#DF2A3F;background-color:#E8F7CF;">当以SYCL作为并行化的唯一来源时</font><font style="color:#DF2A3F;background-color:#FBF5CB;">，我们重点关注内核内使用数据的安全性，以及可能与主机共享的数据</font>。若程序中还存在其他引入并行性的技术（如OpenMP、MPI、TBB等），则需在SYCL编程基础上额外关注这些问题。需要特别说明的是，<font style="color:#DF2A3F;background-color:#FBF5CB;">单个程序中混合使用多种技术是可行的</font>——<font style="background-color:#E8F7CF;">SYCL不必成为程序中唯一的并行性来源</font>。本书不涉及与其他并行技术混合使用这一高级议题。  

## 使用多编译器时的注意事项（Considerations When Using Multiple Compilers）
<font style="color:#DF2A3F;background-color:#FBF5CB;">支持SYCL的C++编译器也支持与其他C++编译器生成的目标代码（如库文件、目标文件等）进行链接</font>。通常而言，使用多编译器时产生的问题与任意C++编译器场景相同，需考虑名称修饰规则、采用相同标准库、统一调用约定等事项。这些注意事项与我们混用Fortran或C等其他语言编译器时需处理的问题本质相同。  

此外，应用程序必须使用与编译程序时所用编译器配套的SYCL运行时环境。<font style="color:#DF2A3F;background-color:#FBF5CB;">混用不同SYCL编译器与SYCL运行时（SYCL runtimes）存在风险</font>——<font style="background-color:#E8F7CF;">不同运行时可能对关键SYCL对象采用不同的实现方式和数据布局结构</font>。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL与非SYCL源语言的互操作性</font>，是<font style="background-color:#E8F7CF;">指SYCL能够与其他编程语言（如OpenCL、C或CUDA）编写的内核函数或设备函数协同工作，或使用由其他编译器预编译的中间表示代码的能力</font>。有关与非SYCL源语言互操作性的更多信息，请参阅第20章。  

最后，用于编译SYCL设备代码的同一编译器工具链也需要完成我们编译的链接阶段。若使用来自不同编译器工具链的链接器进行链接，将无法生成可运行的程序，因为不支持SYCL的编译器不知道如何正确整合主机代码与设备代码。  

## 调试（Debugging）
本节提供一些实用的调试建议，以缓解并行程序（尤其是面向异构机器的程序）调试过程中特有的难题。 

我们切不可忘记，当应用程序在CPU设备上运行时，我们仍拥有调试的选项。这一调试技巧在第2章中被列为方法#2。由于设备架构通常比通用CPU包含更少的调试钩子，工具往往能更精准地探测CPU上的代码。将所有内容运行于CPU的重要区别在于，许多涉及同步的错误（包括主机与设备间来回传输内存的操作）将会消失。尽管我们最终仍需调试所有此类错误，但这种调试方式支持渐进式排错，从而可以先解决部分错误，再处理其他问题。经验表明，<font style="color:#DF2A3F;background-color:#FBF5CB;">尽可能频繁地在目标设备上运行程序至关重要</font>，同时将代码向CPU（及其他设备）的可移植性纳入调试流程——多设备运行既能帮助暴露问题，也有助于判断所遇错误是否特定于某个设备。  

> **<font style="color:#DF2A3F;">运行在CPU上的调试提示是一个功能强大的调试工具</font>**。  
>

<font style="color:#DF2A3F;background-color:#FBF5CB;">并行编程错误（尤其是数据竞争和死锁）</font><font style="background-color:#E8F7CF;">通常在</font><font style="color:#DF2A3F;background-color:#E8F7CF;">主机端运行全部代码</font><font style="background-color:#E8F7CF;">时更易被工具检测和消除</font>。令人懊恼的是，当程序在主机与设备协同运行时，我们最常遭遇此类并行编程错误导致的故障。此时若能牢记"回退至纯CPU模式"这一强大的调试策略将极为有益。值得欣慰的是，SYCL通过精心设计始终保留这一可轻松调用的调试选项。  

> **<font style="color:#DF2A3F;"> 调试提示如果程序死锁，请检查主机访问器是否被正确销毁以及内核中的工作项是否遵守 SYCl 规范中的同步规则。  </font>**
>

 开始调试时建议使用以下编译器选项： 

+ -g：在输出文件中加入调试信息 
+ -ferror-limit=1：使用C++模板库（如SYCL频繁调用的库）时保持可读性 
+ -Werror -Wall -Wpedantic: 强制编译器执行严格编码规范，帮助避免生成需在运行时调试的错误代码。    

我们确实不必为了在SYCL中使用C++而纠缠于修正那些学究式的警告，因此选择不使用-Wpedantic是可以理解的。  

当我们将代码留待运行时即时编译时，仍有可检查的代码存在。这在很大程度上取决于编译器所采用的层级架构，因此查阅编译器文档以获取建议是明智之举。  

### 调试死锁与其他同步问题（Debugging Deadlock and Other Synchronization Issues）
<font style="color:#DF2A3F;background-color:#FBF5CB;">并行编程依赖于并行任务间的正确协调</font>。数据的使用必须受限于其就绪状态——此类数据依赖性需通过程序逻辑进行编码，以确保行为正确性。  

调试依赖性问题，特别是涉及USM时，当同步/依赖逻辑出现错误可能颇具挑战性。我们可能会遇到程序挂起（无法完成）或间歇性生成错误信息的情况。此类场景中常出现"常规运行失败但调试模式下却完美运行"的现象。这类间歇性故障往往源于依赖关系未通过等待机制、锁、队列提交间的显式依赖等手段实现正确同步。  

实用的调试技巧包括： 

+ 将乱序队列切换为顺序队列
+ 在代码中适当位置插入`queue.wait()`调用  

在调试过程中使用这两种方法（或其中之一）有助于识别依赖信息缺失的位置。若此类修改导致程序故障发生变化或消失，则强烈提示同步/依赖逻辑中存在需修正的问题。修复后，我们便会移除这些临时调试措施。  

### 调试内核代码（Debugging Kernel Code）
在<font style="color:#DF2A3F;background-color:#FBF5CB;">调试内核代码</font>时，建议<font style="background-color:#E8F7CF;">首先在CPU设备上运行</font>（如第2章所述）。第2章中的设备选择器代码可轻松修改，通过接收运行时选项或编译时选项，在我们进行调试时将工作重定向至主机设备。  

在调试内核代码时，SYCL定义了一种可在内核中使用的C++风格流（图13-4）。DPC++编译器还提供了实验性的C风格printf实现，该实现具备实用功能，但存在若干限制。  

```cpp
q.submit([&](handler &h) {  
    stream out(1024, 256, h);  
    h.parallel_for(range{8}, [=](id<1> idx) {  
        out << "Testing my sycl stream (this is work-item ID:" << idx << ")\n";  
    });  
});  
```

在调试内核代码时，经验表明我们应<font style="background-color:#E8F7CF;">将断点设置在parallel_for之前或内部，而非直接置于parallel_for上</font>。若在parallel_for处设置断点，即使执行后续操作后仍可能多次触发断点。这条C++调试建议同样适用于SYCL等模板扩展场景——当编译器展开模板调用时，设在模板调用处的断点会转化为一系列复杂断点（详见第13章实用技巧327节）。某些实现方式或可缓解此问题，但关键在于：<font style="background-color:#E8F7CF;">通过避免直接在parallel_for本身设置断点，我们能在所有实现中规避部分调试困惑</font>。  

### 调试运行时故障（Debugging Runtime Failures）
<font style="color:#DF2A3F;background-color:#E8F7CF;">在即时编译过程中发生运行时错误时，我们可能遇到</font><font style="color:#DF2A3F;background-color:#FBF5CB;">三种情况</font>：一是<font style="background-color:#E8F7CF;">显式使用了当前硬件不支持的功能</font>（如fp16或simd8），二是<font style="background-color:#E8F7CF;">编译器/运行时本身的缺陷</font>，三是<font style="background-color:#E8F7CF;">意外编写了未被检测的无效代码，直到触发运行时异常并生成晦涩的错误信息</font>。这三种情况都可能令人望而生畏。值得庆幸的是，即便是粗略检查也能让我们更清楚地识别问题根源——这可能使我们获得规避问题的知识，或至少能帮助向编译器团队提交简明的缺陷报告。无论哪种情形，了解现有辅助工具都至关重要。  

 我们的程序输出中显示运行时错误的示例如下：  

```cpp
terminate called after throwing an instance of 'sycl::_  V1::runtime_error'  
    what(): Native API failed. Native API returns: ...
```

或

```cpp
terminate called after throwing an instance of 'sycl::_  V1::compile_program_error'  
    what(): The program was built for 1 devices  ...
error: Kernel compiled with required subgroup size 8, which is  unsupported on this platform  
in kernel: 'typeinfo name for main::'lambda'(sycl::_V1::nd_  item<2>)'  
    error: backend compiler failed build.  -11 (PI_ERROR_BUILD_PROGRAM_FAILURE)
```

在此处看到这些异常情况，我们可以意识到宿主程序本可被设计来捕获此类错误。第一个异常展示了访问任何非原生支持API时的通用错误提示（本例中使用了平台不支持的宿主机端内存分配方式）；第二个异常更易理解，因为程序为不兼容设备指定了SIMD8指令集（该设备实际支持的是SIMD16）。运行时编译器故障并不需要终止应用程序——我们可以捕获这些错误，或通过编码规避，或双管齐下。第五章将深入探讨这一主题。  

当我们遇到运行时故障且难以快速调试时，不妨尝试使用提前编译重新构建。若目标设备支持提前编译选项，这往往是个简单的尝试手段，可能产生更易于理解的诊断信息。倘若错误能在编译阶段而非即时编译或运行时显现，编译器提供的错误信息通常比即时编译或运行时产生的有限错误信息更具参考价值，其中往往包含更实用的调试线索。  

图13-5列出了编译器或运行时支持的两种标志及其他环境变量（适用于Windows和Linux系统），用于辅助高级调试。这些是DPC++编译器特有的高级调试选项，旨在检查和控制编译模型。本书未对这些选项进行讨论或使用；其详细说明可在线查阅GitHub项目intel.github.io/llvm-docs/EnvironmentVariables.html及tinyurl.com/IGCoptions。  

![图13-5. DPC++编译器高级调试选项](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744547567583-1b36e284-4192-41e2-9ecf-44dcb0162d0e.png)

本书未对这些选项作进一步说明，此处提及旨在为必要时的高级调试提供途径。这些选项或能帮助我们洞察规避问题或缺陷的方法——源代码可能无意间触发了某些问题，通过修正代码即可解决。若仍无法解决，则这些选项专用于编译器本身的深度调试，因而更适用于编译器开发者而非普通用户。部分高级用户认为这些选项颇具价值，故在此提及，后续章节将不再赘述。如需深入探究，请参阅DPC++编译器GitHub项目：intel.github.io/llvm-docs/EnvironmentVariables.html。  

> **<font style="color:#DF2A3F;">调试技巧 当其他选项都用尽并且我们需要调试运行时问题时，我们会寻找可能为我们提供原因提示的转储工具。</font>**  
>

### 队列分析与相应计时能力（Queue Profiling and Resulting Timing Capabilities）
许多设备支持队列性能分析功能（通过`device::has(aspect::queue_profiling)`检测，关于aspect概念的详细说明请参阅第12章）。该功能通过简洁而强大的接口，可便捷获取队列提交时间、设备端实际开始执行时间、设备端完成时间以及命令完成时间等详细计时信息。相较于使用主机计时机制（如chrono），<font style="color:#DF2A3F;background-color:#FBF5CB;">此类性能分析能提供更精确的设备端时序数据，因其通常不包含主机与设备间的数据传输耗时</font>。具体示例可参见图13-6与图13-7，其输出样例展示于图13-8中。需特别说明的是，图13-8所示的输出样例仅用于演示该技术的可行性，未经优化处理，在任何情况下均不得作为评估特定系统选择优势的依据。  

 aspect::queue_profiling表明该设备支持通过property::queue::enable_profiling进行队列性能分析。对于此类设备，我们可以在构造队列时指定property::queue::enable_profiling——属性列表是队列构造函数的可选最终参数。这样做会激活SYCL运行时对提交到该队列的命令组的性能分析信息捕获。捕获的信息随后可通过SYCL事件类的get_profiling_info成员函数获取。如果队列关联的设备不具备aspect::queue_profiling特性，将导致构造函数抛出带有errc::feature_not_supported错误码的同步异常。  

可通过事件类（event class）的`get_profiling_info`成员函数查询事件的性能分析信息，需指定`info::event_profiling`枚举的某一性能分析参数。每个信息参数的可能取值及限制条件由与该事件关联的SYCL后端规范定义。`info::event_profiling`中的所有参数均在SYCL规范题为"SYCL事件类性能分析描述符"的表格中列明，其概要说明则见于规范附录章节"事件信息描述符"下。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">每个性能分析描述符返回一个时间戳</font>，该<font style="background-color:#E8F7CF;">时间戳表示自某个实现定义的时间基准以来所经过的纳秒数</font>。所有共享同一后端的事件均保证采用相同的时间基准；因此，<font style="color:#DF2A3F;background-color:#FBF5CB;">计算同一后端两个时间戳的差值即可得出对应事件之间所经历的纳秒数</font>。  

最后需要提醒的是，<font style="color:#DF2A3F;background-color:#FBF5CB;">启用事件分析功能确实会增加系统开销</font>，因此<font style="background-color:#FBF5CB;">最佳实践是在开发或调优阶段启用该功能，而在生产环境中将其禁用</font>。  

>  **<font style="color:#DF2A3F;">提示 由于开销很小，因此仅在开发或调整期间启用队列分析 - 在生产中禁用。 </font>** 
>

```cpp
#include <iostream>
#include <sycl/sycl.hpp>
using namespace sycl;

// 本例使用的数组类型及数据规模
constexpr size_t array_size = (1 << 16);
typedef std::array<int, array_size> IntArray;

// 定义向量加法函数（参见图13-7）
void InitializeArray(IntArray &a) {
    for (size_t i = 0; i < a.size(); i++) 
        a[i] = i;
}

int main() {
    IntArray a, b, sum;
    InitializeArray(a);
    InitializeArray(b);

    queue q(property::queue::enable_profiling{});
    
    std::cout << "向量大小: " << a.size() 
              << "\n运行设备: " << q.get_device().get_info<info::device::name>() 
              << "\n";

    VectorAdd(q, a, b, sum);

    return 0;
}
```

```cpp
void VectorAdd(queue &q, const IntArray &a, const IntArray &b, IntArray &sum) {
    range<1> num_items{a.size()};
    buffer a_buf(a), b_buf(b);
    buffer sum_buf(sum.data(), num_items);

    auto t1 = std::chrono::steady_clock::now(); // 开始计时
    event e = q.submit([&](handler &h) {
        auto a_acc = a_buf.get_access<access::mode::read>(h);
        auto b_acc = b_buf.get_access<access::mode::read>(h);
        auto sum_acc = sum_buf.get_access<access::mode::write>(h);

        h.parallel_for(num_items, [=](id<1> i) {
            sum_acc[i] = a_acc[i] + b_acc[i];
        });
    });
    q.wait();

    double timeA = (e.template get_profiling_info<info::event_profiling::command_end>() - 
                   e.template get_profiling_info<info::event_profiling::command_start>());

    auto t2 = std::chrono::steady_clock::now(); // 结束计时
    double timeB = (std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count());

    std::cout << "性能分析: 设备端向量加法完成耗时 " << timeA << " 纳秒\n";
    std::cout << "计时器: 设备端向量加法完成耗时 " << timeB * 1000 << " 纳秒\n";
    std::cout << "计时器比性能分析多耗时 " << (timeB * 1000 - timeA) << " 纳秒\n";
}
```

```cpp
/*
向量大小：65536  
运行设备：Intel(R) UHD Graphics P630 [0x3e96]  
性能分析：向量加法在设备上完成，耗时57602纳秒  
计时器：向量加法在设备上完成，耗时2.85489e+08纳秒  
计时器比性能分析多耗时2.85431e+08纳秒  

向量大小：65536  
运行设备：NVIDIA GeForce RTX 3060  
性能分析：向量加法在设备上完成，耗时17410纳秒  
计时器：向量加法在设备上完成，耗时3.6071e+07纳秒  
计时器比性能分析多耗时3.60536e+07纳秒  

向量大小：65536  
运行设备：Intel(R) Data Center GPU Max 1100  
性能分析：向量加法在设备上完成，耗时9440纳秒  
计时器：向量加法在设备上完成，耗时5.6976e+07纳秒  
计时器比性能分析多耗时5.69666e+07纳秒
*/
```

### 追踪与性能分析工具接口（Tracing and Profiling Tools Interfaces） 
<font style="color:#DF2A3F;background-color:#FBF5CB;">追踪与分析工具</font>能帮助我们理解应用程序的运行时行为，并常常为算法优化提供启示。这些洞察通常具有可移植性，能推广到多种设备类型，因此我们建议<font style="background-color:#E8F7CF;">根据您的平台偏好选用最有价值的追踪与分析工具</font>。当然，对特定平台的深度调优可能需要在目标平台上进行实际操作。<font style="background-color:#E8F7CF;">对于追求最大可移植性的应用，我们建议优先寻找那些能够实现跨平台适配的优化机会</font>。  

当我们的SYCL程序运行在OpenCL运行时之上并使用OpenCL后端时，可以通过OpenCL拦截层（github.com/intel/opencl-intercept-layer）来运行程序。该工具能检查、记录并修改应用程序（或高层运行时）生成的OpenCL命令。它支持多种控制选项，初始推荐设置为ErrorLogging（错误日志）、BuildLogging（构建日志），也可考虑CallLogging（调用日志，但会产生大量输出）。通过DumpProgramSPIRV可实现实用的程序SPIR-V转储功能。OpenCL拦截层是独立工具，不属于任何特定OpenCL实现，因此可与多数SYCL编译器配合使用。  

还有许多其他优秀工具可用于收集 SYCL 开发人员常用的性能数据。它们是开源的 (github.com/intel/pti-gpu) 以及帮助我们入门的示例。  

 以下是最常用的两款工具：   
• **onetrace**：面向OpenCL和Level Zero后端的主机与设备追踪工具，支持DPC++（CPU与GPU）及OpenMP GPU传输功能   
• **oneprof**：面向OpenCL和Level Zero后端的GPU硬件指标采集工具，支持DPC++及OpenMP* GPU传输

这两款工具均利用来自运行时插桩的信息，因此适用于GPU和CPU。任何使用这些运行时的编译器（支持SYCL、ISPC和OpenMP）都能从中受益。建议查阅工具官网以了解其对您具体场景的适用性。通常我们总能找到受支持的平台，并借助这些工具获取程序的有价值信息——即便目标平台并非全部受支持。程序分析所得的多数洞察具有普适性。  

## 初始化数据与访问内核输出（Initializing Data and Accessing Kernel Outputs）
本节将深入探讨一个令SYCL新用户感到困惑的主题，这也是我们在作为SYCL开发者初期最常遇到的典型错误（根据我们的经验）。  

简言之，<font style="color:#DF2A3F;background-color:#FBF5CB;">当我们从主机内存分配（如数组或向量）创建缓冲区时，在缓冲区被销毁前无法直接访问原始的主机内存分配</font>。缓冲区在其整个生命周期内拥有构造时传入的任何主机内存所有权。虽然存在极少数机制允许在缓冲区存活期间访问原始主机内存（如缓冲区互斥锁），但这些高级功能无助于解决本文描述的早期错误。 

> 人人都会犯这个错误——意识到这一点能帮助我们快速调试，而非长时间纠结其中!!!  
>
> **<font style="color:#DF2A3F;">若我们从主机内存分配中构建缓冲区，在缓冲区销毁前，不得直接访问原始主机内存分配</font>**！只要缓冲区存在，其便独占该内存区域。<font style="color:#DF2A3F;background-color:#FBF5CB;">务必明晰缓冲区的生命周期——以及该作用域内的使用规则</font>！  
>

一个常见的错误出现在当主程序访问主机分配内存时，该内存仍被缓冲区所持有。一旦发生这种情况，所有预期都将失效——因为我们无法确定缓冲区正在如何使用该内存分配。若数据出现错误也不足为奇，我们试图读取输出结果的计算内核可能尚未开始运行！如第3章和第8章所述，SYCL框架构建于异步任务图机制之上。在尝试使用任务图操作的输出数据前，必须确保代码已执行至同步点：此时任务图已完成运算并将数据提供给主机。无论是缓冲区的销毁还是主机访问器的创建，都是触发此类同步的操作。  

图13-9展示了一种我们经常编写的常见代码模式：通过关闭定义缓冲区的块作用域来使其销毁。当缓冲区离开作用域并被销毁后，我们便能安全地通过传递给缓冲区构造函数的原始主机分配来读取内核结果。  

```cpp
constexpr size_t N = 1024;  // 在任意可用设备上设置队列
queue q;                    // 在主机端创建容器并初始化
std::vector<int> in_vec(N), out_vec(N);

// 初始化输入和输出向量
for (int i = 0; i < N; i++)
  in_vec[i] = i;
std::fill(out_vec.begin(), out_vec.end(), 0);

// 细节：创建新作用域以便缓冲区自动销毁
{
  // 使用主机分配（此处为vector）创建缓冲区
  buffer in_buf{in_vec}, out_buf{out_vec};

  // 向队列提交内核
  q.submit([&](handler& h) {
    accessor in{in_buf, h};
    accessor out{out_buf, h};
    h.parallel_for(range{N}, [=](id<1> idx) { out[idx] = in[idx]; });
  });

  // 关闭缓冲区所在作用域！
  // 缓冲区析构将等待内核写入完成，并将数据从缓冲区拷贝回主机分配
  // （本例中的std::vector）。缓冲区析构后，可安全访问原in_vec和out_vec
}

// 验证输出是否符合预期
// 警告：必须确保缓冲区已析构，才能安全重用in_vec和out_vec
// 缓冲区存活期间持有这些分配，主机代码使用它们既不安全也无法获取最新值
// 下方代码是安全的，因为前述作用域闭合已确保缓冲区在此处前销毁
for (int i = 0; i < N; i++)
  std::cout << "out_vec[" << i << "]=" << out_vec[i] << "\n";
```

将缓冲区与现有主机内存关联的常见原因有以下两点（如图13-9所示）：

1. <font style="color:#DF2A3F;">简化缓冲区数据初始化</font>。可以直接从已初始化完成的主机内存（由应用程序本身或其他模块完成初始化）构造缓冲区。
2. <font style="color:#DF2A3F;">减少代码输入量</font>。使用右花括号'}'闭合作用域的方式（虽然更容易出错）比创建缓冲区host_accessor的写法更为简洁。  

若我们<font style="color:#DF2A3F;background-color:#FBF5CB;">使用主机分配来转储或验证内核的输出值，需将缓冲区分配置于块作用域（或其他作用域）内，以便控制其析构时机</font>。<font style="color:#DF2A3F;background-color:#E8F7CF;">必须确保在访问主机分配获取内核输出前销毁该缓冲区</font>。图13-9展示了正确操作方式，而图13-10则呈现了缓冲区仍存活时就访问输出值的典型错误。  

> 经验丰富的用户可能更倾向于使用缓冲区销毁（buffer destruction）将内核结果数据返回至主机内存分配。但对于大多数用户，尤其是新开发者，建议使用作用域主机访问器（scoped host accessors）。  
>

```cpp
constexpr size_t N = 1024;  // 在任意可用设备上设置队列
queue q;  // 创建主机容器用于主机端初始化
std::vector<int> in_vec(N), out_vec(N);  

// 初始化输入输出向量
for (int i = 0; i < N; i++) in_vec[i] = i; 
std::fill(out_vec.begin(), out_vec.end(), 0);  

// 使用主机分配（本例中为vector）创建缓冲区
buffer in_buf{in_vec}, out_buf{out_vec};  

// 向队列提交内核
q.submit([&](handler& h) {
    accessor in{in_buf, h};
    accessor out{out_buf, h};

    h.parallel_for(range{N}, [=](id<1> idx) {
        out[idx] = in[idx];
    });
});

// 错误！！！我们正在使用主机分配out_vec，但缓冲区out_buf仍然存在并拥有该分配！
// 由于内核可能尚未运行，且缓冲区没有理由将任何输出复制回主机（即使内核已运行）
// 我们很可能会看到初始化值（零）被打印出来
for (int i = 0; i < N; i++) 
    std::cout << "out_vec[" << i << "]=" << out_vec[i] << "\n";
```

为避免这些错误，我们建议<font style="color:#DF2A3F;">在开始使用SYCL的C++编程时，优先采用主机访问器而非缓冲区作用域机制</font>。主机访问器允许从主机端访问缓冲区，且一旦其构造函数执行完毕，即可确保此前对缓冲区的所有写入操作（例如在创建host_accessor前提交的内核任务）均已执行完成且数据可见。本书将混合使用两种风格（即主机访问器与通过缓冲区构造函数传递的主机分配），以便读者熟悉这两种方式。<font style="color:#DF2A3F;">对于初学者而言，采用主机访问器通常更不易出错</font>。图13-11展示了如何在不先销毁缓冲区的情况下，使用主机访问器读取内核计算的输出结果。  

```cpp
constexpr size_t N = 1024;  // 在任意可用设备上设置队列  
queue q;  // 创建主机端容器用于初始化  
std::vector<int> in_vec(N), out_vec(N);  

// 初始化输入和输出向量  
for (int i = 0; i < N; i++) in_vec[i] = i;  
std::fill(out_vec.begin(), out_vec.end(), 0);  

// 基于主机分配（此处使用vector）创建缓冲区  
buffer in_buf{in_vec}, out_buf{out_vec};  

// 向队列提交内核  
q.submit([&](handler& h) {  
    accessor in{in_buf, h};  
    accessor out{out_buf, h};  
    h.parallel_for(range{N}, [=](id<1> idx) {  
        out[idx] = in[idx];  
    });  
});  

// 验证输出是否符合预期值  
// 使用主机访问器！缓冲区仍在作用域内/存活  
host_accessor A{out_buf};  
for (int i = 0; i < N; i++)  
    std::cout << "A[" << i << "]=" << A[i] << "\n";  
```

当缓冲区处于活跃状态时（例如在典型缓冲区生命周期的两端——初始化缓冲区内容和读取内核计算结果时），均可使用主机访问器。图13-12展示了这种模式的示例。  

```cpp
constexpr size_t N = 1024;  // 在任何可用设备上设置队列  
queue q;  // 创建大小为N的缓冲区  
buffer<int> in_buf{N}, out_buf{N};  

// 使用主机访问器初始化数据  
{ // 关键：开始host_accessor的生命周期作用域！  
    host_accessor in_acc{in_buf}, out_acc{out_buf};  
    for (int i = 0; i < N; i++) {  
        in_acc[i] = i;  
        out_acc[i] = 0;  
    }  
} // 关键：关闭作用域使主机访问器失效！  

// 向队列提交内核  
q.submit([&](handler& h) {  
    accessor in{in_buf, h};  
    accessor out{out_buf, h};  
    h.parallel_for(range{N}, [=](id<1> idx) {  
        out[idx] = in[idx];  
    });  
});  

// 检查所有输出是否符合预期值  
// 使用主机访问器！缓冲区仍在作用域内/存活  
host_accessor A{out_buf};  
for (int i = 0; i < N; i++)  
    std::cout << "A[" << i << "]=" << A[i] << "\n";  
```

需要补充的最后一个细节是，主机访问器有时会在应用程序中引发相反的错误，因为它们同样具有生命周期。当缓冲区的主机访问器处于活动状态时，运行时系统将禁止任何设备使用该缓冲区！由于运行时不会分析主机程序以判断其何时可能访问主机访问器，因此它<font style="color:#DF2A3F;">确认主机程序已完成缓冲区访问的唯一方式，就是等待主机访问器的析构函数执行</font>。如图13-13所示，如果主机程序正在等待某些内核运行（例如调用queue::wait()或获取另一个主机访问器），而SYCL运行时又需要等待之前的主机访问器销毁后才能运行使用该缓冲区的内核，这种情况可能导致应用程序看似挂起。  

> 在使用主机访问器时，务必确保在不再需要时及时销毁它们，以解除对缓冲区被内核或其他主机访问器使用的锁定。  
>

```cpp
constexpr size_t N = 1024;  // 在任何可用设备上设置队列
queue q;                    // 使用主机分配创建缓冲区（本例中使用vector）
buffer<int> in_buf{N}, out_buf{N};

// 使用主机访问器初始化数据
host_accessor in_acc{in_buf}, out_acc{out_buf};
for (int i = 0; i < N; i++) {
  in_acc[i] = i;
  out_acc[i] = 0;
}

// 错误：主机访问器in_acc和out_acc仍存活！
// 后续的q.submit调用将无法在设备上启动，因为运行时无法感知
// 我们已通过主机访问器完成对缓冲区的访问。由于主机优先获得访问权
// （早于队列提交），设备内核必须等待主机完成缓冲区更新才能启动。
// 程序将在此处表现为挂起！此时请使用调试器。

// 将内核提交至队列
q.submit([&](handler& h) {
  accessor in{in_buf, h};
  accessor out{out_buf, h};

  h.parallel_for(range{N}, [=](id<1> idx) { out[idx] = in[idx]; });
});

std::cout << "此程序将在此处死锁！！！用于数据初始化的\n"
          << "host_accessors仍处于作用域内，因此运行时不允许\n"
          << "内核在设备上启动执行（主机可能仍在初始化内核\n"
          << "所需数据）。下一行代码将获取输出缓冲区的主机访问器，\n"
          << "该操作会等待内核先运行。由于in_acc和out_acc尚未\n"
          << "析构，运行时认为运行内核不安全，导致死锁。\n";

// 检查输出是否符合预期值
// 使用主机访问器！缓冲区仍在作用域内/存活
host_accessor A{out_buf};

for (int i = 0; i < N; i++)
  std::cout << "A[" << i << "]=" << A[i] << "\n";
```

### 多种翻译单元（Multiple Translation Units）
当我们需要在内核中调用定义于其他翻译单元的函数时，这些函数必须被标记为SYCL_EXTERNAL。若缺少该修饰符，编译器将仅将该函数编译为供设备代码外部使用（从而导致从设备代码内部调用该外部函数的行为被视为非法）。  

SYCL_EXTERNAL函数存在若干限制，但这些限制不适用于在同一翻译单元内定义的函数：

+ SYCL_EXTERNAL仅能用于函数声明
+ SYCL_EXTERNAL函数的参数或返回值类型不得使用原始指针，必须使用显式指针类替代
+ SYCL_EXTERNAL函数不得调用parallel_for_work_item方法
+ SYCL_EXTERNAL函数不可在parallel_for_work_group作用域内调用  

若我们尝试编译一个调用了非同一翻译单元内且未以SYCL_EXTERNAL声明的函数的内核时，预期会出现如下类似的编译错误：  

```cpp
error: SYCL kernel cannot call an undefined function without  SYCL_EXTERNAL attribute
```

 如果函数本身在编译时未使用SYCL_EXTERNAL属性，我们可能会遇到链接错误或运行时故障，例如  

```cpp
terminate called after throwing an instance of '...compile_  
program_error'...  
    error: undefined reference to ...
```

SYCL不要求编译器必须支持SYCL_EXTERNAL；该功能通常是可选的。DPC++支持SYCL_EXTERNAL功能。  

## 多翻译单元的性能影响（Performance Implication of Multiple Translation Units）
编译模型的一个隐含含义（参见本章前文）是：若将设备代码分散在多个翻译单元中，相较于集中存放设备代码，可能会触发更多即时编译的调用。这一现象高度依赖于具体实现方式，并会随着实现的逐步成熟而发生变化。  

这种对性能的影响在我们大部分开发工作中微不足道，可以忽略不计。但当我们需要进行精细调优以最大化代码性能时，可以考虑采取以下两种措施来缓解这些影响：(1) 将设备代码集中编译到同一个翻译单元；(2) 采用提前编译来彻底避免即时编译的影响。由于这两项措施都需要我们付出额外努力，因此我们只在完成开发工作并试图榨取应用程序最后一丝性能时才会采用。当进行此类深度调优时，值得通过测试修改来观察它们对我们所使用的具体SYCL实现产生的影响。  

## 匿名Lambda的命名之道（When Anonymous Lambdas Need Names）
<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL允许为lambda表达式命名，以便工具链在需要时使用以及用于调试目的</font>（例如，启用基于用户自定义名称的显示功能）。根据SYCL 2020规范，lambda命名是可选项。本书大部分内容中，内核均采用匿名lambda形式实现，因为<font style="color:#DF2A3F;">结合SYCL使用C++时通常无需命名</font>（第10章讨论的用于传递编译选项的lambda命名场景除外）。  

当我们需要在代码库中混合使用多个供应商的SYCL工具时，工具链可能要求为lambda表达式命名。这一操作通过在使用该lambda的SYCL操作构造（例如parallel_for）中添加<class uniquename>来实现。此命名机制使得来自不同供应商的工具能够在单一编译过程中以预定义的方式交互，同时也有助于在调试工具和层级中显示我们定义的内核名称。  

若<font style="color:#DF2A3F;background-color:#E8F7CF;">需使用内核查询功能，我们还需为内核命名</font>。SYCL标准委员会未能在SYCL 2020标准中就此强制要求达成解决方案。例如，<font style="background-color:#E8F7CF;">查询内核的preferred_work_group_size_multiple属性时，需调用kernel类的get_info()成员函数，这要求获取kernel类的实例，而该实例又需要我们知晓内核名称（及其kernel_id）才能从相应的kernel_bundle中提取其句柄</font>。  

## 总结（Summary）
当今流行文化常将实用技巧称为"生活妙招"。遗憾的是，编程文化中"hack"一词常带有负面含义，因此作者未将本章命名为"SYCL妙招"。毫无疑问，本章所探讨的C++与SYCL结合使用的实用技巧仅是冰山一角。随着我们共同探索如何充分发挥C++与SYCL的协同效应，更多实用技巧将由众人共同发掘。  