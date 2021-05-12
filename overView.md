# Overview(概览)
+ ## [Origin of Vulkan](#originVulkan)
    - ## [What it takes to draw a triangle](#wit)
    - ## [Step 1 - Instance and physical device selection](#step1)
    - ## [Step 2 - Logical device and queue families](#step2)
    - ## [Step 3 - Window surface and swap chain](#step3)
    - ## [Step 4 - Image views and framebuffers](#step4)
    - ## [Step 5 - Render passes](#step5)
    - ## [Step 6 - Graphics pipeline](#step6)
    - ## [Step 7 - Command pools and command buffers](#step7)
    - ## [Step 8 - Main loop](#step8)
    - ## [Summary](#summary_a)
+ ## [API concepts](#api_concept)
    - ## [Coding conventions](#codingCov)
    - ## [Validation layers](#validation)


> # <A NAME="originVulkan">Origin of Vulkan(Vulkna的由来)</a>
> + 和以前的图形API一样，Vulkan被设计为基于GPU的跨平台抽象。大部分这些图形API的问题在于在它们被设计的时代就采用了图形硬件，而图形硬件被固定的功能所限制。程序员需要用标准的格式去提供顶点的数据，并且灯光和阴影选项方面受GPU制造商的支配。
> + 随着显卡架构的越发成熟，他们开始提供越来越多的可编程的功能。所有的这些新的功能必须以某些方式与现有的API集成到一起。 This resulted in less than ideal abstractions and a lot of guesswork on the graphics driver side to map the programmer's intent to the modern graphics architectures.   这就是为什么要进行如此多的显卡驱动更新去提高游戏的性能，有时是很大幅度的提升。由于这些驱动的复杂性，应用软件的程序开发人员还需要处理供应商之间的不一致问题，例如着色器的语法。除了这些新功能，过去十年来还涌入了具有强大图形硬件的移动设备。这些移动GPU根据其性能和空间要求具有不同的体系结构。其中的一个例子就是tiled rendering（平铺渲染？？），它可以通过为程序员提供更多对该功能控制权来提高性能。源自这些API的另外一个限制就是有限的多线程支持，这可能会导致CPU的瓶颈。
> + Vulkan通过从头开始设计用于现代图形体系结构来解决这些问题。通过允许程序员使用更详细的API明确指定其意图，它减少了驱动程序的开销，并且允许多个线程并行来创建和提交命令。通过使用单个编译器切换到标准字节代码格式，它可以减少着色器编译中的不一致。 最后，它通过将图形和计算功能统一为一个API来认可现代图形卡的通用处理能力。

> # <A NAME="wit">What it takes to draw a triangle</a>
> + 现在我们将概述一下在一个良好的vulkan程序中渲染三角形所需的所有步骤。下一章将详细介绍这里介绍的所有概念。 这只是为了让您有一个大的图景，可以将所有单个组件相关联。


> # <A NAME="step1">Step 1 - Instance and physical device selection</a>
> + ### 关键词：VkInstance VkPhysicalDevices
> + 一个Vulkan的软件通过VKInstance去设置Vulkan API。通过描述一个应用程序以及任何将要用到的API扩展来创建一个实例。在创建了实例之后，可以查询Vulkan支持的硬件，并且选择一个或者多个VkPhysicalDevices来用于操作。可以查询诸如VRAM大小和设备功能之类的属性，以用来选择所需的设备，例如偏向于使用专业的图形卡。

> # <A NAME="step2">Step 2 - Logical device and queue families</a>
> + ### 关键词：VkDevice VkPhysicalDeviceFeatures queue_families VkQueue 
> + Physical_device
逻辑设备以及队列系列
在选择好正确的使用的硬件设备之后，我们需要去创建一个VkDevice（logical device），这是我们用来更加具体的描述我们将要使用的VkPhysicalDeviceFeatures，例如像多Viewport的渲染以及64位浮点数。同时我们需要去具体的指出我们需要用到哪个队列族（queue families）。通过将Vulkan执行的大多数操作（例如绘制命令和内存操作）提交给VkQueue可以异步执行。Queue是由queue family分配的。每一个queue family可以在这个队列中支持一组特定的操作。例如，可能有用于图形，计算和内存传输操作的单独队列系列。Queue family同样可以用来去作为一个判别因素去选择physical device。支持Vulkan的设备有可能不提供任何图形功能，但是当今所有具有Vulkan支持的图形卡通常都将支持我们感兴趣的所有队列操作。

> # <A NAME="step3">Step 3 - Window surface and swap chain</a>
> + ### 关键词：VkSurfaceKHR VkSwapchainKHR Surface render_target
> + 除非只对离屏的渲染感兴趣，我们都需要创建一个窗口用来展示图像。我们可以用原生的平台API或者类似GLFW和SDL去创建窗口。在这个教程中我们将使用GLFW，但是将在后续的文章中去讲述。
我们还需要另外两个组件才能真正的去渲染进入窗口：一个是window surface（VkSurfaceKHR）以及交换链（VkSwapchainKHR）。记住这个KHR后缀，它代表这个对象是Vulkan扩展的一部分。Vulkan API本身是完全和平台无关联的，因此我们需要一个标准化的WSI(window system interface)扩展去和窗口管理器交互。Surface是一个需要渲染到窗口上的跨平台抽象，通常通过提供对当前平台的窗口handle的引用来实例化，例如windows系统上的HWND。幸运的是，GLFW这个库已经将处理指定平台的功能已经搭建好了。
交换链是render target的一个集合。它主要是确保我们将要渲染的图像和当前存在在屏幕上的图像不一样。这对于确保渲染一个完整的图像非常重要。每次我们想要去绘制一帧我们都必须让交换链提供给我们用于渲染的图像。当我们完成绘制一帧时，这一帧的图像会被返回给交换链中去显示在屏幕上。render target和将完成的图像呈现至屏幕的数量取决于当前的显示模式（present mode）。通常的显示模式有双缓冲（double buffering（vsync））和三缓冲。我们将在交换链创建的这一章节中去详细的讲述研究。
某些平台允许您直接渲染到显示器，而无需通过VK_KHR_display和VK_KHR_display交换链扩展与任何窗口管理器进行交互。这些可以允许你创建一个代表整个屏幕的表面。例如可以实现一个我们自己的窗口管理器。

> # <A NAME="step4">Step 4 - Image views and framebuffers</a>
> + ### 关键词：VkImageView VkFrameBuffer image_views
> + 为了去绘制一个从交换链中获得的图像，我们需要将其绑定到VkImageView以及VkFrameBuffer中。一个图像引用（image views）引用了要使用图片的特定部分，而帧缓冲（framebuffers）则引用了图像引用中的颜色，深度，模板目标（stencil targets）。因为在交换链中可能有许多不同的图像，我们需要在这之前先为其将图像视图（image view）以及帧缓冲创建出来并且在绘制的时候选择合适的图像视图和帧缓冲。

> # <A NAME="step5">Step 5 - Render passes</a>
> + ### 关键词：VkFrameBuffer color_target render_pass
> + Vulkan中的渲染过程？（render pass）描述了渲染操作期间使用的图像类型，将如何使用它们，以及应如何处理其内容。在最初的三角形渲染应用程序中，我们将要告诉Vulkan我们将要只使用一个图像作为颜色目标？（color target）并且想要在绘制操作之前将其清除为纯色。渲染过程仅描述图像的类型，而VkFramebuffer实际上将特定图像绑定到这些插槽。

> # <A NAME="step6">Step 6 - Graphics pipeline</a>
> + ### 关键词：VkShaderModule graphics_pipeline render_pass
> + Vulkan中的图形管线（graphics pipeline）是通过创建VkPipeline对象用来设置。它描述了显卡的可配置状态，例如视口大小和深度缓冲区操作以及使用VkShaderModule对象的可编程状态。VkShaderModule对象由着色器字节码（shader byte code）所创建。驱动还需要知道管线中需要用到什么渲染目标（render target），我们通过引用渲染过程来指定这些目标。
> + 与现有的API相比，Vulkan最具特色的功能之一是几乎所有的图形管道配置都需要预先配置。这意味着如果我们想切换成不同的shader或者略微的改变顶点布局（vertex layout），则需要完全的重新创建图形管线。这同时也以为这我们将必须为渲染操作所需的所有不同的组合预先创建许多的VkPipeline对象。只有一些基础的配置，例如视窗大小以及clean color，可以被动态的去改变。所有的状态也需要被明确的描述，例如没有默认的颜色混合状态。
> + 好消息是，因为我们所作的相当于提前编译与即时编译，这样的驱动程序有跟多的被优化的机会以及runtime的性能更可以被预测。因为大型的状态切换（例如切换到不同的图形管线）变的非常明确。

> # <A NAME="step7">Step 7 - Command pools and command buffers</a>
> + ### 关键词：VkCommandBuffer VkCommandPool queue_family
> + 就想之前所讲的，我们需要在Vulkan中执行的操作（例如绘图操作）都需要去提交到队列中。这些操作首先需要被记录在VkCommandBuffer中，然后才能提交。这些命令缓冲（command buffers）由VkCommandPool分配且这个CommandPool与指定的queue family联系在一起。为了去绘制一个简单的三角形，我们需要使用以下的操作来记录为command buffer：
>   - 开始渲染过程（render pass）
>   - 绑定一个图形管线（graphics pipeline）
>   - 绘制三个顶点
>   - 结束这个渲染过程
> + 因为帧缓冲中的图像取决于交换链将为我们提供的特定图像，我们需要为每一个可能的图像记录一个命令缓冲，并在绘制时选择一个正确的图像。另一种方法就是每帧再次记录命令缓冲，但是效率不高。

> # <A NAME="step8">Step 8 - Main loop</a>
> + ### 关键词：vkAcquireNextImageKHR vkQueueSubmit vkQueuePresentKHR
> + 现在，绘制命令已经被包装到命令缓冲中，主循环非常简单。我们首先使用vkAcquireNextImageKHR从交换链中获取图像。然后我们可以为这个图像选择一个合适的命令缓冲，并用vkQueueSubmit去执行它。最后我们使用vkQueuePresentKHR将图像返回到交换链中，并且呈现到屏幕上。
> + 提交到队列的操作是异步执行的。因此，我们必须使用诸如信号量之类的同步对象来确保正确的执行顺序。必须设置绘制命令缓冲区的执行以等待图像获取完成，否则可能会开始渲染到仍在读取以在屏幕上呈现的图像。反过来，vkQueuePresentKHR调用需要等待渲染完成，为此，我们将使用第二个信号量，该信号量在渲染完成后发出信号。

> # <A NAME="summary_a">Summary</a>
> + 这次快速的旅途将使您对绘制第一个三角形的工作有一个基本的了解。 实际程序包含更多步骤，例如分配顶点缓冲区，创建统一缓冲区和上传纹理图像，这些将在后续章节中介绍，但我们将从简单开始，因为Vulkan拥有足够的陡峭学习曲线。 请注意，我们将通过在顶点着色器中最初嵌入顶点坐标而不是使用顶点缓冲区来作弊。 这是因为管理顶点缓冲区需要首先熟悉命令缓冲区。
> + ## 简而言之，要绘制第一个三角形，我们需要：
>   - 创建一个VkInstance
>   - 选择一个支持的显卡设备（VkPhysicalDevice）
>   - 创建一个VkDevice和VkQueue用来绘制并且呈现
>   - 创建一个窗口，窗口面（window surface）和交换链（swap chain）
>   - 将交换链中的图片包入VkImageView中
>   - 创建一个渲染过程（render pass）用来指定渲染目标（render target）和用途
>   - 为渲染过程创建一个帧缓冲
>   - 设置一个图形渲染管线
>   - 用绘制命令（draw command）为每一个可能的交换链中的图片分配并且记录一个command buffer
>   - 通过获取图像，提交正确的绘制命令缓冲区并将图像返回到交换链来绘制框架
> + 这是很多步骤，但是在接下来的章节中，每个步骤的目的都会变得非常简单明了。 如果您对单个步骤与整个程序之间的关系感到困惑，则应返回本章。

> # <A NAME="api_concept">API Concept</a>
>> # <A NAME="codingCov">Coding conventions(代码规则)</a>
>> #### Vulkan中所有的功能，枚举，结构体都被定义在vulkan.h这个头文件中，而这个头文件被包含在LunarG开发的Vulkan SDK中，我们将在下一个章节中研究安装这个SDK。
>> +  函数具有小写的vk前缀
>> +  枚举，结构体这类的是VK作为前缀并且枚举值具有的是VK_前缀
>> +  API大量使用结构体为函数提供参数。 例如，对象创建通常遵循以下模式：
>> ```cpp
>>VkXXXCreateInfo createInfo{};
>>createInfo.sType = VK_STRUCTURE_TYPE_XXX_CREATE_INFO;
>>createInfo.pNext = nullptr;
>>createInfo.foo = ...;
>>createInfo.bar = ...;
>>VkXXX object;
>>if (vkCreateXXX(&createInfo, nullptr, &object) != VK_SUCCESS) 
>>{
>>    std::cerr << "failed to create object" << std::endl;
>>    return false;
>>}
>> ```
>>
>>  Vulkan中的许多结构体要求我们在sType成员中显示指定结构的类型。pNext成员可以指向一个扩展的结构体而在本教程中始终为nullptr。创建或者销毁一个对象的函数将需要VkAllocationCallbacks参数，这个参数允许我们用来去自定义地用来分配驱动内存的分配器，在这个教程中也始终为nullptr。
>> 几乎所有的函数都返回一个VkResult，这个结果要么是VK_SUCCESS或者一段错误的代码。这个规范面熟了每个函数可以返回的错误代码以及它们的含义。