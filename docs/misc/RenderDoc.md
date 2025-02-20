# 前言

对于图形程序开发者来说，学会使用RenderDoc图形调试器可以帮助你全面了解渲染管线绑定的资源和运行状态，从而确认问题所在。

[**RenderDoc官网**](https://renderdoc.org/)

# 运行程序

为了调试我们的程序，需要通过RenderDoc来执行程序。

![image-20220329201219288](..\assets\RenderDoc\01.png)

选择File - Launch Application后，在Program - Executable Path中选择要打开的程序。

> **注意：在你自己编写的项目需要将exe放到项目(.vcxproj)所在的位置，或者让VS在生成程序的时候输出到项目位置！**

如果待调试的程序需要加载Assimp的动态库，我们还需要添加环境变量：

![image-20220329201114026](..\assets\RenderDoc\02.png)

然后就可以点击Launch运行程序了。

# 截取一帧画面

在进入程序后，按下Print Screen(PrtSc)键截取一帧有问题的画面，然后就可以看到程序窗口说已经捕获了一帧：

![img](..\assets\RenderDoc\03.png)

捕获完成后退出程序即可，捕获的一帧文件类型为`*.rdc`

你可以在一次调试截取多帧画面，但基本上目前我们只需要截取一帧画面就可以退出程序了。

# 事件浏览器(Event Broser)

下面是图形调试器的主界面：

![image-20220329201732492](..\assets\RenderDoc\04.png)

**事件浏览器**展示了DirectX中关于`ID3D11DeviceContext`的重要调用，呈现了这一帧绘制涉及到的`Clear`、`Draw`、`Dispatch`、`Present`和`Resolve`等命令。选择具体某个事件，可以在下面的`API Inspector`看到在这个事件之前大概15个`DeviceContext`的调用事件。

事件浏览器会将绘制到同一系列渲染目标和深度缓冲区的事件折叠成一个Pass，我们可以展开观察里面的具体绘制过程。

**在选中某次绘制后**，我们可以观察的有：

- **Texture Viewer**：完成当前绘制后渲染目标的结果、深度缓冲区的结果、像素着色器调试
- **Pipeline State**：观察当前渲染管线有哪些阶段是被激活的，以及不同的阶段状态是怎样的
- **Mesh Viewer**：观察当前正在渲染的模型从顶点输入是什么情况，经过顶点着色输出后又是什么情况，并且能够观察正在渲染的模型
- **Resource Inspector**：观察当前绘制后有哪些资源，状态如何

接下来会按教程的顺序来讲可能需要查看的内容

## Pipeline State

在管线状态中我们可以清楚地看到当前有哪些执行的阶段，选择IA（输入装配阶段）可以看到**输入布局**、**顶点缓冲区输入**和**图元类型**

![image-20220329232515181](..\assets\RenderDoc\05.png)

如果找不到窗口可以去菜单栏Window找到Pipeline State。

## Mesh Viewer

点击上图中的Mesh View内的立方体可以跳转到模型线框观察页面，同时可以观察输入的顶点数据：

![image-20220329233055825](..\assets\RenderDoc\06.png)

通过Controls可以切换摄像机模式为第一人称，然后使用WSAD移动

如果屏幕上没有渲染出想要的东西，首先应当检查的是输出的顶点`SV_POSITION`是否位于NDC空间内，具体为：
$$
-1\leq\frac{x}{w}\leq 1\\
-1\leq\frac{y}{w}\leq 1\\
0\leq\frac{z}{w}\leq 1
$$
要调试某个顶点，只需要在VS Input中选择一个顶点右键 - Debug this vertex即可进入着色器调试。但调试环节我们留到后面再讲。

## Texture Viewer

![image-20220329204409574](..\assets\RenderDoc\07.png)

在Texture Viewer中我们可以观察绑定到管线上的图片(Input)，以及渲染管线输出到的渲染目标、深度缓冲区(Output)。在选择某个Output图后，我们右键选中一个像素，右下角的Pixel Context就会显示具体的位置：

![image-20220329204859358](..\assets\RenderDoc\08.png)

选择History可以查看在此之前有哪些绘制事件影响到当前像素，选择Debug则可以调试当前像素。

### 观察深度/模板缓冲区

选中深度/模板缓冲区，一般情况下越远的物体显得越白，越近显得越黑，且深度图的颜色分布大多在白色上。

![image-20220329205947775](..\assets\RenderDoc\09.png)

而如果使用了反向Z，越远的物体显得越黑，越近显得越白，且分布大多在黑色上，这时候看深度图就是纯黑一片，根本不知道什么情况：

![image-20220329210052144](..\assets\RenderDoc\10.png)

由于此时深度值大部分在靠近0的位置上，我们需要缩小显示范围来提高较远物体的亮度：

![image-20220329210255396](..\assets\RenderDoc\28.png)

为了观察模板测试的结果，我们先选中Stencil，如果模板的输出值为1，可能需要将Range右边的条拖到最左边才看得到（白色区域模板值为1，黑色区域模板值为0）：

![image-20220329211321085](..\assets\RenderDoc\11.png)

在Overlay中，我们可以观察当前绘制中影响到的像素区域、深度测试（绿Pass红Fail）、模板测试、背面剔除等结果。下图演示了模型的线框在图中的位置：

![image-20220329211834124](..\assets\RenderDoc\12.png)

## Resource Inspector

在这里可以观察与当前绘制相关的所有资源：

![image-20220330095847660](..\assets\RenderDoc\27.png)

选中某个资源后，可以看到和它相关的资源、资源在哪些事件中被用到、资源初始化相关的调用。

## 观察常量缓冲区

在管道状态的着色器阶段中，我们可以看到绑定的常量缓冲区：

![image-20220329235005935](..\assets\RenderDoc\13.png)

其中Slot的名称来自着色器声明cbuffer时的名称，Buffer的名称则需要在C++代码中设置，具体参考下一节。

选择某一个常量缓冲区，点击Go处的箭头，我们就可以看到里面的具体内容：

![image-20220329235216150](..\assets\RenderDoc\14.png)

>  **注意：在当前教程中我们会传入经过DirectXMath转置后的矩阵，但是在这里观察值的时候，依然是以行矩阵的方式显示才是正常的！即平移分量位于第四行。**

若常量缓冲区的值在从C++端传入到这里出现问题，你还需要去观察常量缓冲区的打包是否出现了问题。

关于HLSL的打包规则，可以查看这里：
[深入理解HLSL常量缓冲区打包规则](misc/Packing.md)

## 为图形调试器的对象添加自定义名称

看前面的图片，Buffer在没有指定名称的时候默认是以`Buffer 142`的形式显示的。等对象一多，我们就难以判别管线所绑定的对象是否正确。因此在某些需要的情况下，我们可以在C++代码来为对象指定名称。

在`d3dUtil.h`中提供了两个系列的函数，一个用于D3D设备创建出来的对象，一个用于DXGI对象。通过`SetPrivateData`方法，并使用`WKPDID_D3DDebugObjectName`的`GUID`使得我们可以为其设置图形调试器下的名称（string_view版本要求C++17，或者可以参照旧`d3dUtil.h`中的实现）：

```cpp
// ------------------------------
// D3D11SetDebugObjectName函数
// ------------------------------
// 为D3D设备创建出来的对象在图形调试器中设置对象名
// [In]resource				D3D11设备创建出的对象
// [In]name					对象名
inline void D3D11SetDebugObjectName(_In_ ID3D11DeviceChild* resource, _In_ std::string_view name)
{
#if (defined(DEBUG) || defined(_DEBUG)) && (GRAPHICS_DEBUGGER_OBJECT_NAME)
	resource->SetPrivateData(WKPDID_D3DDebugObjectName, (UINT)name.length(), name.data());
#else
	UNREFERENCED_PARAMETER(resource);
	UNREFERENCED_PARAMETER(name);
#endif
}

// ------------------------------
// D3D11SetDebugObjectName函数
// ------------------------------
// 为D3D设备创建出来的对象在图形调试器中清空对象名
// [In]resource				D3D11设备创建出的对象
inline void D3D11SetDebugObjectName(_In_ ID3D11DeviceChild* resource, _In_ std::nullptr_t)
{
#if (defined(DEBUG) || defined(_DEBUG)) && (GRAPHICS_DEBUGGER_OBJECT_NAME)
	resource->SetPrivateData(WKPDID_D3DDebugObjectName, 0, nullptr);
#else
	UNREFERENCED_PARAMETER(resource);
#endif
}

// ------------------------------
// DXGISetDebugObjectName函数
// ------------------------------
// 为DXGI对象在图形调试器中设置对象名
// [In]object				DXGI对象
// [In]name					对象名
inline void DXGISetDebugObjectName(_In_ IDXGIObject* object, _In_ std::string_view name)
{
#if (defined(DEBUG) || defined(_DEBUG)) && (GRAPHICS_DEBUGGER_OBJECT_NAME)
	object->SetPrivateData(WKPDID_D3DDebugObjectName, (UINT)name.length(), name.c_str());
#else
	UNREFERENCED_PARAMETER(object);
	UNREFERENCED_PARAMETER(name);
#endif
}

// ------------------------------
// DXGISetDebugObjectName函数
// ------------------------------
// 为DXGI对象在图形调试器中清空对象名
// [In]object				DXGI对象
inline void DXGISetDebugObjectName(_In_ IDXGIObject* object, _In_ std::nullptr_t)
{
#if (defined(DEBUG) || defined(_DEBUG)) && (GRAPHICS_DEBUGGER_OBJECT_NAME)
	object->SetPrivateData(WKPDID_D3DDebugObjectName, 0, nullptr);
#else
	UNREFERENCED_PARAMETER(object);
#endif
}
```

在已经设置过名字的情况下，想要更名需要先调用`nullptr_t`重载版本，再调用正常版本。

![image-20220330000345180](..\assets\RenderDoc\15.png)

设置好后，在图形调试的时候一看名字就能知道绑定的情况了。

如果你不希望使用调试器对象具名化，可以在`d3dUtil.h`的开头找到这样的宏：

```cpp
// 默认开启图形调试器具名化
// 如果不需要该项功能，可通过全局文本替换将其值设置为0
#ifndef GRAPHICS_DEBUGGER_OBJECT_NAME
#define GRAPHICS_DEBUGGER_OBJECT_NAME (1)
#endif
```

将其修改后只会剩下默认的`DDSTextureLoader`和`WICTextureLoader`的对象具名化。

**注意：在你的Release版本应用程序应该避免出现对调试对象名称的设置。你可以将相关代码移出项目。**



## 查看着色器资源视图中的纹理资源

以下图像素着色器阶段的为例：

![image-20220330000750678](..\assets\RenderDoc\16.png)

我们可以很清楚地看到资源的绑定情况，红色表示当前Slot没有资源绑定上去，如果对没有绑定纹理的对象进行采样，会在程序调试运行时的调试输出窗口看到DX Error。当然本示例红的也并不影响，因为会在着色器检查Dimension是否为0从而避开采样。

绿色的资源姑且认为是一个有`UNKNOWN`含义的DXGI格式，在通过SRV具体化。点击Go的箭头我们可以观察传入的着色器资源。

## 查看管线状态、采样器

基本上光栅化状态、深度/模板状态和混合状态都是所见即所得

![image-20220330004713099](..\assets\RenderDoc\17.png)

![image-20220330005056885](..\assets\RenderDoc\18.png)

采样器则在像素着色器阶段选中采样器可以查看

![image-20220330005209364](..\assets\RenderDoc\19.png)

虽然这些状态你也可以在C++看

# 着色器调试

接下来就开始进入到重点部分了，使用图形调试器的核心目的还是要观察着色器运行的时候遇到了哪些问题。当然有时候甚至会遇到该有的着色器却被跳过不执行的情况，这时候就先要去前面排查该绑定的资源、状态、着色器、输入是否都OK了，然后才是对上一个正常运行的着色器进行调试。

对于顶点着色器，在Mesh Viewer中选择要调试的顶点右键 - Debug this vertex即可

![image-20220330011212478](..\assets\RenderDoc\20.png)

对于像素着色器，在Texture Viewer中的Output选择RT后，右键选取某一像素，在Pixel Context处点Debug即可

![](..\assets\RenderDoc\08.png)

而调试计算着色器，需要在Pipeline State选择CS，按下图选择Debug，然后填写要调试的线程组编号和组内线程编号（或者全局线程ID）：

![image-20220330011004473](..\assets\RenderDoc\21.png)

然后就进入到了着色器调试界面：

![image-20220330005757769](..\assets\RenderDoc\22.png)

因为鼠标操作麻烦，我们需要记住几个快捷键：**F10**单步跳过，**F11**单步进入，**ctrl+F11**单步跳出

左侧`Constants & Resources`可以查看顶点输入、使用的常量、资源等，右侧`Watch`可以添加变量观察

鼠标悬停在代码的变量可以观察变量值

右键代码Go to disassembly可以转汇编查看

左侧file list可以查看用到的hlsl文件，以及编译shader时候的预定义宏

此时首先你需要优先关注局部变量中各个会被用到的常量、输入值是否都是正常的，如果出现常量缓冲区中的值全0或者乱值的情况，说明常量缓冲区可能没有被更新。

## 修改着色器再运行

这是VS的图形调试器所没有的功能，在修改了某次绘制用到的着色器代码并编译后，就可以影响到当前及之后的所有绘制。

下面是一个例子，这里尝试修改某个绘制的像素着色器代码：

![image-20220330093349754](..\assets\RenderDoc\23.png)

然后尝试修改下面`g_VisualizePerSampleShaing`为`true`，使得当前绘制的像素颜色强制为红色：

![image-20220330093454839](..\assets\RenderDoc\24.png)

完成后选择`Apply changes`，返回`Texture Viewer`观察渲染目标的输出变化：

![image-20220330093929058](..\assets\RenderDoc\25.png)

可以看到，那些执行PS的像素都被染成了红色，观看后续的帧也可以发现的确产生了影响：

![image-20220330093913385](..\assets\RenderDoc\26.png)

如果要退回变化，则回到像素着色器的Edit处，选择`Remove changes`即可。

# 以编程方式捕获图形信息

因为目前暂时还没有使用的需要，具体信息查看下面文档：

https://renderdoc.org/docs/in_application_api.html

如果某些DrawCall、Dispatch不是每帧都会产生的话，编程捕获的方式还是有必要的。

# 总结

**调试技巧需要经常使用才能够熟练掌握**，相比普通调试来说，图形调试会更加复杂。目前RenderDoc的调试体验比VS的图形调试器会好一些，并且最近VS的图形调试器有些问题，调试不了shader。**在初学DX的阶段容易在资源管理上出问题，因此重点是要先确认在绘制之前，绑定到渲染管线的各种资源是否正常，然后才是对着色器代码进行调试。**所以前期准备工作的出错一般占很大的一部分，而着色器代码引发的错误可能只是占较小的一部分。**等到了渲染管线的资源绑定管理体系逐渐稳定以后，使用图形调试的重心才会逐渐转移到以着色器代码的调试为主。**有时候图形调试器解决不了的问题，还需要仔细观察普通调试下的输出窗口是否有渲染管线绘制事件执行时输出的报错信息。

当然里面还有很多强大的功能没有挖掘出来，或者现在还不是比较常用而没列出来。有兴趣的读者可以查看renderdoc的文档：

[Introduction — RenderDoc documentation](https://renderdoc.org/docs/introduction.html)

这篇博客在后续还会有所变动，因为后续个人的学习会引发新的调试需求而变动。
