我们已掌握了如何在设备上部署代码（第二章）与数据（第三章）——现在要做的，就是运用决策艺术来确定如何操作这些资源。为此，我们将补充完善此前有意省略或简化的内容。本章标志着从基础教学示例向实际并行代码的过渡，并对前几章中简要展示的代码样本细节进行深入扩展。  

用一门新的并行语言编写我们的第一个程序可能看起来是一项艰巨的任务，特别是如果我们对并行编程还不熟悉。语言规范并非为应用程序开发者编写，通常假定读者已掌握某些术语知识；它们不会回答类似以下问题：

• 为什么并行性有多种表达方式？   
• 我应该选用哪种并行性表达方法？   
• 关于执行模型，我究竟需要了解多少？  

本章旨在探讨这些问题及其他相关内容。<font style="background-color:#E8F7CF;">我们将介绍数据并行核的概念，通过实际代码示例分析不同核形式的优缺点，并重点阐述核执行模型的核心要素</font>。  

## 内核内的并行性（Parallelism Within Kernels）
近年来，并行内核已成为表达数据并行性的强大手段。<font style="color:#DF2A3F;">基于内核方法的核心设计目标在于实现跨多种设备的可移植性，并确保程序员的高效开发效率</font>。因此，内核通常不会硬编码以适应特定数量或配置的硬件资源（例如核心数、硬件线程数、SIMD[单指令多数据]指令集）。相反，<font style="color:#DF2A3F;background-color:#FBF5CB;">内核通过抽象概念来描述并行性，具体实现（即编译器与运行时的组合）可据此将抽象概念映射到目标设备上可用的硬件并行资源</font>。尽管这种映射方式由具体实现定义，但我们能够（且应当）信任实现方会选择合理的映射策略，从而有效利用硬件并行能力。  

以硬件无关的方式充分发掘并行性，可确保应用程序能够灵活扩展（或缩减）以适应不同平台的性能需求，但...  

>  <font style="color:#DF2A3F;background-color:#FBF5CB;">确保功能可移植性并不等同于确保高性能</font>！  
>

所支持的设备存在显著的多样性，我们必须认识到<font style="color:#601BDE;">不同架构是为不同应用场景设计和优化的</font>。<font style="color:#DF2A3F;background-color:#FBF5CB;">无论使用何种编程语言，若希望在特定设备上实现最高性能，始终需要额外的人工优化工作</font>。此类<font style="color:#DF2A3F;background-color:#E8F7CF;">设备针对性优化包括</font>：<font style="background-color:#E8F7CF;">为特定缓存大小设计数据分块</font>、<font style="background-color:#E8F7CF;">选择可分摊调度开销的工作粒度</font>、<font style="background-color:#E8F7CF;">利用专用指令或硬件单元</font>，最重要的是<font style="background-color:#E8F7CF;">选用合适的算法</font>。部分案例将在第15、16和17章中进一步探讨。  

在应用程序开发过程中，如何在性能、可移植性和生产力之间取得恰当的平衡，是我们都必须面对的挑战——这也是本书无法全面解决的难题。但我们希望证明，<font style="background-color:#E8F7CF;">通过SYCL扩展的C++语言能够提供全套工具，使开发者既能维护通用的可移植代码，又能用同一种高级编程语言实现针对特定平台的优化代码</font>。剩下的就留给读者自行探索了！  

##  循环与内核（Loops vs. Kernels）
迭代循环本质上是一种串行结构：循环的每次迭代均按顺序执行（即依次进行）。优化编译器或许能够判定循环的部分或全部迭代可并行执行，但必须采取保守策略——若编译器智能不足或缺乏足够信息以证明并行执行始终安全，则必须保留循环的顺序语义以确保正确性。

```cpp
for(int i=0;i<N;++i){
    c[i]=a[i]+b[i];
}
```

请看图4-1中的循环结构，它描述了一个简单的向量加法操作。即便在如此简单的情况下，证明该循环可并行执行也非易事：仅当数组c不与a或b内存重叠时，并行执行才是安全的——而这一条件在通常情况下若不通过运行时检查便无法得到证明！为解决此类问题，编程语言陆续引入了新特性，使我们能够为编译器提供额外信息来简化分析（例如通过restrict关键字声明指针不会重叠），或者彻底绕过分析流程（例如声明循环的所有迭代相互独立，或明确定义循环应如何调度至并行资源）。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">并行循环的确切含义存在一定模糊性</font>——由于不同并行编程语言和运行时对该术语的重载使用——但多数常见的并行循环结构实为对顺序循环应用的编译器转换。这类编程模型使我们能够先编写顺序循环，而后再提供关于如何安全并行执行不同迭代的信息。此类模型功能强大，能与其他前沿编译器优化良好协同，并大幅简化并行编程，但也可能使开发者不常在开发早期阶段就充分考虑并行性。  

<font style="background-color:#E8F7CF;">并行内核并非循环结构，亦不具有迭代特性</font>。确切而言，<font style="color:#DF2A3F;">内核描述的是可被多次实例化并应用于不同输入数据的单一操作</font>；<font style="color:#DF2A3F;">当并行启动内核时，该操作的多个实例可被同时执行</font>。  

```cpp
launch N kernel instances {
    int id = get_instance_id();  // unique identifier in [0, N)
    c[id] = a[id] + b[id];
}
```

图4-2展示了使用伪代码将我们的简单循环示例重写为内核的形式。该内核中的并行化机会清晰明确：该内核可由任意数量的实例并行执行，且每个实例独立作用于不同的数据片段。通过将此操作编写为内核，我们声明其可安全地并行运行（且理想情况下应当以并行方式运行）。  

简言之，<font style="color:#DF2A3F;background-color:#FBF5CB;">基于内核的编程并非将并行性改造至现有串行代码的方法，而是一种用于编写显式并行应用程序的方法论</font>。  

> 我们越早将思维从并行循环转向内核，就越容易使用C++与SYCL编写高效的并行程序。  
>

##  多维核（Multidimensional Kernels）
许多其他语言的并行结构是一维的，将工作直接映射到对应的一维硬件资源（如硬件线程数量）。而SYCL中的并行内核是更高层次的概念，其维度更能反映我们代码通常试图解决的问题特性（在一维、二维或三维空间中）。  

然而，我们必须认识到，<font style="color:#DF2A3F;">并行内核提供的多维索引是程序员的一种便利工具，其底层实现可能仍基于一维空间</font>。理解这种映射行为的运作机制，对于某些优化策略（例如调整内存访问模式）而言至关重要。  

一个重要的考量是<font style="color:#601BDE;">确定哪个维度是连续的或单位步长的</font>（即在多维空间中的位置在一维映射中彼此相邻）。<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL中与并行性相关的所有多维量均遵循相同约定</font>：<font style="background-color:#E8F7CF;">维度编号从0到N-1，其中维度N-1对应连续维度</font>。当多维量以列表形式书写（例如在构造函数中）或类支持多个下标运算符时，此编号从左到右适用（左侧始于维度0）。该约定与标准C++中多维数组的行为保持一致。  

图4-3展示了<font style="color:#601BDE;">采用SYCL规范将二维空间映射为线性索引的示例</font>。我们当然可以突破该规范并采用自定义的线性化索引方法，但必须谨慎行事——偏离SYCL规范可能对受益于跨步访问（stride-one accesses）的设备产生性能负面影响。  

![图4-3. 尺寸为(2, 8)的二维范围映射至线性索引](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744035495362-02c36c79-8646-433c-9424-d4532feac473.png)

如果应用程序需要超过三个维度，我们必须手动负责在多维索引和线性索引之间进行映射，使用模运算或其他技术。  

##  语言特性概述（Overview of Language Features）
一旦决定编写并行内核，我们必须<font style="color:#DF2A3F;">确定要启动的内核类型及其在程序中的表示方式</font>。并行内核的表达方式多种多样，若要精通此语言，需逐一熟悉这些选项。  

###  将内核代码与宿主代码分离（Separating Kernels from Host Code）
我们有以下几种替代方案来分离主机代码和设备代码，这些方法可以在应用程序中混合使用：<font style="background-color:#E8F7CF;">C++ lambda表达式</font>或<font style="background-color:#E8F7CF;">函数对象</font>、<font style="background-color:#E8F7CF;">通过互操作性接口（如OpenCL C源代码字符串）定义的内核</font>，或者<font style="background-color:#E8F7CF;">二进制文件</font>。其中部分选项已在第2章介绍，其余内容将在第10章和第20章详细阐述。  

这些并行化表达的基本概念在所有选项中都是共通的。为保持一致性和简洁性，<font style="background-color:#D9DFFC;">本章所有代码示例均采用C++ lambda表达式来呈现内核</font>。  

>  <font style="color:#DF2A3F;">LAMBDA表达式无害论 </font> 
>
> 无需完全理解C++规范中关于lambda表达式的全部内容即可开始使用SYCL——我们只需知道<font style="background-color:#FBF5CB;">lambda表达式的主体代表内核，且（按值）捕获的变量将作为参数传递给内核</font>。
>
> <font style="background-color:#E8F7CF;">使用lambda表达式替代更冗长的内核定义机制不会产生性能损失</font>。支持SYCL的C++编译器始终能识别何时lambda表达式表示并行内核的主体，并相应地为并行执行进行优化。   
如需回顾C++ lambda表达式及其在SYCL中的使用注意事项，请参阅第1章。关于使用lambda表达式定义内核的更具体细节，请参阅第10章。  
>

##  并行内核的不同形式（Different Forms of Parallel Kernels）
<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL中有三种不同的内核形式</font>，分别支持不同的执行模型和语法规范。开发者可采用任意内核形式编写可移植内核，且<font style="background-color:#E8F7CF;">无论采用何种形式编写的内核均可通过调优在各类设备上实现高性能</font>。有时我们可能需要选用特定内核形式，以便更轻松地表达特定并行算法，或利用其他形式无法实现的语言特性。  

第一种形式用于<font style="color:#DF2A3F;">基础的数据并行内核</font>，为编写内核提供最温和的入门方式。使用基础内核时，我们牺牲了对调度等底层功能的控制，以使内核的表达尽可能简单。各个内核实例如何映射到硬件资源完全由实现方式控制，因此随着基础内核复杂度的增加，其性能表现将越来越难以预测。  

第二种形式扩展了基础内核，以提供对底层性能调优功能的访问。出于历史原因，这种形式被称为<font style="color:#DF2A3F;">ND范围（N维范围）数据并行</font>，其中最关键的是它允许将某些内核实例分组，从而让我们能够对数据局部性以及单个内核实例与执行它们的硬件资源之间的映射施加一定控制。

第三种形式提供了一种实验性的替代语法，用于<font style="background-color:#E8F7CF;">通过类似嵌套并行循环的语法来表达ND-range内核</font>。这种形式被称为<font style="color:#DF2A3F;">层次化数据并行</font>，意指用户源代码中出现的嵌套结构所构成的层次体系。目前编译器对该语法的支持尚不成熟，许多SYCL实现处理层次化数据并行内核的效率不及前两种形式。此外，该语法尚不完整——SYCL中诸多提升性能的特性与层次化内核不兼容或无法在其中调用。<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL的层次化并行机制正处于更新阶段，其规范文件特别注明建议新代码暂勿采用层次化并行功能直至该特性完善为止</font>；秉承这一建议精神，<font style="color:#DF2A3F;background-color:#FBF5CB;">本书后续内容将仅讲解基础并行模式与ND-range并行</font>。  

我们将在本章末尾重新讨论如何在不同核函数形式之间进行选择，届时我们将更详细地探讨它们的特性。  

##  基础数据并行内核（Basic Data-Parallel Kernels）  
<font style="color:#2F4BDA;">最基本的并行内核形式适用于</font>"令人尴尬的并行"操作（即可以<font style="color:#2F4BDA;">完全独立且以任意顺序应用于每个数据块的操作</font>）。通过采用这种形式，我们赋予实现方案对工作调度的完全控制权。因此这是描述性编程结构的一个范例——我们声明该操作具有"令人尴尬的并行"特性，而<font style="color:#2F4BDA;">所有调度决策均由实现方案自行裁定</font>。  

基础数据并行核以单程序多数据（SPMD）模式编写——单个"程序"（即核函数）被应用于多组数据。需注意的是，由于数据依赖性分支的存在，<font style="color:#2F4BDA;">该编程模型仍允许每个核函数实例在代码执行过程中采取不同的路径</font>。  

SPMD编程模型的最大优势之一，在于它<font style="color:#DF2A3F;">允许相同的"程序"被映射到多种层级和类型的并行结构中，而无需我们进行任何显式指令</font>。同一程序的实例既可采用流水线方式处理，也能打包后通过SIMD指令执行，或分配到多个硬件线程上运行，甚至实现三者的混合应用。  

###  理解基础数据并行内核（Understanding Basic Data-Parallel Kernels）  
一个基本并行内核的执行空间被称为其执行范围，<font style="background-color:#CEF5F7;">内核的每个实例被称为一个工作项（work-item）</font>。图4-4以图示形式对此进行了说明。  

![图4-4. 基础并行内核的执行空间示意图（展示64个项目的二维范围）](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744175350006-2d887e1a-fa6c-429d-8c90-7e897bda68c3.png)

<font style="color:#DF2A3F;background-color:#FBF5CB;">基础数据并行内核的执行模型</font>非常简单：它<font style="background-color:#E8F7CF;">允许完全并行执行，但并不保证或要求这一点</font>。各项目可以按任意顺序执行，包括在单个硬件线程上顺序执行（即不进行任何并行操作）。因此，若内核假设所有项目都将并行执行（例如通过尝试同步各项目），则极易导致程序在某些设备上挂起。  

然而，<font style="background-color:#CEF5F7;">为确保正确性，我们在编写内核时必须始终假设其可能以并行方式执行</font>。例如，开发者有责任通过原子内存操作（参见第19章）妥善保护对内存的并发访问，以防止竞态条件的发生。  

###  编写基本数据并行内核（Writing Basic Data-Parallel Kernels）
基础数据并行内核通过`parallel_for`函数实现。图4-5展示了如何运用该函数进行向量加法运算——这正是我们为并行加速器编程设计的"Hello, world!"范例。  

```cpp
h.parallel_for(range{N}, [=](id<1> idx) { 
  c[idx] = a[idx] + b[idx]; 
});
```

该函数仅接受两个参数：第一个是用于<font style="background-color:#E8F7CF;">指定各维度启动项数量的范围（或整数）</font>，第二个是为<font style="background-color:#E8F7CF;">范围内每个索引执行的内核函数</font>。内核函数可接受多种不同类别的参数，具体使用哪一类取决于所需功能由哪个类提供——我们将在后文对此进行详细讨论。  

图4-6展示了该函数用于表达矩阵加法的极相似用法——（数学意义上）除使用二维数据外与向量加法完全相同。这种特性通过内核得以体现：两段代码间唯一的差异仅在于所用range类和id类的维度！这种编码方式之所以可行，是因为<font style="color:#DF2A3F;">SYCL访问器支持通过多维id进行索引</font>。尽管形式看似奇特，这种设计却极具威力，<font style="background-color:#E8F7CF;">使我们能够编写基于数据维度的通用模板化内核</font>。  

```cpp
h.parallel_for(range{N, M}, [=](id<2> idx) { 
    c[idx] = a[idx] + b[idx]; 
});  
```

在C/C++中更常见的是使用多个索引和多个下标运算符来访问多维数据结构，这种显式索引方式也受到访问器的支持。采用多索引的方式，当内核同时处理不同维度的数据，或当内核的内存访问模式比直接使用项ID所能描述的更为复杂时，可显著提升代码可读性。  

例如，图4-7中的矩阵乘法核函数必须提取索引的两个独立分量，才能描述两个矩阵行与列的点积运算。作者认为，始终<font style="background-color:#E8F7CF;">使用多重下标运算符（如[j][k]）比混合多种索引模式并构造二维id对象（如id(j,k)）更具可读性</font>，但这纯属个人偏好问题。  

本章后续示例均采用多重下标运算符，以确保所访问缓冲区的维度不存在歧义。  

```cpp
h.parallel_for(range{N, N}, [=](id<2> idx) {  
    int j = idx[0];  
    int i = idx[1];  
    for (int k = 0; k < N; ++k) {  
        c[j][i] += a[j][k] * b[k][i]; // 或 c[idx] += a[id(j,k)] * b[id(k,i)];  
    }  
});  
```

![图4-8. 将矩阵乘法工作映射至执行范围中的项](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744175928976-f09e2462-ed41-4400-8d69-70ea296b97ec.png)

图4-8中的示意图展示了矩阵乘法内核中的工作如何映射到各个计算单元。需注意的是，计算单元的数量由输出矩阵的范围大小决定，且相同的输入值可能被多个计算单元读取：每个计算单元通过顺序遍历A矩阵的（连续）行和B矩阵的（非连续）列，最终计算出C矩阵的单个元素值。  

###  基础数据并行核函数的详细说明（Details of Basic Data-Parallel Kernels）
基础数据并行内核的功能通过三个C++类公开：`range`、`id`和`item`。在前几章中我们已经多次见到`range`和`id`类，但此处我们将以不同的视角重新审视它们。  

####  range类（The range Class）
一段范围（range）表示一维、二维或三维的空间范围。该范围的维度是模板参数，因此<font style="color:#DF2A3F;background-color:#FBF5CB;">必须在编译时确定</font>；但其<font style="color:#DF2A3F;background-color:#FBF5CB;">各维度的大小是动态的，需在运行时通过构造函数传入</font>。范围类的实例既用于描述并行结构的执行范围，也用于描述缓冲区的尺寸。  

图4-9展示了range类的简化定义，包括其构造函数及用于查询范围的各类方法。  

```cpp
template <int Dimensions = 1> class range {  
public:  
    // 构造具有一维、二维或三维范围的range对象  
    range(size_t dim0);  
    range(size_t dim0, size_t dim1);  
    range(size_t dim0, size_t dim1, size_t dim2);  
    // 返回指定维度的范围大小  
    size_t get(int dimension) const;  
    size_t &operator[](int dimension);  
    size_t operator[](int dimension) const;  
    // 返回各维度大小的乘积  
    size_t size() const;  
    // 同时支持范围的算术运算  
};  
```

####  id类（The id Class）  
标识符（id）表示对一维、二维或三维范围进行索引的下标。其定义在多方面与范围（range）类似：<font style="color:#DF2A3F;">其维度同样必须在编译时确定</font>，且<font style="color:#DF2A3F;">可用于并行结构中</font><font style="color:#DF2A3F;background-color:#FBF5CB;">为内核的单个实例编制索引</font><font style="color:#DF2A3F;">，或作为缓冲区的偏移量</font>。  

如图4-10中id类的简化定义所示，从概念上讲，id不过是一个容纳一、二或三个整数的容器。我们可用的操作也非常简单：可以<font style="color:#DF2A3F;">查询每个维度中某个索引的分量</font>，也可以<font style="color:#DF2A3F;">通过简单算术运算来计算新的索引</font>。  

虽然我们可以构造一个id来表示任意索引，但<font style="color:#DF2A3F;">要获取与特定内核实例相关联的id，我们必须将其（或包含它的项）作为内核函数的参数接收</font>。<font style="background-color:#E8F7CF;">该id（或其成员函数返回的值）必须传递给任何需要查询索引的函数</font>——目<font style="background-color:#E8F7CF;">前尚不存在可在程序中任意位置查询索引的自由函数</font>，但未来版本的SYCL可能会对此进行简化。  

<font style="background-color:#D9EAFC;">每个接受 id 的内核实例仅知道其被分配计算的索引范围，而对范围本身一无所知</font>。若希望内核实例知晓自身索引及范围，则需改用 item 类。  

```cpp
template <int Dimensions = 1> class id {  
public:  
    // 构造具有一维、二维或三维的id对象  
    id(size_t dim0);  
    id(size_t dim0, size_t dim1);  
    id(size_t dim0, size_t dim1, size_t dim2);  
    // 返回指定维度的id分量  
    size_t get(int dimension) const;  
    size_t &operator[](int dimension);  
    size_t operator[](int dimension) const;  
    // 支持id的算术运算  
};  
```

#### item类（The item Class）
一个项（item）<font style="color:#DF2A3F;background-color:#FBF5CB;">表示内核函数的一个独立实例</font>，它封装了内核的执行范围以及该实例在该范围内的索引（分别使用range和id表示）。与range和id一样，<font style="background-color:#FBF5CB;">其维度必须在编译时确定</font>。  

图4-11给出了item类的简化定义。item与id的主要区别在于：<font style="background-color:#FBF5CB;">item额外提供了查询执行范围属性（如大小）的功能，并包含计算线性化索引的便捷函数</font>。与id类似，获取特定内核实例关联item的唯一方式，是将其作为内核函数的参数接收。  

```cpp
template <int Dimensions = 1, bool WithOffset = true>
class item {
public:
    // 返回该工作项在内核执行范围中的索引
    id<Dimensions> get_id() const;
    size_t get_id(int dimension) const;
    size_t operator[](int dimension) const;
    // 返回当前工作项所属内核的执行范围
    range<Dimensions> get_range() const;
    size_t get_range(int dimension) const;
    // 返回该工作项的偏移量（当WithOffset == true时）
    id<Dimensions> get_offset() const;
    // 返回该工作项的线性化索引
    // 例如：id(0) * range(1) * range(2) + id(1) * range(2) + id(2)
    size_t get_linear_id() const;
};
```

## 显式ND范围核（Explicit ND-Range Kernels）
并行内核的第二种形式将基础数据并行内核的扁平执行范围替换为分组的执行范围。这种形式最<font style="color:#DF2A3F;background-color:#FBF5CB;">适用于需要在核函数中体现局部性概念的场景</font>。<font style="background-color:#E8F7CF;">针对不同类型的组，定义了特定的行为并予以保证，从而让我们更深入理解或控制工作负载如何映射到特定硬件平台</font>。  

这些显式ND范围内核因此是一种更具规定性的并行结构示例——我们为每种工作组类型规定了工作映射方式，而实现必须遵循该映射规则。然而它并非完全规定性，因为各组本身可以按任意顺序执行，且实现过程仍保留对每种工作组类型如何映射到硬件资源的灵活性。这种规定性与描述性编程的结合，使我们能够在不破坏内核可移植性的前提下，针对局部性进行内核设计与优化。  

与基本的数据并行内核类似，ND范围内核采用SPMD（单程序多数据）风格编写，其中所有工作项都执行相同的内核"程序"，并应用于多个数据片段。关键区别在于：<font style="color:#DF2A3F;">每个程序实例可查询其在所属工作组中的位置，并能访问针对每种组类型特有的附加功能</font>（参见第9章）。  

###  理解显式ND-Range并行内核（Understanding Explicit ND-Range Parallel Kernels）
ND-range内核的执行范围被划分为工作组（`work-groups`）、子组（`sub-groups`）和工作项（`work-items`）。<font style="color:#DF2A3F;background-color:#FBF5CB;">ND-range代表总执行范围</font>，它被划分为大小均匀的工作组（即<font style="color:#DF2A3F;background-color:#FBF5CB;">工作组尺寸必须在每个维度上精确整除ND-range尺寸</font>）。<font style="color:#DF2A3F;background-color:#FBF5CB;">每个工作组可进一步由实现方式划分为子组</font>。理解针对工作项及各类分组定义执行模型，是编写正确且可移植程序的重要环节。  

图4-12展示了一个尺寸为(8,8,8)的ND-range被划分为8个尺寸为(4,4,4)工作组（work-group）的实例。每个工作组包含16个一维子组（sub-group），每个子组含4个工作项（work-item）。需特别注意维度编号规则：<font style="color:#DF2A3F;">子组始终为一维结构</font>，因此ND-range和工作组的第2维度将转换为子组的第0维度。  

![图4-12 划分为工作组、子组和工作项的三维ND范围](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744269320046-3c9939de-287e-445e-8156-b6414f18c539.png)

<font style="color:#DF2A3F;background-color:#FBF5CB;">从各类工作组到硬件资源的具体映射由实现定义，正是这种灵活性使得程序能够在多种硬件上执行</font>。例如，工作项可以完全按顺序执行、通过硬件线程和/或SIMD指令并行执行，甚至由专为核心配置的硬件流水线执行。  

本章仅聚焦于ND-range执行模型在通用目标平台上的语义保证，暂不涉及该模型向具体硬件平台的映射细节。<font style="color:#DF2A3F;background-color:#FBF5CB;">关于GPU、CPU及FPGA的硬件映射实现与性能优化建议，请分别参阅第15、16和17章</font>。  

####  工作项（Work-Items）
<font style="color:#DF2A3F;background-color:#FBF5CB;">工作项代表内核函数的各个实例</font>。<font style="background-color:#E8F7CF;">在没有其他分组的情况下，工作项可以按任意顺序执行，且除了通过全局内存的原子内存操作外（参见第19章），它们无法相互通信或同步</font>。  

####  工作组（Work-Groups）
<font style="color:#DF2A3F;background-color:#FBF5CB;">ND范围中的工作项被组织成工作组</font>。<font style="background-color:#E8F7CF;">工作组的执行顺序可以任意，且不同工作组中的工作项无法相互通信，除非通过全局内存的原子内存操作（参见第19章）</font>。然而，当使用特定结构时，工作组内的工作项具有某些调度保证，这种局部性提供了额外功能：  

1. 工作组中的工作项可访问工作组局部存储器（local memory），该存储器在某些设备上可能映射至专用的高速存储器（fast memory）（参见第9章）。  
2. 工作组中的工作项可通过工作组屏障（work-group barriers）实现同步，并利用工作组内存栅栏（work-group memory fences）确保内存一致性（参见第9章）。  
3. 工作组中的工作项（work-items）可访问群组函数（group functions）——这些函数提供了常见通信例程的实现（参见第9章），以及群组算法（group algorithms）——这些算法实现了诸如归约（reductions）和扫描（scans）等常见并行模式（参见第14章）。  

<font style="color:#DF2A3F;background-color:#FBF5CB;">工作组中的工作项数量通常在运行时针对每个内核进行配置</font>，因为<font style="background-color:#E8F7CF;">最佳分组既取决于可用并行度（即ND范围的大小），也取决于目标设备的特性</font>。我们可以通过设备类的查询函数（参见第12章）获取特定设备支持的工作组内最大工作项数量，并需确保为每个内核请求的工作组大小是有效的。  

工作组执行模型中存在一些值得强调的微妙之处。  

首先，尽管<font style="background-color:#E8F7CF;">工作组中的工作项被调度到单个计算单元上执行</font>，但<font style="color:#DF2A3F;background-color:#FBF5CB;">工作组的数量与计算单元的数量之间并不存在必然关联</font>。实际上，<font style="color:#DF2A3F;">ND-range范围内的工作组数量可能远超设备能够同时执行的工作组数量</font>！我们可能试图编写依赖特定设备调度机制来实现跨工作组同步的内核代码，但必须强烈警告不要采用这种做法——此类内核或许在当前版本能够运行，但无法保证在未来的实现中仍然有效，且极有可能在移植到其他设备时出现故障。  

第二，尽管工作组中的工作项被调度为可以相互协作，但并不要求提供任何特定的进度保证（forward progress guarantees）——在屏障（barriers）和集合（collectives）操作之间按顺序执行工作组内的工作项是一种有效的实现方式。<font style="color:#DF2A3F;background-color:#FBF5CB;">同一工作组内工作项之间的通信和同步，仅在使用提供的屏障和集合函数时才能保证安全</font>，而手动编写的同步例程可能导致死锁。  

>  **团队协作中的思维模式 ** 
>
> 工作组（work-groups）在许多方面与其他编程模型（如线程构建模块Threading Building Blocks）中的任务概念相似：任务可以按任意顺序执行（由调度程序控制）；对机器进行任务超额订阅是可行（甚至可取）的；而尝试在一组任务间实现屏障通常并非良策（因其代价高昂或与调度程序不兼容）。若我们已熟悉基于任务的编程模型，不妨将工作组视为数据并行任务，这种类比或有助理解。  
>

####  子组（Sub-Groups）
<font style="color:#DF2A3F;background-color:#FBF5CB;">在许多现代硬件平台上，工作组内被称为子组的工作项子集会在额外的调度保证下执行</font>。例如，由于编译器向量化的作用，子组中的工作项可能被同时执行；此外，子组本身可能因被映射至独立的硬件线程而具备强前向进度保证。  

在使用单一平台时，我们很容易将关于这些执行模型的假设固化到代码中，但这会导致代码本质上不安全且不可移植——当在不同编译器之间迁移时，甚至在同一供应商的不同代硬件之间迁移时，这些代码都可能出现故障！  

将子组定义为核心语言特性提供了一种安全选择，无需做出可能后续被证明与特定设备相关的假设。利用子组功能还能让我们在底层（即接近硬件层面）推理工作项的执行过程，这是跨多种平台实现极致性能的关键所在。  

与工作组类似，<font style="color:#DF2A3F;background-color:#FBF5CB;">子组内的工作项可通过组函数和组算法实现同步、确保内存一致性或执行通用并行模式</font>。然而，<font style="color:#DF2A3F;background-color:#FBF5CB;">子组并不存在类似工作组本地内存的对应结构（即不存在子组本地内存）</font>。取而代之的是，子组中的工作项可直接交换数据——无需显式内存操作——通过使用一组俗称"洗牌"操作的子集算法（第9章）。  

>  **为何“洗牌”？（WHY “SHUFFLE”?）**
>
> 在诸如OpenCL、CUDA和SPIR-V等语言中，所有"shuffle"操作均在其名称中包含"shuffle"一词（例如sub_group_shuffle、__shfl和OpGroupNonUniformShuffle）。而SYCL则采用了不同的命名约定，以避免与C++标准库中定义的std::shuffle函数（该函数用于随机重排范围内元素的顺序）产生混淆。  
>

<font style="color:#DF2A3F;background-color:#FBF5CB;">子组的某些方面由具体实现定义，不在我们控制范围内</font>。然而，<font style="background-color:#E8F7CF;">对于给定的设备、内核和ND-range组合，子组具有固定的（一维）大小，我们可以通过内核类的查询函数获取这一大小</font>（参见第10章和第12章）。<font style="color:#DF2A3F;">默认情况下，每个子组的工作项数量也由实现选择</font>——我们<font style="background-color:#E8F7CF;">可以在编译时通过请求特定子组大小来覆盖此行为，但必须确保所请求的子组大小与设备兼容</font>。  

与工作组类似，<font style="background-color:#E8F7CF;">子组中的工作项并不要求提供任何特定的前向进度保证</font>——实现方式可以自由选择按顺序执行子组中的每个工作项，仅在遇到子组集体函数时才切换工作项。然而在某些设备上，工作组内的所有子组最终都能确保执行（取得进展），这是若干生产者-消费者模式的基石。当前这种行为由具体实现定义，因此<font style="background-color:#E8F7CF;">若要保持内核的可移植性，我们不能依赖子组的执行进度</font>。我们预计未来版本的SYCL将通过设备查询机制来明确描述子组的进度保证特性。  

<font style="color:#DF2A3F;background-color:#E8F7CF;">在为特定设备编写内核时，工作项到子组的映射关系是已知的，我们的代码常可利用这种映射特性来提升性能</font>。然而常见的错误是假设代码在某一设备上运行正常，就能适用于所有设备。图4-13和4-14展示了将多维内核中范围为{4,4}的工作项映射到子组时的两种可能情形（最大子组大小为8）。图4-13的映射生成两个八工作项的子组，而图4-14的映射则生成四个四工作项的子组！  

![图4-13. 一种可能的子组映射关系，其中允许子组尺寸大于工作组最高编号（连续）维度的范围，因此子组看似"循环环绕"。](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744269828263-f90a61ef-3236-44df-add3-2bd2bda58b9c.png)

![图4-14. 另一种可能的子组映射方式（其中子组规模不允许超过工作组最高编号[连续]维度的范围）](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744269860400-f28ef7c0-f73c-4ae6-a6c1-861e83696ac1.png)

<font style="color:#DF2A3F;background-color:#FBF5CB;">SYCL目前无法查询工作项如何映射到子组，也不支持请求特定映射机制</font>。<font style="color:#DF2A3F;background-color:#FBF5CB;">编写可移植子组代码的最佳方式是使用一维工作组，或采用最高维度可被内核所需子组大小整除的多维工作组</font>。  

>  **分群思考（THINKING IN SUB-GROUPS）**
>
>  若我们采用的编程模型需考虑<font style="background-color:#E8F7CF;">显式向量化处理</font>，可将每个子组（sub-group）视为打包进SIMD寄存器的工作项集合，其中子组内的每个工作项对应一个SIMD通道。当多个子组并行执行且设备确保其持续推进时，此思维模型可进一步延伸——将每个子组视作并行执行的独立向量指令流。  
>

### 编写显式ND范围数据并行内核（Writing Explicit ND-Range Data-Parallel Kernels）
```cpp
range global{N, N};  
range local{B, B};  
h.parallel_for(nd_range{global, local}, [=](nd_item<2> it) {  
    int j = it.get_global_id(0);  
    int i = it.get_global_id(1);  
    for (int k = 0; k < N; ++k) {  
        c[j][i] += a[j][k] * b[k][i];  
    }  
}); 
```

图4-15采用ND-range并行内核语法重新实现了我们先前介绍的矩阵乘法内核，图4-16中的示意图则展示了该内核的工作量如何映射到各工作组内的工作项。通过这种方式<font style="background-color:#E8F7CF;">对工作项进行分组，不仅能确保数据访问的局部性，还有望提高缓存命中率</font>：例如图4-16中的工作组采用(4,4)的局部范围，虽然包含16个工作项，但仅需访问单工作项四倍的数据量——这意味着我们从内存加载的每个数值均可重复使用四次。  

![图4-16. 将矩阵乘法映射到工作组与工作项](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744270249506-113350bd-f894-4f99-a785-690a1ab9ef6e.png)

迄今为止，我们的矩阵乘法示例依赖于硬件缓存来优化同一工作组内工作项对矩阵A和B的重复访问。此类硬件缓存在传统CPU架构中十分普遍，且在GPU架构中也日益普及，但部分架构具备显式管理的"暂存器"存储器，可提供更高性能（例如通过更低延迟）。<font style="background-color:#E8F7CF;">ND范围内核可使用局部访问器来描述应分配至工作组局部内存的数据，实现层则可自由将这些分配映射至专用内存（若存在该硬件）</font>。工作组局部内存的具体用法将在第9章详述。  

###  显式ND范围数据并行内核的细节（Details of Explicit ND-Range Data-Parallel Kernels）
ND-range数据并行内核所使用的类与基础数据并行内核不同：`range`被替换为`nd_range`，`item`被替换为`nd_item`。此外还引入了两个新类，用于表示工作项可能归属的不同工作组类型：<font style="color:#DF2A3F;">与工作组相关的功能封装在</font>`<font style="color:#DF2A3F;">group</font>`<font style="color:#DF2A3F;">类中，与子组相关的功能则封装在</font>`<font style="color:#DF2A3F;">sub_group</font>`<font style="color:#DF2A3F;">类中</font>。  

#### `nd_range` 类（The nd_range Class）
`<font style="color:#DF2A3F;background-color:#FBF5CB;">nd_range</font>`<font style="color:#DF2A3F;background-color:#FBF5CB;">表示使用两个</font>`<font style="color:#DF2A3F;background-color:#FBF5CB;">range</font>`<font style="color:#DF2A3F;background-color:#FBF5CB;">类实例的分组执行范围</font>：<font style="background-color:#E8F7CF;">一个表示全局执行范围，另一个表示每个工作组（</font>`<font style="background-color:#E8F7CF;">work-group</font>`<font style="background-color:#E8F7CF;">）的本地执行范围</font>。图4-17给出了nd_range类的简化定义。  

或许令人略感意外的是，nd_range类完全没有提及子组：<font style="background-color:#E8F7CF;">子组范围既未在构造时指定，也无法被查询</font>。这一省略出于两个原因：首先，子组作为底层实现细节，在许多内核中可被忽略；其次，部分设备仅支持单一有效子组规模，若处处指定该规模将导致不必要的冗长表述。所有与子组相关的功能均封装于后续将讨论的专用类中。  

```cpp
template <int Dimensions = 1>
class nd_range {
public:
    // 通过全局范围和局部工作组范围构造nd_range
    nd_range(range<Dimensions> global, range<Dimensions> local);

    // 返回全局范围和局部工作组范围
    range<Dimensions> get_global_range() const;
    range<Dimensions> get_local_range() const;

    // 返回全局范围中的工作组数量
    range<Dimensions> get_group_range() const;
};
```

#### `nd_item` 类（The nd_item Class）
`nd_item`是ND范围（ND-range）形式的`item`，它同样封装了内核的执行范围以及该范围内`item`的索引。`nd_item`与`item`的区别在于其位置信息的查询与表示方式，如图4-18简化类定义所示。例如，我们可以通过`get_global_id()`函数查询该`item`在（全局）ND范围中的索引，或通过`get_local_id()`函数查询该`item`在其（局部）父工作组中的索引。  

`nd_item`类还提供了用于获取描述项所属工作组和子组的类句柄的函数。这些类提供了查询ND-range中项索引的替代接口。  

```cpp
template <int Dimensions = 1> class nd_item {  
public:  
    // 返回当前工作项在内核执行范围中的索引  
    id<Dimensions> get_global_id() const;  
    size_t get_global_id(int dimension) const;  
    size_t get_global_linear_id() const;  

    // 返回当前工作项所在内核的执行范围  
    range<Dimensions> get_global_range() const;  
    size_t get_global_range(int dimension) const;  

    // 返回当前工作项在其父工作组内的索引  
    id<Dimensions> get_local_id() const;  
    size_t get_local_id(int dimension) const;  
    size_t get_local_linear_id() const;  

    // 返回当前工作项所在父工作组的执行范围  
    range<Dimensions> get_local_range() const;  
    size_t get_local_range(int dimension) const;  

    // 返回包含当前工作项的工作组或子组的句柄  
    group<Dimensions> get_group() const;  
    sub_group get_sub_group() const;  
};
```

#### `group`类（The group Class）
群组类封装了与工作组相关的所有功能，其简化定义如图4-19所示。  

```cpp
template <int Dimensions = 1> class group {  
public:  
    // 返回该工作组在内核执行范围中的索引  
    id<Dimensions> get_id() const;  
    size_t get_id(int dimension) const;  
    size_t get_linear_id() const;  

    // 返回内核执行范围内的工作组数量  
    range<Dimensions> get_group_range() const;  
    size_t get_group_range(int dimension) const;  

    // 返回该组内工作项的数量  
    range<Dimensions> get_local_range() const;  
    size_t get_local_range(int dimension) const;  
};
```

组类(group class)提供的许多功能在`nd_item`类中都有对应实现：例如调用`group.get_group_id()`等价于调用`item.get_group_id()`，调用`group.get_local_range()`等价于调用`item.get_local_range()`。

如果我们不使用任何组函数或算法，是否仍需使用组类？

直接调用`nd_item`中的函数而非创建中间组对象岂不更简单？这里存在权衡取舍：<font style="color:#DF2A3F;">使用组类需要编写稍多的代码，但代码可读性可能更佳</font>。以图4-20所示代码片段为例：可以明确看出body函数需要被组内所有工作项调用，且`parallel_for`循环体中`get_local_range()`返回的范围显然是组的范围。这段代码完全可以仅用`nd_item`实现，但代码的阅读流畅性可能会降低。  

```cpp
void body(group& g);  
h.parallel_for(nd_range{global, local}, [=](nd_item<1> it) {  
    group<1> g = it.get_group();  
    range<1> r = g.get_local_range();  
    ...  
    body(g);  
});
```

群体类(group class)启用的另一强大功能，是通过模板参数编写通用群体函数的能力，这类函数可接受任意类型的群体。尽管SYCL尚未(在C++20意义上)正式定义Group"概念"(concept)，但`group`和`sub_group`类公开了通用接口，允许使用`sycl::is_group_v`等特征来约束模板化的SYCL函数。目前，这种<font style="color:#DF2A3F;background-color:#FBF5CB;">通用编码形式的主要优势在于</font>：<font style="background-color:#E8F7CF;">既能支持任意维度的工作组(work-group)，又能允许函数调用者决定应在工作组内的工作项(work-items)间还是子组(sub-group)的工作项间分配任务</font>。SYCL的群体接口设计具有可扩展性，我们预期未来版本中将出现更多代表不同工作项分组形式的类。

#### sub_group类（The sub_group Class）  
<font style="color:#DF2A3F;background-color:#FBF5CB;">子组（sub_group）类封装了与子组相关的所有功能</font>，其简化定义如图4-21所示。与工作组（work-group）不同，<font style="background-color:#E8F7CF;">子组类是访问子组功能的唯一途径——其所有功能均未在nd_item类中重复实现</font>。  

```cpp
class sub_group {
public:
    // 返回子组的索引
    id<1> get_group_id() const;

    // 返回当前工作项所属父工作组中子组的数量
    range<1> get_group_range() const;

    // 返回当前工作项在子组中的局部索引
    id<1> get_local_id() const;

    // 返回当前子组中的工作项数量
    range<1> get_local_range() const;

    // 返回当前工作项所属父工作组中任意子组的最大工作项容量
    range<1> get_max_local_range() const;
};
```

需注意，<font style="background-color:#E8F7CF;">查询当前子组中的工作项数量和查询工作组内任意子组中最大工作项数量分别由不同函数实现</font>。这些数值是否存在差异以及差异程度，取决于特定设备上子组的具体实现方式，但其设计意图是反映编译器设定的目标子组大小与运行时实际子组大小之间的潜在区别。例如：极小的工作组可能包含的工作项数量少于编译时子组大小；或采用不同规模的子组来处理无法被子组大小整除的工作组及维度。  

##  将计算映射至工作项（Mapping Computation to Work-Items）
目前展示的大部分代码示例都<font style="background-color:#E8F7CF;">默认一个核函数实例对应于单次数据上的单一操作</font>。这种编写核函数的方式虽直观，但SYCL或任何内核形式均未强制要求此类一对一映射——我们始终完全掌控着将数据(及计算任务)分配给独立工作项的过程，而将<font style="background-color:#E8F7CF;">这种分配方式参数化往往能有效提升性能可移植性</font>。  

###  一对一映射（One-to-One Mapping）
当我们编写的内核实现了工作任务与工作项之间的一对一映射时，这些内核必须始终以与待处理工作量完全匹配的范围（range）或nd_range大小启动。这是编写内核最直观的方式，且在多数情况下能高效运行——我们可以相信实现方案会将工作项合理地映射到硬件资源上。  

然而，<font style="color:#DF2A3F;background-color:#FBF5CB;">当针对特定系统与实现组合进行性能调优时，可能需要更密切地关注底层调度行为</font>。工作组对计算资源的调度方式由具体实现定义，且可能采用动态机制（即当某个计算资源完成一个工作组后，其执行的下一个工作组可能来自共享队列）。动态调度对性能的影响并非固定，其重要性取决于多种因素，包括内核函数每次实例的执行时长以及调度是在软件层面（如CPU）还是硬件层面（如GPU）实现。  

### 多对一映射（Many-to-One Mapping）
另一种方法是编写具有工作项与工作任务多对一映射关系的内核。此时，范围的含义会发生微妙变化：该范围不再描述待处理的工作总量，而是指代所使用的并行工作单元数量。通过调整工作单元数量及每个单元分配的任务量，我们可以精细化调控任务分发机制以实现性能最优化。  

编写这种形式的内核需要进行两处修改：  

1. 内核必须接受一个描述工作总量的参数。 
2. 内核必须包含一个向工作项分配任务的循环。  

图4-22给出了此类内核的一个简单示例。需注意的是，内核内部的循环结构略显特殊——其起始索引为工作项在全局范围内的编号，而步长则为工作项总数。这种数据到工作项的轮询调度方式既确保了循环的所有N次迭代都将由某个工作项执行，又使相邻工作项能访问连续的内存地址（从而提升缓存局部性与向量化性能）。类似地，工作也可分配到工作组之间或单个工作组内的工作项之间，以进一步增强局部性。  

```cpp
size_t N = ...; // 工作量  
size_t W = ...; // 工作线程数  
h.parallel_for(range{W}, [=](item<1> it) {  
    for (int i = it.get_id()[0]; i < N; i += it.get_range()[0]) {  
        output[i] = function(input[i]);  
    }  
});  
```

这些工作分配模式相当常见，我们预计未来版本的SYCL将引入语法糖来简化ND-range内核中工作分配的表达。

## 选择核函数形式
在不同内核形式之间的选择主要取决于个人偏好，并深受以往使用其他并行编程模型和语言的经验影响。  

选择特定内核形式的另一个主要原因是，只有这种形式才能提供内核所需的某些功能。遗憾的是，在开发开始前往往难以确定需要哪些功能——尤其当我们对不同内核形式及其与各类别的交互尚不熟悉时。  

我们根据自身经验构建了两份指南，以帮助驾驭这一复杂领域。<font style="color:#DF2A3F;">这些指南应视为初步建议</font>，绝非旨在取代实际探索——在不同内核形态间做出选择的最佳方式，始终是花时间逐一实践书写，从而判断哪种形态最适合我们的应用场景与开发风格。  

第一份指南是图4-23中的流程图，它根据以下方式选择内核形式。

1. 我们是否具备并行编程的先前经验
2. 我们是从零开始编写新代码，还是移植用其他语言编写的现有并行程序
3. 我们的内核是高度并行（embarrassingly parallel）的，还是会在内核函数的不同实例间复用数据
4. 无论我们是使用SYCL编写新内核以追求极致性能、提升代码可移植性，还是因为它比底层语言能更高效地表达并行计算  

![图4-23. 帮助选择合适的内核形态](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744271742087-eeb5f713-0c00-4489-8ee2-a967190e9be0.png)

第二条指导原则涉及内核形式所公开的功能集。工作组、子组、组屏障、组本地内存、组函数（如广播）及组算法（如扫描、归约）仅适用于ND-range内核。因此，在我们需要表达复杂算法或进行性能微调的场景中，应优先选用ND-range内核。  

每个内核形式可用的特性会随着语言发展而变化，但基本趋势预计将保持不变：基础数据并行内核不会暴露局部性感知特性，而显式ND范围内核将开放所有支持性能优化的功能。  

##  总结（Summary）
本章介绍了使用SYCL在C++中表达并行性的基础知识，并讨论了编写数据并行内核时每种方法的优缺点。  

SYCL支持多种形式的并行计算，我们希望提供的这些信息能帮助读者做好准备，深入探索并开始编写代码！  

我们目前仅触及皮毛，后续章节将深入探讨本章介绍的诸多概念与类：第9章将详述本地内存的使用、屏障及通信例程；第10章和第20章将讨论除lambda表达式外定义内核的其他方法；第15、16和17章将探究ND-range执行模型在具体硬件上的详细映射；而第14章则会介绍使用SYCL表达常见并行模式的最佳实践。  