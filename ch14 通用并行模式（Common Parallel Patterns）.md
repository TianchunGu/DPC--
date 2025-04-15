当我们作为程序员处于最佳状态时，能够识别工作中的模式，并运用那些经过时间验证的最佳解决方案。并行编程亦是如此，若不去研究那些已被证明行之有效的模式，将铸成大错。以大数据应用中采用的MapReduce框架为例：其成功很大程度上源于基于两个简单却高效的并行模式——map（映射）与reduce（归约）。  

<font style="color:#DF2A3F;">在并行编程中，存在若干反复出现的通用模式</font>，<font style="color:#DF2A3F;">这些模式与我们使用的编程语言无关</font>。它们<font style="color:#DF2A3F;">具有高度通用性，可应用于任何并行化层级（如子组、工作组、完整设备）和任何计算设备（如CPU、GPU、FPGA）</font>。然而，<font style="color:#601BDE;">这些模式的特质（例如可扩展性）会影响其在不同设备上的适用性</font>。<font style="background-color:#E8F7CF;">某些情况下，使应用程序适配新设备仅需调整参数或优化现有模式的实现；而在其他场景中，完全更换并行模式可能会显著提升性能</font>。  

掌握这些常见并行模式的使用方法、时机和场景，是提升我们在SYCL（乃至广义并行编程）中熟练度的关键。对于已有并行编程经验者通过观察这些模式在SYCL中的表达方式，能够快速上手并熟悉该语言的功能特性。  

 本章旨在回答以下问题： 

+ 我们需要理解哪些常见模式？ 
+ 这些模式如何与不同设备的性能关联？ 
+ 哪些模式已作为SYCL函数或库提供？ 
+ 如何通过直接编程实现这些模式？  

## 理解模式（Understanding thePatterns）
此处讨论的模式是McCool等人在《结构化并行编程》（Structured Parallel Programming）一书中描述的并行模式的子集。我们未涉及与并行类型相关的模式（如fork-join、分支限界等），而<font style="color:#DF2A3F;background-color:#FBF5CB;">专注于对编写数据并行内核最有价值的部分算法模式</font>。  

我们坚信，<font style="color:#DF2A3F;">理解这一并行模式子集对于成为高效的SYCL程序员至关重要</font>。图14-1中的表格从高层次概述了不同模式的主要应用场景、关键特性，以及这些特性如何影响它们对不同硬件设备的适配性。  

![图14-1. 并行模式及其对不同设备类型的适配性](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744641927403-6f2f6596-fc25-4ed6-8e41-7aaace393a85.png)

### 映射（Map）
映射模式（The map pattern）是所有并行模式中最简单的一种，对于有函数式编程语言经验的读者来说会立即感到熟悉。如图14-2所示，<font style="color:#DF2A3F;">通过应用某个函数，将范围内的每个输入元素独立地映射为一个输出</font>。许多数据并行操作都可以表示为映射模式的实例（例如向量加法）。  

![图14-2. 映射模式](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744641997581-5194ed87-dfbb-4b3f-8e88-d433c46433f5.png)

由于函数的每次应用都是完全独立的，映射表达式通常非常简单，主要依赖编译器和/或运行时系统来完成大部分繁重工作。<font style="color:#2F4BDA;">按照映射模式编写的内核应适用于任何设备，且其性能会随着可用硬件并行度的增加而实现良好扩展</font>。  

然而，在决定将整个应用程序重写为一系列映射核之前，我们必须审慎思考！这种开发方式虽然能高效产出成果，并确保应用程序可移植到多种设备类型，但也<font style="color:#DF2A3F;">会诱使我们忽视那些可能显著提升性能的优化策略</font>（例如改进数据复用、融合计算核）。  

### 模板（Stencil）
模板模式（The stencil pattern）与映射模式密切相关。如图14-3所示，该模式<font style="color:#DF2A3F;">通过对输入数据及模板定义的相邻数据集施加特定函数运算，最终生成单一输出结果</font>。模板模式常见于众多领域，包括科学/工程应用（如有限差分算法）与计算机视觉/机器学习应用（如图像卷积运算）。

  ![图14-3. 模板模式](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744642145522-c95f34f1-cfb7-4a9a-8b8d-f3287f9e0aa1.png)

当模板模式以非原位方式执行时（即，将输出写入独立的存储位置），该函数可独立应用于每个输入。然而，现实场景中的模板调度往往更为复杂：相邻输出的计算需要相同数据，而多次从内存加载这些数据会降低性能；此外，为减少应用程序的内存占用，我们可能希望采用原位方式应用模板（即覆盖原始输入值）。  

因此，<font style="color:#DF2A3F;background-color:#FBF5CB;">模板核（stencil kernel）对不同设备的适用性高度依赖于模板特性及输入问题的性质</font>。一般而言，  

+ <font style="color:#DF2A3F;background-color:#FBF5CB;">小型模板（stencil）计算</font><font style="background-color:#E8F7CF;">可受益于GPU的暂存存储器（scratchpad storage）</font>。   
+ <font style="color:#DF2A3F;background-color:#FBF5CB;">大型模板计算</font><font style="background-color:#E8F7CF;">可受益于CPU（相对）较大的缓存</font>。 
+ <font style="color:#DF2A3F;background-color:#FBF5CB;">针对小规模输入的小型模板计算</font><font style="background-color:#E8F7CF;">通过FPGA上的脉动阵列实现，可获得显著的性能提升</font>。  

由于<font style="color:#DF2A3F;">模板易于描述但难以高效实现</font>，许多模板应用会采用领域特定语言（DSL）。目前已有若干嵌入式DSL利用C++的模板元编程能力，在编译时生成高性能模板计算核心。  

### 归约（Reduction）
归约（reduction）是一种常见的并行模式，它<font style="color:#DF2A3F;background-color:#FBF5CB;">通过一个通常具有结合律和交换律的运算符（例如加法）来合并部分计算结果</font>。最普遍存在的归约实例包括求和运算（例如计算点积时）或计算最小值/最大值（例如利用最大速度来设定时间步长）。  

图14-4展示了通过树形归约（tree reduction）实现的规约模式，这是一种常见实现方式，对N个输入元素的范围进行归约需要log2(N)次合并操作。尽管树形归约较为普遍，但也可能存在其他实现方式——总体而言，<font style="color:#DF2A3F;">我们不应假定规约操作会以特定顺序合并数值</font>。  

![图14-4. 归约模式](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744642330024-4c50c556-fe94-4065-ba39-500211766082.png)

在实际应用中，计算内核很少能达到完全并行化的理想状态。即便存在这种可能，也常需与归约操作（如MapReduce框架中的实现）结合使用以汇总结果。这使归约成为最关键的并行计算模式之一——<font style="color:#DF2A3F;">我们必须在所有计算设备上高效执行该操作，深入理解其机制至关重要</font>。  

为不同设备调整归约操作是一项微妙的平衡艺术，需在计算部分结果的时间与合并它们的时间之间权衡：并行度不足会延长计算时间，而过度并行则会增加合并耗时。  

为了提高整体系统利用率，人们可能会倾向于采用不同设备分别执行计算和组合步骤，但这种优化措施必须谨慎考虑设备间数据传输的开销。实际上，我们发现<font style="color:#DF2A3F;">直接在数据生成时、于同一设备上执行归约操作通常是最佳方案</font>。因此，<font style="background-color:#FBF5CB;">要利用多设备提升归约模式的性能，不能依赖任务并行机制，而需通过另一层级的数据并行实现（即每个设备对部分输入数据执行归约操作）</font>。  

### 扫描（Scan）
扫描模式通过二元关联运算符计算广义前缀和，输出数组的每个元素代表一个部分结果。若元素i的部分和包含区间[0, i]内所有元素（即包含i的求和），则称为<font style="color:#DF2A3F;background-color:#FBF5CB;">包容性扫描</font>；若元素i的部分和仅包含区间[0, i)内元素（即排除i的求和），则称为<font style="color:#DF2A3F;background-color:#FBF5CB;">排他性扫描</font>。  

乍看之下，扫描操作似乎<font style="color:#DF2A3F;">本质上是串行的——每个输出值都依赖于前一个输出值的结果</font>！尽管扫描操作确实比其他模式具有更少的并行化机会（因此可扩展性可能较低），但图14-5表明，<font style="background-color:#E8F7CF;">通过对同一数据进行多次扫描，仍然可以实现并行扫描算法</font>。 

![图14-5. 扫描模式](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744642425870-e180921b-dda3-4c9e-b728-f187ea1112ea.png)

由于<font style="color:#DF2A3F;">扫描操作内部的并行机会有限，执行扫描任务的最佳硬件设备高度依赖于问题规模</font>：<font style="background-color:#FBF5CB;">较小规模的问题更适合CPU处理</font>，因为只有<font style="background-color:#FBF5CB;">较大规模问题才具备足够的数据并行性以充分利用GPU</font>。<font style="background-color:#FBF5CB;">对于FPGA和其他空间架构而言，问题规模的影响较小，因为扫描操作天然适合流水线并行</font>。与归约操作类似，通常建议在生成数据的同一设备上执行扫描操作——在优化过程中统筹考虑扫描操作在应用中的部署位置与方式，往往比孤立地优化扫描操作能获得更好的效果。  

###  打包与解包（Pack andUnpack）  
<font style="color:#DF2A3F;">打包和解包模式与扫描操作密切相关，通常基于扫描功能实现</font>。我们在此单独讨论它们，是因为这些模式能够高效实现某些常见操作（例如向列表追加元素），而这些操作与前缀和之间的关联可能并不直观。  

#### 打包（Pack）
图14-6所示的<font style="color:#DF2A3F;background-color:#FBF5CB;">打包模式（pack pattern）</font><font style="background-color:#E8F7CF;">根据布尔条件舍弃输入范围内的元素，将未被舍弃的元素连续排列到输出范围中</font>。该布尔条件可以是预先计算的掩码，也可以通过应用某个函数到每个输入元素实时计算得出。  

![图14-6. 包装模式](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744642518071-b0392cf8-102e-4750-b02a-b0a30594edaa.png)

与扫描操作类似，<font style="color:#DF2A3F;">打包操作本质上具有串行特性</font>。给定一个待打包/复制的输入元素时，计算其在输出范围内的位置需要获知之前有多少元素也被打包/复制至输出端。该信息等价于对驱动打包操作的布尔条件进行独占式扫描的结果。  

#### 解包（Unpack）
如图14-7所示（正如其名称所示），<font style="color:#DF2A3F;">解包模式（unpack pattern）与打包模式（pack pattern）互为逆向操作</font>。该模式将输入范围的连续元素解包至输出范围的非连续位置，同时保持其他元素不变。该模式最典型的应用场景是解压先前打包的数据，但也可用于填补因某些前期计算导致的数据"间隙"。  

![图14-7. 解包模式](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744642578489-a7c2e683-f95a-4d0a-9164-7e275f0a91ba.png)

##  使用内置函数与库（Using Built-In Functions and Libraries）
许多这类<font style="color:#DF2A3F;">模式可直接利用SYCL的内置功能或供应商提供的基于SYCL编写的库来实现</font>。<font style="background-color:#E8F7CF;">在大型实际软件工程项目中，充分发挥这些函数与库的作用，是实现性能、可移植性与开发效率最佳平衡的有效途径</font>。  

###  SYCL归约库（The SYCL Reduction Library）  
相较于要求每个开发者自行维护可移植且高性能的归约核函数库，SYCL提供了一种便捷的抽象机制，用于描述具有归约语义的变量。这种抽象既简化了归约核函数的表达方式，又显式声明了正在执行的归约操作，使得实现层能够根据设备类型、数据类型和归约操作的组合，灵活选择不同的归约算法。  

图14-8所示内核展示了使用归约库的范例。需注意的是，该内核主体并未包含任何显式归约操作——我们仅需声明内核包含一个归约过程，该过程通过plus仿函数对sum变量实例进行组合。这种声明方式为系统实现自动生成优化归约序列提供了充分依据。  

```cpp
h.parallel_for( 
    range<1>{N}, reduction(sum, plus<>()), 
    [=](id<1> i, auto& sum) { sum += data[i]; });  
```

归约操作（reduction）的结果不保证会被写回原变量，直到内核执行完成。除这一限制外，访问缩减结果的行为与访问SYCL中其他变量完全一致：访问存储在缓冲区中的缩减结果需要创建相应的设备或主机访问器，而访问存储在统一共享内存(USM)分配中的缩减结果则可能需要显式同步和/或内存迁移操作。  

SYCL归约库与其他语言的归约抽象机制存在一项重要差异：它限制了我们在内核执行期间对归约变量的访问——既无法查看归约变量的中间值，也不允许使用指定组合函数之外的方式更新归约变量。这些限制既避免了难以调试的错误（例如在计算最大值时误操作归约变量进行累加），又确保了归约操作能在各类异构设备上高效执行。  

#### 归约类（The reduction Class）
归约类（reduction class）是我们用于描述内核中归约操作的接口。构造归约对象的唯一方式是通过图14-9所示的函数之一。需注意归约函数分为三大类（针对缓冲区、USM指针和跨度），每类包含两个重载版本（含或不含恒等变量）。  

```cpp
template <typename BufferT, typename BinaryOperation> 
unspecified reduction(BufferT variable, handler& h, 
                        BinaryOperation combiner, 
                        const property_list& properties = {});  
template <typename BufferT, typename BinaryOperation> 
unspecified reduction(BufferT variable, handler& h, 
                        const BufferT::value_type& identity, 
                        BinaryOperation combiner, 
                        const property_list& properties = {});  
template <typename T, typename BinaryOperation> 
unspecified reduction(T* variable, BinaryOperation combiner, 
                        const property_list& properties = {});  
template <typename T, typename BinaryOperation> 
unspecified reduction(T* variable, const T& identity, 
                        BinaryOperation combiner, 
                        const property_list& properties = {});  
template <typename T, typename Extent, 
          typename BinaryOperation> 
unspecified reduction(span<T, Extent> variables, 
                        BinaryOperation combiner, 
                        const property_list& properties = {});  
template <typename T, typename Extent, 
          typename BinaryOperation> 
unspecified reduction(span<T, Extent> variables, 
                        const T& identity, 
                        BinaryOperation combiner, 
                        const property_list& properties = {});  
```

若归约操作通过缓冲区或USM指针初始化，则该操作为标量归约，作用于数组的首元素；若通过span初始化，则执行数组归约。数组归约的各分量相互独立——可将作用于大小为N的数组归约视为N个具有相同数据类型与运算符的标量归约之集合。  

该函数最简单的重载形式允许我们指定归约变量及用于组合各工作项贡献的运算符。第二组重载形式则允许提供与归约运算符关联的可选恒等值——这是针对用户自定义归约的优化措施，我们将在后文详细讨论。  

请注意，归约函数的返回类型未作规定，且归约类本身完全由实现定义。虽然这对C++类而言可能略显非典型，但该设计允许实现使用不同的类（或通过任意数量模板参数定义的单个类）来表征不同的归约算法。未来版本的SYCL可能会重新审视此设计，以便支持我们在特定执行上下文中显式请求特定归约算法（最有可能通过property_list参数实现）。  

####  减速器类（The reducer Class）
reducer类的一个实例封装了归约变量，它通过暴露有限的接口确保我们无法以任何可能被实现视为不安全的方式更新该变量。图14-10展示了reducer类的简化定义。与reduction类类似，reducer类的精确定义由具体实现决定——其类型取决于归约的执行方式，为了最大化性能，必须在编译时明确这一点。不过，用于更新归约变量的函数和运算符均有明确定义，并保证所有SYCL实现都支持这些操作。  

```cpp
template <typename T, typename BinaryOperation, /* 实现定义 */>  
class reducer {  
    // 将部分结果与reducer的值合并  
    void combine(const T& partial);  
};  

// 其他运算符适用于标准二元操作  
template <typename T>  
auto& operator+=(reducer<T, std::plus<T>>&, const T&);  
```

具体而言，每个归约器都提供一个combine()函数，该函数将来自单个工作项的局部结果与归约变量的值进行合并。此combine函数的具体行为由实现定义，但在编写内核时无需关注。根据归约运算符的不同，归约器还需提供其他运算符；例如，加法归约中定义了+=运算符。这些附加运算符仅为提升编程便捷性和代码可读性而设，其功能与直接调用combine()完全等效。  

在处理数组归约时，归约器提供了一个额外的下标运算符（即operator[]），允许访问数组中的单个元素。该运算符并非直接返回数组元素的引用，而是返回另一个归约器对象——该对象同样暴露了与标量归约相关联的combine()函数及简写运算符。图14-11展示了一个使用数组归约计算直方图的内核简单示例，其中下标运算符仅用于访问工作项所更新的直方图区间。  

```cpp
h.parallel_for( 
    range{N}, 
    reduction(span<int, 16>(histogram, 16), plus<>()), 
    [=](id<1> i, auto& histogram) { 
        histogram[i % B]++; 
    });
```

####  用户自定义归约（User-Defined Reductions）
几种常见的归约算法（例如树形归约）并非让每个工作项直接更新单个共享变量，而是先在私有变量中累积部分结果，待后续阶段进行合并。这类私有变量会引发一个问题：实现代码应如何初始化它们？若将变量初始化为每个工作项的首个贡献值，可能带来性能影响，因为需要额外逻辑来检测和处理未初始化的变量。若将变量初始化为归约运算符的幺元值，则可避免性能损耗，但此方法仅适用于已知幺元值的情形。  

SYCL实现只能自动确定适用于简单算术类型的归约操作及标准函数对象（如plus）作为归约运算符时的正确单位元值。对于用户自定义的归约操作（即作用于用户自定义类型和/或使用用户自定义函数对象的归约），我们通过直接指定单位元值可能获得性能提升。  

对用户自定义归约操作的支持仅限于可平凡复制类型和无副作用的组合函数，但这已足以满足众多实际应用场景的需求。例如，图14-12中的代码展示了通过用户自定义归约操作同时计算向量中最小元素及其位置的用法。  

```cpp
template <typename T, typename I> using minloc = minimum<std::pair<T, I>>;  
int main() {  
    constexpr size_t N = 16;  
    queue q;  
    float* data = malloc_shared<float>(N, q);  
    std::pair<float, int>* res = malloc_shared<std::pair<float, int>>(1, q);  
    std::generate(data, data + N, std::mt19937{});  

    std::pair<float, int> identity = {  
        std::numeric_limits<float>::max(),  
        std::numeric_limits<int>::min()  
    };  
    *res = identity;  

    auto red = sycl::reduction(res, identity, minloc<float, int>());  

    q.submit([&](handler& h) {  
        h.parallel_for(  
            range<1>{N},  
            red,  
            [=](id<1> i, auto& res) {  
                std::pair<float, int> partial = {data[i], i};  
                res.combine(partial);  
            }  
        );  
    }).wait();  

    std::cout << "minimum value = " << res->first << " at " << res->second << "\n";  
    ...  
}
```

###  群组算法（Group Algorithms）
SYCL设备代码中对并行模式的支持由一个独立的组算法库提供。这些函数利用特定工作组（即工作组或子组）的并行性，在有限范围内实现常见的并行算法，并可作为构建模块用于构造其他更复杂的算法。  

SYCL中群组算法的语法基于C++标准库中的算法库，且遵循C++算法的所有限制条件。然而存在一个关键区别：STL算法是从串行（主机）代码调用的，仅暗示库可能采用并行化；而SYCL的群组算法专为在已并行执行的（设备）代码中调用而设计。为确保该差异不被忽视，这些群组算法在语法和语义上都与其C++对应物略有不同。  

SYCL区分两种不同类型的并行算法。若某算法由工作组内所有工作项协作执行，但其他行为与STL算法完全相同，则该算法名称冠以"joint"前缀（因为组成员需"联合"执行该算法）。此类算法从内存读取输入数据并将结果写入内存，且仅能操作给定工作组内所有工作项可见的内存数据。若某算法作用于反映工作组本身的隐式范围，且输入输出存储于工作项私有内存中，则其名称将包含"group"字样（因为该算法直接操作工作组持有的数据）。  

图14-13中的代码示例展示了这两种不同类型的算法，将std::reduce的行为与sycl::joint_reduce和sycl::reduce_over_group的行为进行了对比。  

```cpp
// std::reduce 
// Each work-item reduces over a given input range  
q.parallel_for(number_of_reductions, [=](size_t i) {  
    output1[i] = std::reduce(  
        input + i * elements_per_reduction,  
        input + (i + 1) * elements_per_reduction  
    );  
}).wait();  

// sycl::joint_reduce 
// Each work-group reduces over a given input range  
// The elements are automatically distributed over  
// work-items in the group  
q.parallel_for(nd_range<1>{number_of_reductions * elements_per_reduction, elements_per_reduction}, [=](nd_item<1> it) {  
    auto g = it.get_group();  
    int sum = joint_reduce(  
        g,  
        input + g.get_group_id() * elements_per_reduction,  
        input + (g.get_group_id() + 1) * elements_per_reduction,  
        plus<>()  
    );  
    if (g.leader()) {  
        output2[g.get_group_id()] = sum;  
    }  
}).wait();  

// sycl::reduce_over_group 
// Each work-group reduces over data held in work-item  
// private memory. Each work-item is responsible for  
// loading and contributing one value  
q.parallel_for(  
    nd_range<1>{number_of_reductions * elements_per_reduction, elements_per_reduction},  
    [=](nd_item<1> it) {  
        auto g = it.get_group();  
        int x = input[g.get_group_id() * elements_per_reduction + g.get_local_id()];  
        int sum = reduce_over_group(g, x, plus<>());  
        if (g.leader()) {  
            output3[g.get_group_id()] = sum;  
        }  
    }  
).wait();
```

请注意，在这两种情况下，每个分组算法的第一个参数均接受一个group或sub_group对象（而非执行策略）来描述应参与算法执行的工作项集合。由于算法需由指定组内所有工作项协作完成，其行为应视为类似组屏障——组内所有工作项必须在收敛控制流中遇到相同的算法调用（即全组工作项必须统一执行或跳过该算法），且所有工作项提供的参数必须确保全组对执行的操作达成共识。例如，sycl::joint_reduce要求所有工作项的输入参数必须完全一致，以确保全组工作项处理相同数据并使用相同运算符累积结果。  

图14-14中的表格展示了STL提供的并行算法与分组算法的对应关系，以及可使用的分组类型是否存在限制。需要注意的是，某些情况下分组算法仅能用于子分组；这些情况对应前文介绍的"混洗"操作。  

![图14-14. C++算法与SYCL组算法的映射关系](https://cdn.nlark.com/yuque/0/2025/png/33636091/1744643591969-78d1829a-d63b-4118-90f1-23be469716be.png)

在撰写本文时，组算法仅支持原始数据类型及SYCL识别的一组内置运算符（如加、乘、位与、位或、位异或、逻辑与、逻辑或、最小值和最大值）。这足以覆盖大多数常见应用场景，但未来版本的SYCL有望将集合操作支持扩展至用户自定义类型和运算符。  

##  直接编程（Direct Programming）
尽管我们建议尽可能利用现有库函数，但通过研究如何用"原生"SYCL内核实现每种模式，我们仍能获益良多。  

本章其余部分的内核性能虽不及高度优化的库，但对于深入理解SYCL的功能大有裨益——甚至可作为原型化新库功能的开发起点。

>  使用厂商提供的库！  
>
>  当厂商提供某个函数的库实现时，使用该实现几乎总比重写内核级函数更为可取。  
>

### 映射（Map）
由于映射模式的简洁性，其可直接实现为一个基础并行内核。图14-15所示代码展示了这种实现方式——通过映射模式计算指定范围内每个输入元素的平方根。  

```cpp
// Compute the square root of each input value 
q.parallel_for(N, [=](id<1> i) { 
    output[i] = sqrt(input[i]); 
}).wait();
```

###   模板（stencil）
 将模板直接实现为具有多维缓冲区的多维基础数据并行内核（如图14-16所示）既直观又易于理解。  

```cpp
q.submit([&](handler& h) {  
    accessor input{input_buf, h};  
    accessor output{output_buf, h};  
    // 计算每个单元及其直接相邻单元的平均值  
    h.parallel_for(stencil_range, [=](id<2> idx) {  
        int i = idx[0] + 1;  
        int j = idx[1] + 1;  

        float self = input[i][j];  
        float north = input[i - 1][j];  
        float east = input[i][j + 1];  
        float south = input[i + 1][j];  
        float west = input[i][j - 1];  

        output[i][j] = (self + north + east + south + west) / 5.0f;  
    });  
});
```

然而，这种模板模式的表达方式非常初级，其性能表现不应被寄予过高期望。如本章前文所述，业界公认必须通过利用局部性（采用空间或时间分块技术）来避免对内存中相同数据的重复读取。图14-17展示了使用工作组本地内存实现空间分块的简单示例。  

```cpp
q.submit([&](handler& h) {  
    accessor input{input_buf, h};  
    accessor output{output_buf, h};  
    constexpr size_t B = 4;  
    range<2> local_range(B, B);  
    range<2> tile_size = local_range + range<2>(2, 2); // 包含边界单元  
    auto tile = local_accessor<float, 2>(tile_size, h);  

    // 计算每个单元及其直接邻域的平均值  
    h.parallel_for(  
        nd_range<2>(stencil_range, local_range),  
        [=](nd_item<2> it) {  
            // 将当前分块加载到工作组本地内存  
            id<2> lid = it.get_local_id();  
            range<2> lrange = it.get_local_range();  
            for (int ti = lid[0]; ti < B + 2; ti += lrange[0]) {  
                int gi = ti + B * it.get_group(0);  
                for (int tj = lid[1]; tj < B + 2; tj += lrange[1]) {  
                    int gj = tj + B * it.get_group(1);  
                    tile[ti][tj] = input[gi][gj];  
                }  
            }  

            group_barrier(it.get_group());  

            // 使用本地内存中的值计算模板  
            int gi = it.get_global_id(0) + 1;  
            int gj = it.get_global_id(1) + 1;  

            int ti = it.get_local_id(0) + 1;  
            int tj = it.get_local_id(1) + 1;  
            float self = tile[ti][tj];  
            float north = tile[ti - 1][tj];  
            float east = tile[ti][tj + 1];  
            float south = tile[ti + 1][tj];  
            float west = tile[ti][tj - 1];  
            output[gi][gj] = (self + north + east + south + west) / 5.0f;  
    });  
}); 
```

为特定模板选择最佳优化方案，需要编译时内省块大小、邻域范围及模板函数本身，其所需方法的复杂程度远超本文所探讨的内容。  

### 归约（reduction）
通过利用SYCL提供的语言特性（如同步和线程间通信功能，包括原子操作、工作组和子组函数以及子组"混洗"操作），可以实现归约内核。图14-18和图14-19展示了两种可能的归约实现方案：一种是使用基础parallel_for循环配合每个工作项的原子操作实现的简单归约；另一种是分别采用ND-range并行循环和工作组归约函数、通过利用数据局部性实现的优化归约方案。我们将在第19章更详细地重访这些原子操作的讨论。  

```cpp
q.parallel_for(N, [=](id<1> i) {  
    atomic_ref<int, memory_order::relaxed, 
                    memory_scope::system, 
                    access::address_space::global_space>(*sum) += data[i];  
}).wait();  
```

```cpp
q.parallel_for(nd_range<1>{N, B}, [=](nd_item<1> it) {  
    int i = it.get_global_id(0);  
    auto grp = it.get_group();  
    int group_sum = reduce_over_group(grp, data[i], plus<>());  
    if (grp.leader()) {  
        atomic_ref<int, memory_order::relaxed, memory_scope::system, access::address_space::global_space>(  
            *sum) += group_sum;  
    }  
}).wait();  
```

 还存在许多其他编写归约内核的方法，不同设备可能会偏好不同的实现方式，这源于硬件对原子操作的支持差异、工作组局部内存大小、全局内存容量、快速设备级屏障的可用性，甚至专用归约指令的存在。在某些架构上，采用log2(N)次独立内核调用的树形归约甚至可能更快（或成为必要选择！）  

 我们强烈建议，仅当SYCL归约库不支持某些情况，或需要针对特定设备能力微调内核时——即便如此，也务必在100%确认SYCL内置归约性能不足后——才考虑手动实现归约操作！  

### 扫描（scan）
如本章前文所述，实现并行扫描需对数据进行多轮遍历，且每轮遍历间需进行同步。由于SYCL未提供在ND-range范围内同步所有工作项的机制，设备级扫描的直接实现方案必须通过多个内核配合全局内存传递中间结果来完成。  

 代码（如图14-20、14-21和14-22所示）展示了使用多个内核实现的包容性扫描算法。第一个内核将输入值分配到各工作组中，利用工作组本地内存计算工作组局部扫描（注意此处本可采用工作组内置的inclusive_scan函数替代）。第二个内核通过单一工作组对每个块末值进行局部扫描计算。第三个内核整合这些中间结果以完成最终的前缀和运算。这三个内核分别对应图14-5中架构图的三个层级。  

```cpp
// 阶段1：对输入块执行局部扫描计算  
q.submit([&](handler& h) {  
    auto local = local_accessor<int32_t, 1>(L, h);  
    h.parallel_for(nd_range<1>(N, L), [=](nd_item<1> it) {  
        int i = it.get_global_id(0);  
        int li = it.get_local_id(0);  
          
        // 将输入数据复制到局部内存  
        local[li] = input[i];  
        group_barrier(it.get_group());  
          
        // 在局部内存中执行包含性扫描  
        for (int32_t d = 0; d <= log2((float)L) - 1; ++d) {  
            uint32_t stride = (1 << d);  
            int32_t update = (li >= stride) ? local[li - stride] : 0;  
            group_barrier(it.get_group());  
            local[li] += update;  
            group_barrier(it.get_group());  
        }  
          
        // 将每个项的扫描结果写入输出缓冲区  
        // 并将当前块的最后一个结果写入临时缓冲区  
        output[i] = local[li];  
        if (li == it.get_local_range()[0] - 1) {  
            tmp[it.get_group(0)] = local[li];  
        }  
    });  
}).wait();
```

```cpp
// 第二阶段：对部分结果执行扫描计算  
q.submit([&](handler& h) {  
    auto local = local_accessor<int32_t, 1>(G, h);  
    h.parallel_for(nd_range<1>(G, G), [=](nd_item<1> it) {  
        int i = it.get_global_id(0);  
        int li = it.get_local_id(0);  

        // 将输入复制到局部内存  
        local[li] = tmp[i];  
        group_barrier(it.get_group());  

        // 在局部内存中执行包含性扫描  
        for (int32_t d = 0; d <= log2((float)G) - 1; ++d) {  
            uint32_t stride = (1 << d);  
            int32_t update = (li >= stride) ? local[li - stride] : 0;  
            group_barrier(it.get_group());  
            local[li] += update;  
            group_barrier(it.get_group());  
        }  

        // 将每个工作项的结果覆写回临时缓冲区  
        tmp[i] = local[li];  
    });  
}).wait();
```

```cpp
// 第三阶段：利用部分结果更新局部扫描  
q.parallel_for(nd_range<1>(N, L), [=](nd_item<1> it) {  
    int g = it.get_group(0);  
    if (g > 0) {  
        int i = it.get_global_id(0);  
        output[i] += tmp[g - 1];  
    }  
}).wait();
```

 图14-20与图14-21极为相似，唯一的区别在于区间范围的大小以及输入输出值的处理方式。在实际应用中，这种模式可通过单一函数配合不同参数来实现这两个阶段。此处将二者分别呈现，仅为教学演示之目的。  

###  打包与解包(Pack andUnpack)  
 打包和解包操作也被称为聚集（gather）与分散（scatter）操作。这些操作负责处理数据在内存中的实际排列方式与我们希望向计算资源呈现方式之间的差异。  

#### 打包（pack）
 由于pack操作依赖于独占扫描（exclusive scan），要实现适用于ND范围（ND-range）所有元素的pack操作，必须通过全局内存并在多个内核入队过程中完成。然而，pack操作有一种常见应用场景并不要求对整个ND范围的元素执行该操作——即仅对特定工作组（work-group）或子组（sub-group）内的项执行pack操作。  

 图14-23中的代码片段展示了如何在独占扫描基础上实现分组打包操作。  

```cpp
uint32_t index = 
    exclusive_scan(g, (uint32_t)predicate, plus<>()); 
if (predicate) dst[index] = value;
```

<font style="color:rgb(64, 64, 64);">图14-24中的代码演示了如何在核函数中使用此类打包操作，构建需要额外后处理（在后续的内核中）的元素列表。所示示例基于分子动力学模拟的真实内核：分配给粒子i的子组中的工作项通过协作，识别出粒子i固定距离范围内的所有其他粒子，只有此"邻居列表"中的粒子才会被用于计算每个粒子的作用力。</font>

```cpp
range<2> global(N, 8);  
range<2> local(1, 8);  
q.parallel_for(nd_range<2>(global, local), [=](nd_item<2> it) {  
    int i = it.get_global_id(0);  
    sub_group sg = it.get_sub_group();  
    int sglid = sg.get_local_id()[0];  
    int sgrange = sg.get_local_range()[0];  

    uint32_t k = 0;  
    for (int j = sglid; j < N; j += sgrange) {  
        // 计算i与邻居j之间的距离  
        float r = distance(position[i], position[j]);  

        // 将需要后处理的邻居打包到列表中  
        uint32_t pack = (i != j) && (r <= CUTOFF);  
        uint32_t offset = exclusive_scan_over_group(sg, pack, plus<>());  
        if (pack) {  
            neighbors[i * MAX_K + k + offset] = j;  
        }  

        // 记录目前已打包的邻居数量  
        k += reduce_over_group(sg, pack, plus<>());  
    }  

    num_neighbors[i] = reduce_over_group(sg, k, maximum<>());  
}).wait();
```

<font style="color:rgb(64, 64, 64);">需注意，打包模式从不重新排列元素——被打包至输出数组中的元素顺序与其在输入数组中的顺序完全一致。这一特性使得我们能够利用打包功能实现其他更高层次的并行算法（例如 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">std::copy_if</font>**`<font style="color:rgb(64, 64, 64);"> 和 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">std::stable_partition</font>**`<font style="color:rgb(64, 64, 64);">）。然而，也存在一些基于打包功能实现的并行算法无需维持元素顺序（例如 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">std::partition</font>**`<font style="color:rgb(64, 64, 64);">）。</font>

#### 解包（unpack）
与pack操作类似，我们可以基于scan操作实现unpack功能。图14-25展示了如何在排他性扫描（exclusive scan）的基础上实现子组解包（sub-group unpack）操作。  

```cpp
uint32_t index = 
    exclusive_scan(sg, (uint32_t)predicate, plus<>()); 
return (predicate) ? new_value[index] : original_value;
```

图14-26中的代码展示了如何利用这种子组解包操作来改进具有发散控制流的内核负载均衡（本例以曼德勃罗集计算为例）。每个工作项被分配计算独立像素，迭代执行直至收敛或达到最大迭代次数。随后通过解包操作将已完成的像素替换为新的待计算像素。  

```cpp
// 只要有一个工作项有待处理工作，就继续迭代  
while (any_of_group(sg, i < Nx)) {  
    uint32_t converged = next_iteration(  
        params, i, j, count, cr, ci, zr, zi, mandelbrot);  

    if (any_of_group(sg, converged)) {  
        // 使用解包操作替换已收敛的像素点  
        // 未收敛的像素点保持不变  
        uint32_t index = exclusive_scan_over_group(  
            sg, converged, plus<>());  
        i = (converged) ? iq + index : i;  
        iq += reduce_over_group(sg, converged, plus<>());  

        // 为新的i重置迭代器变量  
        if (converged) {  
            reset(params, i, j, count, cr, ci, zr, zi);  
        }  
    }  
}
```

此类方法对效率的提升程度（以及执行时间的缩短）高度依赖于具体应用场景和输入数据，因为检查完成状态和执行解包操作均会引入额外开销！因此，要在实际应用中成功运用该模式，需根据计算任务中的发散程度和执行内容进行精细调优（例如引入启发式规则：仅当活跃工作项数量低于特定阈值时才触发解包操作）。  

## 总结（summary）
本章展示了如何利用SYCL特性（包括内置函数和库）来实现一些最常见的并行模式。  

SYCL生态系统仍在发展之中，随着开发者对该语言的实践经验积累以及生产级应用程序和库的开发，我们预计将会发现这些模式的新最佳实践。  

### 更多信息（for more information）
+ 《结构化并行编程：高效计算的模式》，Michael McCool、Arch Robison 和 James Reinders 合著，© 2012 年由 Morgan Kaufmann 出版，ISBN 978-0-124-15993-8。 
+  C++ 参考之算法库，https://en.cppreference.com/w/cpp/algorithm。  

