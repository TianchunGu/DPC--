 我们需要探讨作为并行程序"指挥者"的角色。一个编排得当的并行程序堪称艺术杰作——<font style="color:#DF2A3F;">代码全速运行而无须等待数据</font>，因为我们已安排好所有数据在恰当时刻到达与离开；代码经过精心构建，使硬件始终保持最大负载。这简直是梦想铸就的奇迹！  

疾驰人生——岂止一条赛道！——要求我们以指挥家的严谨态度对待工作。为此，不妨将职责构想为任务图谱。  

因此，<font style="color:#DF2A3F;background-color:#FBF5CB;">本章将重点讲解任务图（task graphs）</font>——这一用于正确高效运行复杂内核序列的机制。<font style="color:#DF2A3F;background-color:#FBF5CB;">应用程序中有两类需要排序的操作</font>：<font style="background-color:#E8F7CF;">内核执行（kernel executions）</font>与<font style="background-color:#E8F7CF;">数据迁移（data movement）</font>。任务图正是我们实现精准排序的核心机制。  

首先，我们将快速回顾如何利用依赖关系对第3章中的任务进行排序。接着，我们将介绍SYCL运行时如何构建计算图，并讨论其基本构成单元——命令组（command group）。随后，我们将阐述构建常见模式计算图的不同方法，同时分析显式和隐式数据移动在计算图中的表现形式。最后，我们将探讨与主机端同步计算图的各种实现方式。  

## 什么是图调度?(What Is Graph Scheduling?)
第三章讨论了数据管理及数据使用顺序问题。该章节阐述了SYCL中图机制背后的核心抽象概念——依赖关系(dependences)。<font style="color:#DF2A3F;background-color:#FBF5CB;">内核之间的依赖本质上取决于内核所访问的数据</font>。<font style="background-color:#E8F7CF;">内核必须确保在计算输出结果前读取到正确的数据</font>。  

我们阐述了确保正确执行所需的三种关键数据依赖性。第一种是<font style="color:#DF2A3F;background-color:#FBF5CB;">写后读（RAW）依赖</font>，<font style="background-color:#E8F7CF;">发生于某一任务需要读取另一任务生成的数据时</font>。此类依赖描述了不同计算内核间的数据流向。第二种依赖性出现在<font style="background-color:#E8F7CF;">某一任务需在另一任务读取数据后对其进行更新时</font>，我们称之为<font style="color:#DF2A3F;background-color:#FBF5CB;">读后写（WAR）依赖</font>。最后一种数据依赖性发生在<font style="background-color:#E8F7CF;">两个任务试图写入相同数据时</font>，即<font style="color:#DF2A3F;background-color:#FBF5CB;">写后写（WAW）依赖</font>。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">数据依赖是我们用于构建图的基本单元</font>。这一系列依赖关系足以表达简单的线性内核链，也能描述包含数百个内核且具有复杂依赖关系的大型图结构。无论计算需求涉及何种类型的图，SYCL图都能确保程序根据所表达的依赖关系正确执行。但<font style="background-color:#E8F7CF;">程序员需确保图中的依赖关系能准确反映程序中的所有数据依赖</font>。  

## SYCL中图结构的运作原理(How Graphs Work in SYCL)
<font style="color:#DF2A3F;background-color:#FBF5CB;">命令组可包含三种不同内容</font>：<font style="background-color:#E8F7CF;">动作（action）</font>、其<font style="background-color:#E8F7CF;">依赖项（dependencies）</font>以及杂项<font style="background-color:#E8F7CF;">主机代码（host code）</font>。这三者中，<font style="background-color:#E8F7CF;">动作是必须存在的要素——若无动作，命令组将失去实际功能。大多数命令组还会声明依赖项，但某些情况下可能无需声明</font>。例如,程序中提交的首个动作在启动执行时不依赖任何前置条件，因此无需指定依赖项。命令组内另一种可能出现的内容是运行于主机端的任意C++代码。这种设计完全合法，且能有效辅助动作或其依赖项的声明。需注意的是，此类代码会在命令组创建时立即执行（而非等到依赖条件满足后动作触发时才执行）。  

命令组通常以传递给submit方法的C++ lambda表达式形式呈现。命令组也可通过队列对象上的快捷方法来表示，这些方法接收内核函数及一组基于事件的依赖关系。  

### 命令组动作(Command Group Actions)
命令组可执行的操作分为两类：<font style="color:#DF2A3F;">内核执行</font>与<font style="color:#DF2A3F;">显式内存操作</font>。每个命令组仅能执行单一操作。如先前章节所述，内核通过调用`parallel_for`或`single_task`方法定义，用于表达需要在设备上执行的计算任务。显式数据移动操作属于第二类操作，以USM（统一共享内存）为例，其典型操作包括`memcpy`（内存复制）、`memset`（内存置位）和填充（fill）操作；而缓冲区操作则涵盖复制（copy）、填充（fill）及`update_host`（主机数据更新）等功能。  

### 命令组如何声明依赖关系(How Command Groups Declare Dependences)
命令组的另一个主要组成部分是必须在组定义的操作执行前满足的一系列依赖关系。SYCL允许通过多种方式指定这些依赖关系。  

如果一个程序使用了顺序 SYCL 队列，该队列的顺序语义规定了连续入队的命令组之间存在隐式依赖关系。前一个提交的任务完成之前，后续任务无法执行。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">基于事件的依赖关系是另一种指定命令组执行前必须完成条件的方式</font>。这类依赖可通过两种形式定义：第一种形式适用于将命令组以`lambda`表达式形式传入队列提交方法的情况。此时，程序员需调用命令组处理器对象的`depends_on`方法，并以单个事件或事件向量作为参数。第二种形式适用于通过队列对象上定义的快捷方法创建命令组的情形。当程序员直接在队列上调用`parallel_for`或`single_task`方法时，可将事件或事件向量作为附加参数传入。  

指定依赖关系的最后一种方式是通过创建访问器对象（accessor objects）。访问器通过声明其将如何读写缓冲区对象（buffer object）中的数据，使得运行时系统能够利用这些信息判定不同内核间存在的数据依赖关系。正如本章开篇所述，数据依赖的典型场景包括：某个内核读取另一个内核生成的数据、多个内核写入同一数据区、或某个内核在另一内核读取数据后对该数据进行修改。  

### 例子（Examples）
现在我们将通过几个例子来演示刚刚学到的所有内容。我们将<font style="color:#DF2A3F;background-color:#FBF5CB;">展示如何以多种方式表达两种不同的依赖模式</font>。要说明的两种模式分别是：<font style="background-color:#E8F7CF;">线性依赖链</font>（即一个任务在另一个任务之后执行）和<font style="background-color:#E8F7CF;">"Y"型模式</font>（即两个独立任务必须完成后才能执行后续任务）。  

这些依赖模式的图示见图8-1与8-2。图8-1展示的是线性依赖链：首节点表示数据初始化阶段，次节点则代表将数据归约为单一结果的规约操作。图8-2呈现的是"Y"型模式——我们分别初始化两组不同数据，待数据初始化完成后，通过加法核函数将两个向量求和，最终由图中末节点将结果归约为单一数值。  

![图8-1. 线性依赖链图](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744771483992-a31ce208-846d-40d9-8e00-26a3e2fe4d49.png)

![图8-2. "Y"型依存关系图](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744771493372-594f3bea-ec31-4d9b-920d-5694e8b9ff65.png)

对于每种模式，我们将展示三种不同的实现方式。第一种实现将采用顺序队列；第二种实现将利用基于事件的依赖关系；最后的实现则会通过缓冲区和访问器来表达命令组之间的数据依赖关系。  

图8-3展示了<font style="color:#DF2A3F;">如何使用有序队列表达线性依赖链</font>。由于<font style="background-color:#E8F7CF;">有序队列的语义已天然保证命令组间的顺序执行关系</font>，这个示例显得十分简明。我们提交的第一个内核将数组元素初始化为1，随后提交的内核则将这些元素求和并存储至首元素。由于采用有序队列，我们无需额外操作即可确保第二个内核必须在前一个内核完成后执行。最后等待队列完成所有任务执行，并验证是否获得预期结果。  

```cpp
#include <sycl/sycl.hpp> 
using namespace sycl; 
constexpr int N = 42;  

int main() { 
    queue q{property::queue::in_order()};  
    int *data = malloc_shared<int>(N, q);  
    q.parallel_for(N, [=](id<1> i) { data[i] = 1; });  
    q.single_task([=]() { 
        for (int i = 1; i < N; i++) 
            data[0] += data[i]; 
    }); 
    q.wait();  
    assert(data[0] == N); 
    return 0; 
}
```

图8-4展示了<font style="color:#DF2A3F;">使用乱序队列和基于事件依赖关系的相同示例</font>。在此例中，我们捕获了首次调用`parallel_for`时返回的事件。随后，第二个内核通过将该事件作为参数传递给`depends_on`方法，即可声明对该事件及其所代表内核执行的依赖关系。图8-6将展示如何利用定义内核的快捷方法来简化第二个内核的表达形式。  

```cpp
#include <sycl/sycl.hpp>
using namespace sycl;
constexpr int N = 42;

int main() {
    queue q;
    int *data = malloc_shared<int>(N, q);
    auto e = q.parallel_for(N, [=](id<1> i) { data[i] = 1; });
    q.submit([&](handler &h) {
        h.depends_on(e);
        h.single_task([=]() {
            for (int i = 1; i < N; i++) data[0] += data[i];
        });
    });
    q.wait();
    assert(data[0] == N);
    return 0;
}
```

 图8-5<font style="color:#DF2A3F;">展示了使用缓冲区和访问器替代USM指针重构的线性依赖链示例</font>。本例再次采用无序队列，但通过基于访问器的数据依赖（而非基于事件的依赖）来调度命令组执行顺序。第二内核读取第一内核产生的数据，运行时能识别这种关系——因为我们声明了基于同一底层缓冲对象的访问器。与先前示例不同，此处不等待队列完成所有任务执行，而是构造主机访问器，在第二内核输出与主机端正确性断言之间建立数据依赖。需注意：主机访问器虽能提供主机端数据的最新视图，但若缓冲对象创建时指定了原始主机内存，并不能保证该内存已完成更新。除非先销毁缓冲区，或采用第七章所述互斥机制等更高级方法，否则无法安全访问原始主机内存。  

```cpp
#include <sycl/sycl.hpp>
using namespace sycl;
constexpr int N = 42;

int main() {
    queue q;
    buffer<int> data{range{N}};
    
    q.submit([&](handler &h) {
        accessor a{data, h};
        h.parallel_for(N, [=](id<1> i) {
            a[i] = 1;
        });
    });
    
    q.submit([&](handler &h) {
        accessor a{data, h};
        h.single_task([=]() {
            for (int i = 1; i < N; i++)
                a[0] += a[i];
        });
    });
    
    host_accessor h_a{data};
    assert(h_a[0] == N);
    return 0;
}
```

 图8-6展示了<font style="color:#DF2A3F;">如何使用顺序队列表达"Y"形模式</font>。本例中我们声明了data1和data2两个数组，随后定义两个分别初始化数组的内核。这两个内核虽无相互依赖，但由于队列具有顺序性，内核必须串行执行。注意本例中交换两个内核的执行顺序完全合法。当第二个内核执行完毕后，第三个内核会将第二个数组元素累加到第一个数组中。最终的求和内核将首个数组元素累加，得到与线性依赖链示例相同的结果。该求和内核依赖于前序内核，这种线性依赖关系同样被顺序队列所捕获。最后我们等待所有内核执行完毕，并验证成功计算出目标幻数。  

```cpp
#include <sycl/sycl.hpp>  
using namespace sycl;  
constexpr int N = 42;  

int main() {  
    queue q{property::queue::in_order()};  
    int *data1 = malloc_shared<int>(N, q);  
    int *data2 = malloc_shared<int>(N, q);  

    q.parallel_for(N, [=](id<1> i) { data1[i] = 1; });  
    q.parallel_for(N, [=](id<1> i) { data2[i] = 2; });  
    q.parallel_for(N, [=](id<1> i) { data1[i] += data2[i]; });  

    q.single_task([=]() {  
        for (int i = 1; i < N; i++) data1[0] += data1[i];  
        data1[0] /= 3;  
    });  

    q.wait();  
    assert(data1[0] == N);  
    return 0;  
}
```

图8-7展示了<font style="color:#DF2A3F;">采用乱序队列替代顺序队列的"Y"形模式示例</font>。由于队列顺序不再隐含依赖关系，我们必须通过事件显式定义命令组之间的依赖关系。如图8-6所示，我们首先定义两个无初始依赖关系的独立内核，用事件e1和e2表示这两个内核。当定义第三个内核时，必须指定其依赖于前两个内核。  

我们通过声明该操作需依赖事件e1和e2完成后方可执行来实现这一点。不过在本示例中，我们采用快捷形式来指定这些依赖关系，而非使用处理器的depends_on方法。此处我们将事件作为额外参数传递给parallel_for。由于需要同时传递多个事件，我们采用了接受事件std::vector的重载形式——值得庆幸的是，现代C++通过自动将表达式{e1, e2}转换为相应的向量，为我们简化了这一过程。  

```cpp
#include <sycl/sycl.hpp> 
using namespace sycl;  
constexpr int N = 42;  

int main() {  
    queue q;  
    int *data1 = malloc_shared<int>(N, q);  
    int *data2 = malloc_shared<int>(N, q);  

    auto e1 = q.parallel_for(N, [=](id<1> i) { data1[i] = 1; });  
    auto e2 = q.parallel_for(N, [=](id<1> i) { data2[i] = 2; });  
    auto e3 = q.parallel_for(  
        range{N},  
        {e1, e2},  
        [=](id<1> i) { data1[i] += data2[i]; }  
    );  

    q.single_task(e3, [=]() {  
        for (int i = 1; i < N; i++)  
            data1[0] += data1[i];  
        data1[0] /= 3;  
    });  

    q.wait();  
    assert(data1[0] == N);  
    return 0;  
}
```

 在我们的最后一个示例中（如图8-8所示），我们再次用缓冲区和访问器取代了USM指针和事件。该<font style="color:#DF2A3F;">示例将两个数组data1和data2表示为缓冲区对象</font>。由于必须将访问器与命令组处理器关联，我们的内核不再使用定义内核的快捷方法。第三个内核仍需捕获对前两个内核的依赖关系，此处通过声明缓冲区的访问器实现。由于先前已为这些缓冲区声明过访问器，运行时系统能够正确排序这些内核的执行顺序。此外，在声明访问器b时，我们还向运行时系统提供了额外信息——添加`read_only`访问标签以表明仅读取该数据而不会生成新值。正如在线性依赖链的缓冲区与访问器示例中所见，最终内核通过更新第三个内核产生的数值来实现自我排序。我们通过声明主机访问器来获取计算的最终结果，该访问器将等待最后一个内核执行完毕，再将数据回传至主机端以供读取，从而验证计算结果的正确性。  

```cpp
#include <sycl/sycl.hpp> 
using namespace sycl; 
constexpr int N = 42;  

int main() { 
    queue q;  
    buffer<int> data1{range{N}}; 
    buffer<int> data2{range{N}};  

    q.submit([&](handler &h) { 
        accessor a{data1, h}; 
        h.parallel_for(N, [=](id<1> i) { a[i] = 1; }); 
    });  

    q.submit([&](handler &h) { 
        accessor b{data2, h}; 
        h.parallel_for(N, [=](id<1> i) { b[i] = 2; }); 
    });  

    q.submit([&](handler &h) { 
        accessor a{data1, h}; 
        accessor b{data2, h, read_only}; 
        h.parallel_for(N, [=](id<1> i) { a[i] += b[i]; }); 
    });  
    q.submit([&](handler &h) { 
        accessor a{data1, h}; 
        h.single_task([=]() { 
            for (int i = 1; i < N; i++) 
                a[0] += a[i];  
            a[0] /= 3; 
        }); 
    });  

    host_accessor h_a{data1}; 
    assert(h_a[0] == N); 
    return 0; 
}
```

### 命令组的各个部分何时执行？(When Are the Parts of a Command Group Executed?)
由于任务图是异步的，我们自然会想知道命令组的确切执行时机。至此应当明确的是：内核在其依赖条件满足后即可立即执行，但命令组的主机端部分又会发生什么？  

当一个命令组被提交至队列时，它会在主机上立即执行（在提交调用返回前）。该命令组的主机部分仅执行一次。命令组中定义的所有内核或显式数据操作将被加入队列，以便在设备上执行。  

## 数据移动(Data Movement)
数据移动是SYCL中图的另一个非常重要的方面，对于理解应用程序性能至关重要。然而，如果数据移动在程序中隐式发生（无论是通过缓冲区和访问器，还是通过USM共享分配），它常常会被无意忽视。接下来，我们将探讨数据移动可能影响SYCL中图执行的不同方式。  

### 显式数据移动(Explicit Data Movement)
显式数据移动的优势在于，它明确呈现在计算图中，使程序员能够清晰理解图中的执行过程。我们将<font style="background-color:#E8F7CF;">显式数据操作划分为</font><font style="color:#DF2A3F;background-color:#E8F7CF;">USM相关操作</font><font style="background-color:#E8F7CF;">和</font><font style="color:#DF2A3F;background-color:#E8F7CF;">缓冲区相关操作</font><font style="background-color:#E8F7CF;">两类</font>。  

如我们在第6章所学，<font style="color:#DF2A3F;background-color:#E8F7CF;">统一共享内存（USM）中的显式数据移动发生</font><font style="background-color:#E8F7CF;">在需要将数据在设备分配区域与主机之间复制时</font>。这一操作通过队列类和处理器类中均存在的`memcpy`方法实现。提交该操作或命令组会返回一个事件，该事件可用于将此复制操作与其他命令组进行排序。  

通过调用命令组处理程序对象的`copy`或`update_host`方法，可实现带有缓冲区的显式数据移动。`copy`方法可用于在主机内存与设备上的访问器对象之间手动交换数据，其应用场景多样。一个简单示例是对长时间运行的计算序列进行断点保存。通过该方法，数据可从设备以单向方式写入任意主机内存。若使用缓冲区实现相同功能（大多数情况下，即缓冲区创建时未指定`use_host_ptr`），则需先将数据传输至主机，再从缓冲区内存复制到目标主机内存。  

`update_host`方法是拷贝操作的一种高度特化形式。若缓冲区是基于主机指针创建的，该方法会将访问器所表示的数据回拷至原始主机内存。对于通过特殊属性`use_mutex`创建的缓冲区，当程序需要手动同步主机数据时，此功能尤为实用。然而，大多数程序中鲜少出现此类使用场景。  

### 隐性数据移动(Implicit Data Movement)
隐式数据移动可能对SYCL中的命令组和任务图产生潜在影响。<font style="background-color:#E8F7CF;">在隐式数据移动机制下，数据通过SYCL运行时系统或软硬件协同方式在主机与设备间传输</font>。无论采用何种方式，这类拷贝操作均在用户无显式指令的情况下发生。下面我们仍分别讨论USM和缓冲区两种情形。  

<font style="background-color:#E8F7CF;">在使用统一共享内存(USM)时，主机分配和共享分配会触发隐式数据迁移</font>。如第六章所述，主机分配并非真正移动数据，而是远程访问数据；而共享分配可能在主机与设备之间迁移。由于这种迁移是自动发生的，USM的隐式数据迁移与命令组无需开发者干预。但需注意共享分配存在若干值得关注的特性。  

预取操作的工作方式类似于`memcpy`，旨在让运行时系统在内核尝试使用共享分配内存之前就开始迁移数据。然而，与必须复制数据以确保结果正确的`memcpy`不同，预取通常被视为对运行时的性能优化提示，且不会使内存中的指针值失效（而复制到新地址范围的拷贝操作则会导致指针失效）。即使内核开始执行时预取尚未完成，程序仍能正确运行，因此许多代码可能会选择让计算图中的命令组不依赖于预取操作——毕竟它们并非功能性需求。  

缓冲区的使用也存在微妙之处。<font style="background-color:#E8F7CF;">在使用缓冲区时，命令组必须构建访问器来指定数据的使用方式</font>。这些数据依赖关系表达了不同命令组之间的执行顺序，使我们能够构建任务图。然而，带有缓冲区的命令组有时还承担另一项功能：它们规定了数据移动的要求。  

<font style="background-color:#E8F7CF;">访问器（Accessors）用于声明内核（kernel）将对缓冲区（buffer）执行读写操作</font>。其必然推论是：数据必须同时存在于设备端，若未就位，运行时环境必须在内核开始执行前完成数据传输。因此，SYCL运行时必须持续追踪缓冲区最新版本的物理位置，以便调度数据迁移操作。<font style="color:#DF2A3F;">创建访问器实际上会在任务图中生成一个隐式节点——若需数据迁移，运行时必须优先执行该操作，此后提交的内核才被允许执行</font>。  

让我们再次观察图8-8。在本示例中，前两个内核需要将缓冲区data1和data2拷贝至设备端，运行时系统会隐式创建额外的图节点来执行数据传输。当提交第三个内核的命令组时，这些缓冲区很可能仍驻留在设备端，因此运行时系统无需执行额外数据传输。第四个内核的数据同样可能无需额外传输，但主机访问器的创建会要求运行时系统在访问器可用之前，调度缓冲区data1回传至主机端。  

## 与主机同步(Synchronizing withtheHost)
我们将讨论的最后一个主题是如何将图计算与主机同步执行。本章节中我们已多次涉及相关内容，现在我们将系统性地探讨程序实现同步的各种方法。  

主机同步的第一种方法是我们先前诸多示例中常用的一种：<font style="color:#DF2A3F;background-color:#FBF5CB;">等待队列</font>。队列对象提供两种方法——`wait`和`wait_and_throw`，它们会阻塞主机执行，直到所有提交至该队列的命令组完成处理。这种简单方法适用于多数常见场景，但需注意其同步粒度较为粗放。若需要更精细的同步控制（例如为了潜在的性能提升），后文讨论的其他方法可能更适合应用需求。  

宿主同步的下一种方法是<font style="color:#DF2A3F;background-color:#FBF5CB;">基于事件的同步</font>。相较于基于队列的同步，这种方法提供了更高的灵活性，它允许应用程序仅针对特定操作或命令组进行同步。具体实现方式包括：调用事件的等待方法，或调用事件类中可接受事件向量参数的静态等待方法。  

我们已在图8-5和图8-8中见到下一种方法：主机访问器（host accessors）。<font style="background-color:#FBF5CB;">主机访问器具有双重功能</font>：其一，如其名称所示，它们<font style="color:#DF2A3F;">使数据可供主机访问</font>；其二，通过在当前访问的计算图与主机之间建立新的依赖关系，<font style="color:#DF2A3F;">实现设备与主机的同步</font>。这确保回传到主机的数据能准确反映计算图执行后的运算结果。但需再次强调，<font style="background-color:#E8F7CF;">若缓冲区由现有主机内存构建，则原始内存不保证会包含更新后的数值</font>。  

需注意，<font style="color:#DF2A3F;background-color:#FBF5CB;">主机访问器（host accessor）是阻塞式的</font>。<font style="background-color:#E8F7CF;">在数据就绪之前，主机的执行流程无法越过该主机访问器的创建语句</font>。同理，当主机访问器存在并保持数据可用状态时，缓冲区无法在设备端使用。常见做法是将主机访问器置于额外的C++作用域内创建，以便在访问器不再需要时立即释放数据。这实际展示了下一种主机同步方法的典型应用场景。

SYCL中的某些对象在被销毁并调用其析构函数时具有特殊行为。我们刚刚了解到主机访问器（host accessor）如何使数据保留在主机端直至被销毁。缓冲区和图像在被销毁或离开作用域时同样具有特殊行为：当缓冲区被销毁时，它会等待所有使用该缓冲区的命令组完成执行。一旦缓冲区不再被任何内核或内存操作使用时，运行时环境可能需要将数据复制回主机端。若缓冲区是通过主机指针初始化的，或者向`set_final_data`方法传递了主机指针，则会发生此类数据回传。运行时环境将在对象销毁前完成缓冲区数据的回传，并更新对应的主机指针。  

与主机同步的最终方案涉及第七章首次描述的一项非常见特性。回顾缓冲区对象的构造函数可选择性地接收属性列表，其中创建缓冲区时可传递的有效属性之一是`use_mutex`。当以此方式创建缓冲区时，会附加要求：该缓冲区拥有的内存可与主机应用程序共享。对此内存的访问由初始化缓冲区时所用的互斥锁控制。当安全访问与缓冲区共享的内存时，主机能够获取该互斥锁。若无法获取锁，用户可能需要将内存移动操作加入队列以实现与主机的数据同步。此用法具有高度专用性，在大多数DPC++应用程序中较为罕见。  

## 总结（Summary）
在本章中，我们学习了图的概念，以及它们在SYCL中如何构建、调度和执行。我们详细探讨了命令组的定义及其功能，分析了命令组包含的三个要素：依赖关系、操作动作和杂项主机代码。通过事件机制和访问器描述的数据依赖，我们回顾了任务间依赖关系的指定方法。我们认识到命令组中的单一操作可以是内核函数或显式内存操作，并通过多个示例演示了常见执行图模式的构建方式。随后，我们审视了数据移动作为SYCL图重要组成部分的特性，理解了其在图中显式与隐式两种表现形式。最后，我们全面考察了实现主机与图执行同步的各种方法。  

理解程序流程有助于我们了解在调试运行时故障时可打印的调试信息类型。第13章"调试运行时故障"一节中的表格，结合我们目前从本书中获得的知识，将更易于理解。不过，本书并不试图详细讨论这些高级编译器转储信息。

希望这些内容能让你感觉自己已成为图表专家，能够构建从线性链到包含数百个节点及复杂数据与任务依赖关系的巨型图表！在下一章节中，我们将深入探讨有助于提升应用在特定设备上性能的底层细节。  