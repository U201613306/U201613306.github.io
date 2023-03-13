---
layout: post
title:  OpenGL 学习笔记（搬运中）
categories: Gecko
description: 笔记搬运中ing
keywords: 读书笔记, 图形学，opengl，code 

---

# 写在前面

本文是学习OpenGL的过程中所做的笔记，因为时间关系，虽然笔记已做完，但是还有相当部分待整理收集，我会持续把他搬运过来



![image-20230313213359904](C:\Users\xue\AppData\Roaming\Typora\typora-user-images\image-20230313213359904.png)





# 关于OpenGL

实际的OpenGL库的开发者通常是显卡的生产商。你购买的显卡所支持的OpenGL版本都为这个系列的显卡专门开发的。当你使用Apple系统的时候，OpenGL库是由Apple自身维护的。在Linux下，有显卡生产商提供的OpenGL库，也有一些爱好者改编的版本。这也意味着任何时候OpenGL库表现的行为与规范规定的不一致时，基本都是库的开发者留下的bug。

===由于OpenGL的大多数实现都是由显卡厂商编写的，当产生一个bug时通常可以通过升级显卡驱动来解决。这些驱动会包括你的显卡能支持的最新版本的OpenGL，这也是为什么总是建议你偶尔更新一下显卡驱动。===

# opengl 3.3规范文档
![[glspec33.core.pdf]]



[opengl各版本规范文档](https://registry.khronos.org/OpenGL/index_gl.php)



# 前言
##  立即渲染模式和核心模式

早期的OpenGL使用===立即渲染模式（Immediate mode，也就是固定渲染管线）===，这个模式下绘制图形很方便。OpenGL的大多数功能都被库隐藏起来，开发者很少有控制OpenGL如何进行计算的自由。而开发者迫切希望能有更多的灵活性。随着时间推移，规范越来越灵活，开发者对绘图细节有了更多的掌控。立即渲染模式确实容易使用和理解，但是效率太低。===因此从OpenGL3.2开始，规范文档开始废弃立即渲染模式，并鼓励开发者在OpenGL的核心模式(Core-profile)下进行开发，这个分支的规范完全移除了旧的特性。===

===OpenGL使用新版本特性，就必须要最新的显卡才能支持，这也是为什么新特性一般做成开关，交给用户自己启用。===

## OpenGL本质是状态机


状态机的概念有点类似于渲染管线，都是一直在运行，内容不同，输出不同。上下文类似于当前执行的任务或设定。
==OpenGL自身是一个巨大的状态机(State Machine)：==一系列的变量描述OpenGL此刻应当如何运行。==OpenGL的状态通常被称为OpenGL上下文(Context)==。我们通常使用如下途径去更改OpenGL状态：==设置选项，操作缓冲==。最后，我们==使用当前OpenGL上下文来渲染==。

假设当我们想告诉OpenGL去画线段而不是三角形的时候，我们通过改变一些上下文变量来改变OpenGL状态，从而告诉OpenGL如何去绘图。一旦我们改变了OpenGL的状态为绘制线段，下一个绘制命令就会画出线段而不是三角形。

当使用OpenGL的时候，我们会遇到一些==状态设置函数(State-changing Function)==，这类函数将会改变上下文。以及==状态使用函数(State-using Function)==，这类函数会根据当前OpenGL的状态执行一些操作。只要你记住OpenGL本质上是个大状态机，就能更容易理解它的大部分特性。


## 一段简单的OpenGL代码

OpenGL库是用C语言写的，同时也支持多种语言的派生，但其内核仍是一个C库。由于C的一些语言结构不易被翻译到其它的高级语言，因此OpenGL开发的时候引入了一些抽象层。“对象(Object)”就是其中一个。

==opengl 引入了面向对象的思想==，对象代表一种状态机的设置，只需要把状态机绑定到设置，就可以轻松实现切换


``` c

// OpenGL的状态 (上下文)
struct OpenGL_Context { 
... 
object* object_Window_Target;
... };


// 创建对象 
unsigned int objectId = 0; 
//objectId用引用的形式，保存前面的数字为一个对象
glGenObject(1, &objectId); 
// 绑定对象至上下文
glBindObject(GL_WINDOW_TARGET, objectId); 
// 设置当前绑定到 GL_WINDOW_TARGET 的对象的一些选项 
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_WIDTH, 800); glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_HEIGHT, 600);
// 将上下文对象设回默认，
//绑0等于让GL_WINDOW_TARGET解绑该对象
glBindObject(GL_WINDOW_TARGET, 0);




```

这一小段代码展现了你以后使用OpenGL时常见的工作流。
我们首先创建一个对象，然后用一个id保存它的引用（实际数据被储存在后台）。然后我们将对象绑定至上下文的目标位置（窗口对象目标的位置被定义成GL_WINDOW_TARGET）。接下来我们设置窗口的选项。最后我们将目标位置的对象id设回0，解绑这个对象。设置的选项将被保存在objectId所引用的对象中，一旦我们重新绑定这个对象到GL_WINDOW_TARGET位置，这些选项就会重新生效。

## GLFW+GLAD

用GLFW环境创建OpenGL上下文，并显示出一个简单的窗口来让我们随意使用。

因为OpenGL只是一个标准/规范，具体的实现是由驱动开发商针对特定显卡实现的。由于OpenGL驱动版本众多，它大多数函数的位置都无法在编译时确定下来，需要在运行时查询，比如

创建一个函数指针，调用==wglGetProcAddress()==查询函数指针位置，再利用返回值实例化一个相匹配的函数指针，将函数指针实例作为可执行对象调用，来做到调用特定函数


```c
// 定义函数原型

typedef void (*GL_GENBUFFERS) (GLsizei, GLuint*); 
// 找到正确的函数并赋值给函数指针
GL_GENBUFFERS glGenBuffers = (GL_GENBUFFERS)wglGetProcAddress("glGenBuffers"); 
// 现在函数可以被正常调用了
GLuint buffer; glGenBuffers(1, &buffer);
```
比较麻烦。。。

==使用GLAD创建一个匹配我们的硬件和opengl版本的包含所用函数的头文件，就可以在编程时直接使用对应函数。


所有配置的关键：
1.cmake编译GLFW，生成**glfw3.lib**静态库。
2.为所有的工程新建**Libs**和**Includes**文件夹
3.在工程属性 --静态包含 添加 目录 **Libs**   （这样链接库就可以在**Libs** 链接你想要的.lib）
在include包含里添加**Includes**，这样就可以用系统包含+相对路径include头文件（include<FL/w.h>）

4.  在**Linker(链接器)** 选项卡里的**Input(输入)**选项卡里添加** **glfw3.lib**，因为之前配置过静态链接库路径，此时链接器可以直接找到glfw3.lib。
5.最后，同样用链接器添加opengl的支持库 opengl32.lib(==因为 opengl默认包含于windows SDK中，所以不需要添加lib包含路径,链接器也能直接找到它==)
6.在线网站编译出GLAD的 头文件 .h和 c文件，添加进工程。


#     示例1 窗口打开与关闭

```cpp


#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <iostream>


void framebuffer_size_callback(GLFWwindow* window, int width, int height);
void processInput(GLFWwindow* window);

int main()
{
	//配置GLFW，将主版本号(Major)和次版本号(Minor)都设为3。
	//我们同样明确告诉GLFW我们使用的是核心模式(Core-profile)
	glfwInit();
	glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
	glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
	glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
	//glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
	GLFWwindow* window = glfwCreateWindow(800, 600, "LearnOpenGL", NULL, NULL);
	if (window == NULL)
	{
		std::cout << "Failed to create GLFW window" << std::endl;
		glfwTerminate();
		return -1;
	}
	//将窗口设置为当前线程的主上下文
	glfwMakeContextCurrent(window);
	//GLAD是用来管理OpenGL的函数指针的，所以在调用任何OpenGL的函数之前我们需要初始化GLAD
	//gladLoadGLLoader（） 判断glad 加载任何opengl的函数是否成功
	if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
	{
		std::cout << "Failed to initialize GLAD" << std::endl;
		return -1;
	}
	//回调函数，顾名思义，就是使用者自己定义一个函数，使用者自己实现这个函数的程序内容，
	//然后把这个函数作为参数传入别人（或系统）的函数中，由别人（或系统）的函数在运行时来调用的函数。
	//函数是你实现的，但由别人（或系统）的函数在运行时通过参数传递的方式调用，
	//回调 函数不由自己使用，由别人使用
	//回调函数虽然不符合编程直觉，但是符合事件发生的逻辑，很类似于虚幻的蓝图
	glViewport(0, 0, 800, 600);
	//注册回调函数，告诉GLFW我们希望每当（SetFramebufferSize）窗口调整大小的时候调用framebuffer_size_callback函数
	//FramebufferSize  帧缓冲的大小其实就是窗口大小
	glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

	//循环
	//glfwWindowShouldClose函数在我们每次循环的开始前检查一次GLFW是否被要求退出
	while (!glfwWindowShouldClose(window))
	{
		//processInput封装了glfwGetKey，使得判断按键函数的接口变得统一
		processInput(window);

		//交换缓冲帧（应用程序默认双缓冲）
		glfwSwapBuffers(window);
		//双缓冲(Double Buffer)
		//为什么双缓冲？因为我们不能把正在绘制的图像输出，那样很不连贯。
		//应用程序使用单缓冲绘图时可能会存在图像闪烁的问题。 
		//这是因为生成的图像不是一下子被绘制出来的，而是按照从左到右，由上而下逐像素地绘制而成的。
		//最终图像不是在瞬间显示给用户，而是通过一步一步生成的，这会导致渲染的结果很不真实。
		//为了规避这些问题，我们应用双缓冲渲染窗口应用程序。
		//前缓冲保存着最终输出的图像，它会在屏幕上显示；而所有的的渲染指令都会在后缓冲上绘制。
		//当所有的渲染指令执行完毕后，我们交换(Swap)前缓冲和后缓冲，这样图像就立即呈显出来，之前提到的不真实感就消除了。

		//glfwPollEvents函数类似于刷新状态  检查有没有触发什么事件（比如键盘输入、鼠标移动等）、、
		//、更新窗口状态，并调用对应的回调函数（可以通过回调方法手动设置）。
		glfwPollEvents();

		//在每个新的渲染迭代开始的时候,由于颜色缓冲没有被清理，我们看到的仍然是上一次循环的结果
		//为了演示循环的频率，这里设置每次循环都会利用另一种颜色清理颜色缓冲的功能
		//容易知道，glClearColor函数是一个opengl的状态设置函数，而glClear函数则是对应的状态使用的函数
		glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
		glClear(GL_COLOR_BUFFER_BIT);





	}


	//回收glfw窗口资源
	glfwTerminate();




	return 0;
}




//该函数是回调函数，不直接运行，设计目的是每一次改变窗口大小时运行，
//因此要将其绑定为窗口的callback,通过glfwSetFramebufferSizeCallback（）实现
void framebuffer_size_callback(GLFWwindow* window, int width, int height)
{
	glViewport(0, 0, width, height);
}


//glfwGetKey判断  某个按键是否被按下
void processInput(GLFWwindow* window)
{
	if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
		glfwSetWindowShouldClose(window, true);
}



```


# 示例2 画三角形

### 代码：
先上代码：
```cpp



#include <glad/glad.h>
#include <GLFW/glfw3.h>

#include <iostream>


void framebuffer_size_callback(GLFWwindow* window, int width, int height);
void processInput(GLFWwindow* window);

// settings
const unsigned int SCR_WIDTH = 800;
const unsigned int SCR_HEIGHT = 600;


//顶点着色器shader代码（需要链接进opengl的program中）

//layout (location = 0) in vec3 aPos;  layout() 布局限定符，用于指定变量在内存中的顺序

const char* vertexShaderSource = "#version 330 core\n"
"layout (location = 0) in vec3 aPos;\n"
"void main()\n"
"{\n"
"   gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
"}\0";
const char* fragmentShaderSource = "#version 330 core\n"
"out vec4 FragColor;\n"
"void main()\n"
"{\n"
"   FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);\n"
"}\n\0";





int main()
{
	//初始化GLFW的显示窗口
	glfwInit();
	glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
	glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
	glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

#ifdef __APPLE__
	glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif

	//
	GLFWwindow* window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "LearnOpenGL", NULL, NULL);
	if (window == NULL)
	{
		std::cout << "Failed to create GLFW window" << std::endl;
		//回收资源
		glfwTerminate();
		return -1;
	}
	glfwMakeContextCurrent(window);
	glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

	if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
	{
		std::cout << "Failed to initialize GLAD" << std::endl;
		return -1;
	}


	//glCreateShader（创建的着色器类型）
	unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
	//将源码绑定到着色器对象vertexShader
	
	glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
	//在编译之前还可以用 glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success); 调试代码是否有错
	//使用 glGetShaderInfoLog(vertexShader, 512, NULL, infoLog);返回错误日志

	glCompileShader(vertexShader);
	// check for shader compile errors
	int success;
	char infoLog[512];
	glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);
	if (!success)
	{
		glGetShaderInfoLog(vertexShader, 512, NULL, infoLog);
		std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
	}
	// 同上，创建片段着色器
	unsigned int fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
	glShaderSource(fragmentShader, 1, &fragmentShaderSource, NULL);
	glCompileShader(fragmentShader);
	//错误日志
	
	glGetShaderiv(fragmentShader, GL_COMPILE_STATUS, &success);
	if (!success)
	{
		glGetShaderInfoLog(fragmentShader, 512, NULL, infoLog);
		std::cout << "ERROR::SHADER::FRAGMENT::COMPILATION_FAILED\n" << infoLog << std::endl;
	}






	// 这里类似于编译器的工作，把创建的shader 附加到 shaderprogram 程序上（整合起来），通过glLinkProgram 链接
	unsigned int shaderProgram = glCreateProgram();
	glAttachShader(shaderProgram, vertexShader);
	glAttachShader(shaderProgram, fragmentShader);
	//链接其实就是类似于IDE的链接过程
	glLinkProgram(shaderProgram);




	// 检查链接是否有误，实现同上
	glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
	if (!success) {
		glGetProgramInfoLog(shaderProgram, 512, NULL, infoLog);
		std::cout << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n" << infoLog << std::endl;
	}
	//在把shader附加到program，并链接成功后，shader代码就没用了，及时回收。
	glDeleteShader(vertexShader);
	glDeleteShader(fragmentShader);

	




	//
	float vertices[] = {
		-0.5f, -0.5f, 0.0f, // left  
		 0.5f, -0.5f, 0.0f, // right 
		 0.0f,  0.5f, 0.0f  // top   
	};

	//VAO与VBO  VBO绑定一个buffer对象（可以是GL_ARRAY_BUFFER），直接存储并向GPU发送数据
	//数据是需要切分的，不然GPU不认识。通过glVertexAttribPointer（）定义数据的切分格式  
	//VAO 顶点数组对象，存储从绑定到解绑的所有glVertexAttribPointer配置，也可以存储 EBO
	// VAO和VBO是一对多的关系，多个buffer可以使用一种解释方式。





	unsigned int VBO, VAO;
	
	glGenVertexArrays(1, &VAO);
	glGenBuffers(1, &VBO);
	// 从glBindVertexArray（VAO）绑定VAO开始，所有 配置相关的内容都被保存
	glBindVertexArray(VAO);

	glBindBuffer(GL_ARRAY_BUFFER, VBO);
	glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
	//设置顶点属性（location，维数，每一维数据类型，是否开启归一化，字段偏移）
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
	//启用该顶点的属性（opengl属性默认是禁用的，必须调函数启用），
	//参数是当前属性的location（layout(lacation =0)）
	glEnableVertexAttribArray(0);

	//解绑当前的VBO
	glBindBuffer(GL_ARRAY_BUFFER, 0);

	// 解绑VAO （配置录制结束）
	glBindVertexArray(0);

	  // openGl 默认填充绘图，可以调用glPolygonMode 修改为GL_LINE线段绘图模式
	//glPolygonMode(GL_FRONT_AND_BACK, GL_LINE);

	//渲染循环
	while (!glfwWindowShouldClose(window))
	{

		processInput(window);

		// 
		// 每一次循环，用颜色清理buffer缓存
		glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
		glClear(GL_COLOR_BUFFER_BIT);

		//  之前glLinkProgram 编译好了shader，现在 gluseprogram 调用
		
		glUseProgram(shaderProgram);
		//面向VAO的绘图（VAO甚至包括绑定特定的VBO,EBO与解绑）
		glBindVertexArray(VAO); 
		glDrawArrays(GL_TRIANGLES, 0, 3);
		// glBindVertexArray(0);

	// 同之前，GLFW的窗口绘制采取双缓冲模式，避免闪烁
		glfwSwapBuffers(window);
		glfwPollEvents();
	}

      //   释放  VAO VBO program所有的资源
	glDeleteVertexArrays(1, &VAO);
	glDeleteBuffers(1, &VBO);
	glDeleteProgram(shaderProgram);


	glfwTerminate();
	return 0;
}

//后面是之前封装的退出函数和  改变窗口大小（帧缓冲）的回调函数绑定
void processInput(GLFWwindow* window)
{
	if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
		glfwSetWindowShouldClose(window, true);
}


void framebuffer_size_callback(GLFWwindow* window, int width, int height)
{

	glViewport(0, 0, width, height);
}





```


## 一些重要的概念

### 渲染管线
![[Pasted image 20221202225513.png]]


图形渲染管线的第一个部分是==顶点着色器(Vertex Shader)==，它把一个单独的顶点作为输入。顶点着色器主要的目的是把3D坐标转为另一种3D坐标（后面会解释），同时顶点着色器允许我们对顶点属性进行一些基本处理，比如==顶点着色器输入可以是顶点坐标，颜色，其他属性==

==图元装配(Primitive Assembly)阶段==将顶点着色器输出的所有顶点作为输入（如果是GL_POINTS，那么就是一个顶点），并所有的点装配成指定图元的形状；本节例子中是一个三角形。

图元装配阶段的输出会传递给==几何着色器(Geometry Shader)==。几何着色器把图元形式的一系列顶点的集合作为输入，它可以==通过产生新顶点构造出新的（或是其它的）图元来生成其他形状==。（产生新的顶点或图元）

几何着色器的输出会被传入==光栅化阶段(Rasterization Stage)==，这里它会把==图元映射为最终屏幕上相应的像素==，生成供==片段着色器(Fragment Shader)使用的片段(Fragment)==。在片段着色器运行之前会执行裁切(Clipping)。裁切会丢弃超出你的视图以外的所有像素，用来提升执行效率。（==片段其实就是指一个像素所包含的所有信息==）

==片段着色器的主要目的是计算一个像素的最终颜色==，这也是所有OpenGL高级效果产生的地方。通常，片段着色器包含3D场景的数据（比如光照、阴影、光的颜色等等），这些数据可以被用来计算最终像素的颜色。

在所有对应颜色值确定以后，最终的对象将会被传到最后一个阶段，我们叫做==Alpha测试和混合(Blending)==阶段。这个阶段检测片段的对应的深度（和模板(Stencil)）值（后面会讲），用它们来==判断这个像素是其它物体的前面还是后面==，决定是否应该丢弃。这个阶段也会==检查alpha值（alpha值定义了一个物体的透明度）并对物体进行混合(Blend)==。所以，即使在片段着色器中计算出来了一个像素输出的颜色，在渲染多个三角形的时候最后的像素颜色也可能完全不同。


### VBO
通过==顶点缓冲对象(Vertex Buffer Objects, VBO)==管理顶点内存，它会在GPU内存（通常被称为显存）中储存大量顶点。使用这些缓冲对象的好处是我们可以一次性的发送一大批数据到显卡上，而不是每个顶点发送一次。从CPU把数据发送到显卡相对较慢，所以只要可能我们都要尝试尽量一次性发送尽可能多的数据。当数据发送至显卡的内存中后，顶点着色器几乎能立即访问顶点，这是个非常快的过程。

#### 使用技巧：

`glBindBuffer(GL_ARRAY_BUFFER, VBO);`
指定定义的VBO是GL_ARRAY_BUFFER属性，之后所有对GL_ARRAY_BUFFER的操作，都是对VBO的操作。
`glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);`
向GL_ARRAY_BUFFER(VBO对象)中传入顶点数据

==opengl必须至少要设置顶点着色器和片段着色器后才会工作（其他渲染管线可以默认）==

### 顶点着色器与片段着色器的链接使用

==暂时通过硬编码的方式使用shader代码==

现在，我们暂时将顶点着色器的源代码硬编码在代码文件顶部的C风格字符串中：

```cpp
const char *vertexShaderSource = "#version 330 core\n"
    "layout (location = 0) in vec3 aPos;\n"
    "void main()\n"
    "{\n"
    "   gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
    "}\0";
```


用glCreateShade创建 指定类型的着色器（这里是GL_VERTEX_SHADER）：

```cpp
unsigned int vertexShader;
vertexShader = glCreateShader(GL_VERTEX_SHADER);
```



使用glShaderSource()附加源代码到 创建的shader中，glCompileShader编译。

```cpp
glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
glCompileShader(vertexShader);
```

glShaderSource函数把要编译的着色器对象作为第一个参数。第二参数指定了传递的源码字符串数量，这里只有一个。第三个参数是顶点着色器真正的源码，第四个参数我们先设置为`NULL`。

片段着色器同理（创建着色器，附加源代码，编译着色器）
```cpp
#version 330 core
out vec4 FragColor;

void main()
{
    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);
} 
```



==定义并编译完着色器对象后，需要用着色器程序对象合并和链接==

==着色器程序对象(Shader Program Object)==是==多个着色器合并之后并最终链接完成的版本==。如果要使用刚才编译的着色器我们必须把它们链接(Link)为一个着色器程序对象，然后在渲染对象的时候激活这个着色器程序。已激活着色器程序的着色器将在我们发送渲染调用的时候被使用。

创建一个程序对象

```cpp
unsigned int shaderProgram;
shaderProgram = glCreateProgram();
```

用glLinkProgram链接它们：

```
glAttachShader(shaderProgram, vertexShader);
glAttachShader(shaderProgram, fragmentShader);
glLinkProgram(shaderProgram);
```

### 顶点属性与VAO

我们希望顶点缓冲数据会被解析为下面这样子：

![](https://learnopengl-cn.github.io/img/01/04/vertex_attribute_pointer.png)

-   位置数据被储存为32位（4字节）浮点值。
-   每个位置包含3个这样的值。
-   在这3个值之间没有空隙（或其他值）。这几个值在数组中紧密排列(Tightly Packed)。
-   数据中第一个值在缓冲开始的位置。

有了这些信息我们就可以使用glVertexAttribPointer函数告诉OpenGL该如何解析顶点数据（应用到逐个顶点属性上）了：

```
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
```

==glVertexAttribPointer用于配合VBO，告诉GPU如何划分buffer中的数据。==


但是每一个VBO，我们都要写glVertexAttribPointer和EBO等配置，很不方便。可以使用 ==顶点数组对象(Vertex Array Object, VAO)== 

**注： OpenGL的核心模式要求必须使用VAO，所以它知道该如何处理我们的顶点输入。如果我们绑定VAO失败，OpenGL会拒绝绘制任何东西**。

顶点数组对象**VAO**从创建开始，就会储存以下这些内容：
（VAO存储配置，相关连的VBO,EBO等）
==VAO能记录很多函数的调用和VBO,EBO==
-   glEnableVertexAttribArray和glDisableVertexAttribArray的调用。
-   通过glVertexAttribPointer设置的顶点属性配置。
-   通过glVertexAttribPointer调用与顶点属性关联的VBO。
- glbindbuffer 调用
-   还有EBO

==一般当你打算绘制多个物体时，你首先要生成/配置VAO1,2,3（和对应的VBO及属性指针)，然后储存它们供后面使用。当我们打算绘制物体的时候就拿出相应的VAO，绑定它，绘制完物体后，再解绑VAO。==

**现代opengl是以VAO为单位的配置管理**

```cpp
// 4. 绘制物体 
glUseProgram(shaderProgram); 
glBindVertexArray(VAO1，2，3 ，4);//绑定了VAO就绑定了对应的VBO，EBO等一系列函数操作
someOpenGLFunctionThatDrawsOurTriangle();
```


### 元素缓冲对象 EBO
在渲染顶点这一话题上我们还有最后一个需要讨论的东西——元素缓冲对象(Element Buffer Object，EBO)，也叫索引缓冲对象(Index Buffer Object，IBO)

**EBO一样可以保存在VAO中作为配置**


 EBO是一个缓冲区，就像一个顶点缓冲区对象一样，它存储 OpenGL 用来决定要绘制哪些顶点的索引。这种所谓的索引绘制(Indexed Drawing)正是我们问题的解决方案。首先，我们先要定义（不重复的）顶点，和绘制出矩形所需的索引：

```cpp
float vertices[] = {
    0.5f, 0.5f, 0.0f,   // 右上角
    0.5f, -0.5f, 0.0f,  // 右下角
    -0.5f, -0.5f, 0.0f, // 左下角
    -0.5f, 0.5f, 0.0f   // 左上角
};

unsigned int indices[] = {
    // 注意索引从0开始! 
    // 此例的索引(0,1,2,3)就是顶点数组vertices的下标，
    // 这样可以由下标代表顶点组合成矩形

    0, 1, 3, // 第一个三角形
    1, 2, 3  // 第二个三角形
};
```

与VBO类似，我们先绑定EBO到GL_ELEMENT_ARRAY_BUFFER,然后用glBufferData把索引复制到缓冲里。


```cpp
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
//结构类似，向EBO中存入索引内容 indices
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW)
```

使用时，注意**用glDrawElements来替换glDrawArrays函数，表示我们要从索引缓冲区渲染三角形** 

```cpp
//同VBO,绑定ID到指定的BUFFER ,在 调用特定的draw函数，从索引区绘制三角形

glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
//参数为 索引大小，数据类型 
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
// 解绑
glBindVertexArray(0);
```


### 关于VAO,VBO,EBO的使用注意
**glDrawElements函数**从当前绑定到GL_ELEMENT_ARRAY_BUFFER目标的EBO中获取其索引。这意味着我们每次想要使用索引index 渲染对象时 都必须绑定相应的EBO，这种麻烦类似于VBO的麻烦 。==所以VAO也支持记录EBO的操作。==

![](https://learnopengl-cn.github.io/img/01/04/vertex_array_objects_ebo.png)

==注意：当目标是**GL_ELEMENT_ARRAY_BUFFER**的时候，VAO会储存glBindBuffer的函数调用。这也意味着它也会储存解绑调用，所以确保你没有在解绑VAO之前解绑索引数组缓冲，否则它就没有这个EBO配置了。==（**说人话，就是EBO的绑定和解绑都会被VAO存储，一定要保证VAO最先解绑，否则会记录不该记录的东西**）

## 练习1

1.添加更多顶点到数据中，使用glDrawArrays，尝试绘制两个彼此相连的三角形。

解：glDrawArray以VAO为单位画出对应绑定的VBOs中的顶点。如果图元是triangle的话，顺序就是每三个一组（对应的属性指针也是这么设置的）。

```cpp

float vertices[] = { 
// first triangle 
-0.9f, -0.5f, 0.0f, // left
-0.0f, -0.5f, 0.0f, // right 
-0.45f, 0.5f, 0.0f, // top 
// second triangle 
0.0f, -0.5f, 0.0f, // left 
0.9f, -0.5f, 0.0f, // right 
0.45f, 0.5f, 0.0f // top
};


glDrawArray(GL_TRIANGLES, 0, 6); //图元 ，偏移 ，总点数


```

## 练习2
2.创建相同的两个三角形，但对它们的数据使用不同的VAO和VBO：
解答：其实考察就是面向于VAO的绘图。


```cpp
	float firstTriangle[] = {
		-0.5f, -0.5f, 0.0f, // left  
		 0.5f, -0.5f, 0.0f, // right 
		 0.0f,  0.5f, 0.0f  // top   
	};
	float secondTriangle[] = {
	 0.0f, -0.5f, 0.0f, // left 
	 0.9f, -0.5f, 0.0f, // right 
	 0.45f, 0.5f, 0.0f // top 
	 };
	 
	
//VAO绑定
	unsigned int VBO[2], VAO[2];
	glGenVertexArrays(2,&VAO);
	glGenBuffers(2,&VBO);
	//记录第一个VAO操作集
	glBindVertexArray(VAO[0]);
	glBindBuffer(GL_ARRAY_BUFFER, &VBO[0]);
	glBufferData(GL_ARRAY_BUFFER,sizeof(firstTriangle), firstTriangle,GL_STATIC_DRAW);
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
	glEnableVertexAttribArray(0);

	//注意：绑定第二个VAO自动视为解绑第一个VAO

	glBindVertexArray(VAO[1]);
	glBindBuffer(GL_ARRAY_BUFFER, &VBO[1]);
	glBufferData(GL_ARRAY_BUFFER, sizeof(secondTriangle), secondTriangle, GL_STATIC_DRAW);
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
	glEnableVertexAttribArray(0);


........

//面向VAO绘图
glBindVertexArray(VAOs[0]); 
glDrawArrays(GL_TRIANGLES, 0, 3); 
// 绑定VAO2
glBindVertexArray(VAOs[1]);
glDrawArrays(GL_TRIANGLES, 0, 3)





```

## 练习3
3.创建两个着色器程序，第二个程序使用一个不同的片段着色器，输出黄色；再次绘制这两个三角形，让其中一个输出为黄色
解答：顶点着色器一样，不过需要创建不同的片段着色器，然后创建不同的着色器程序来编译链接对应的片段着色器。  同时，==因为两个三角形连片段着色器都不同了，最好是使用不同的VAO将流程分开（非强制）==

```cpp
//关键代码
unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
//两个不同的片段着色器
unsigned int fragmentShaderOrange = glCreateShader(GL_FRAGMENT_SHADER); 
unsigned int  fragmentShaderYellow = glCreateShader(GL_FRAGMENT_SHADER); 
shaderProgramOrange = glCreateProgram(); 
unsigned int shaderProgramYellow = glCreateProgram(); 

glShaderSource(vertexShader, 1, &vertexShaderSource, NULL); glCompileShader(vertexShader); 
//分别编译片段着色器
glShaderSource(fragmentShaderOrange, 1, &fragmentShader1Source, NULL); glCompileShader(fragmentShaderOrange); glShaderSource(fragmentShaderYellow, 1, &fragmentShader2Source, NULL); glCompileShader(fragmentShaderYellow); 
//将顶点着色器和片段着色器1,2分别绑定于着色器程序1,2.编译并链接
glAttachShader(shaderProgramOrange, vertexShader); glAttachShader(shaderProgramOrange, fragmentShaderOrange); glLinkProgram(shaderProgramOrange);

glAttachShader(shaderProgramYellow, vertexShader); glAttachShader(shaderProgramYellow, fragmentShaderYellow); glLinkProgram(shaderProgramYellow); 

//VAOs保存VBO及属性指针配置
glBindVertexArray(VAOs[0]); 
glBindBuffer(GL_ARRAY_BUFFER, VBOs[0]);
glBufferData(GL_ARRAY_BUFFER, sizeof(firstTriangle), firstTriangle, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0); 
glEnableVertexAttribArray(0); 

glBindVertexArray(0); 
//绑定下一个VAO时，自动解绑上一个VAO
glBindVertexArray(VAOs[1]); 
glBindBuffer(GL_ARRAY_BUFFER, VBOs[1]);
glBufferData(GL_ARRAY_BUFFER, sizeof(secondTriangle), secondTriangle, GL_STATIC_DRAW); 
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, (void*)0); 
glEnableVertexAttribArray(0);
//解绑第二个VAO
glBindVertexArray(0); 
while (!glfwWindowShouldClose(window)) { 

glUseProgram(shaderProgramOrange);

glBindVertexArray(VAOs[0]); 
glDrawArrays(GL_TRIANGLES, 0, 3); 

glUseProgram(shaderProgramYellow); 
glBindVertexArray(VAOs[1]);
glDrawArrays(GL_TRIANGLES, 0, 3); 


}



```

# 示例3 着色器
## 代码

代码改动很少，就是先在片段着色器中定义好uniform，再激活程序，绘图前利用uniform设置片段着色器中的颜色。

```cpp

#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <iostream>
#include <cmath>
void framebuffer_size_callback(GLFWwindow* window, int width, int height); void processInput(GLFWwindow *window); 
const unsigned int SCR_WIDTH = 800;
const unsigned int SCR_HEIGHT = 600; 
const char *vertexShaderSource ="#version 330 core\n" "layout (location = 0) in vec3 aPos;\n" "void main()\n" "{\n" " gl_Position = vec4(aPos, 1.0);\n" "}\0"; 

//片段着色器代码中定义了一个uniform
const char *fragmentShaderSource = "#version 330 core\n" "out vec4 FragColor;\n" "uniform vec4 ourColor;\n" "void main()\n" "{\n" " FragColor = ourColor;\n" "}\n\0"; 




int main() {

glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3); glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3); glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE); 
#ifdef __APPLE__ 
glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE); 
#endif
GLFWwindow* window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "LearnOpenGL", NULL, NULL);
if (window == NULL) 
{ std::cout << "Failed to create GLFW window" << std::endl; glfwTerminate(); 
 return -1; } 
 glfwMakeContextCurrent(window);
  glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);  
  if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)) 
  { std::cout << "Failed to initialize GLAD" << std::endl;
   return -1; 
   }

    unsigned int vertexShader =glCreateShader(GL_VERTEX_SHADER); glShaderSource(vertexShader, 1, &vertexShaderSource, NULL); glCompileShader(vertexShader); 
    // check for shader compile errors int success; 
    char infoLog[512];
     glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success); 
     if (!success) { 
     glGetShaderInfoLog(vertexShader, 512, NULL, infoLog); 
     std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl; 
     } 
     // fragment shader 
     unsigned int fragmentShader = glCreateShader(GL_FRAGMENT_SHADER); glShaderSource(fragmentShader, 1, &fragmentShaderSource, NULL); glCompileShader(fragmentShader); 
     // check for shader compile errors
      glGetShaderiv(fragmentShader, GL_COMPILE_STATUS, &success); 
      if (!success) { 
      glGetShaderInfoLog(fragmentShader, 512, NULL, infoLog); 
      std::cout << "ERROR::SHADER::FRAGMENT::COMPILATION_FAILED\n" << infoLog << std::endl; }
      
       // 同之前，创建着色器程序，附加着色器，同时链接程序
        unsigned int shaderProgram = glCreateProgram(); glAttachShader(shaderProgram, vertexShader);   glAttachShader(shaderProgram, fragmentShader); glLinkProgram(shaderProgram);
          // check for linking errors 
          glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
           if (!success) { 
           glGetProgramInfoLog(shaderProgram, 512, NULL, infoLog); std::cout << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n" << infoLog << std::endl; 
           } 
           glDeleteShader(vertexShader);
            glDeleteShader(fragmentShader);



             float vertices[] = { 
             0.5f, -0.5f, 0.0f, // bottom right 
             -0.5f, -0.5f, 0.0f, // bottom left 
             0.0f, 0.5f, 0.0f // top 
             }; 
             unsigned int VBO, VAO;
              glGenVertexArrays(1, &VAO);
               glGenBuffers(1, &VBO);
               glBindVertexArray(VAO); 
    glBindBuffer(GL_ARRAY_BUFFER, VBO); 
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
     glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0); 
     glEnableVertexAttribArray(0); 
     //结束保存VAO后，解绑VAO是一个好习惯
     glBindVertexArray(0);



//因为整个循化只用一种VAO，所以不需要解绑，可以放在循环外
      glBindVertexArray(VAO);
       while (!glfwWindowShouldClose(window)) {
        // input // ----- 
        processInput(window); 
        glClearColor(0.2f, 0.3f, 0.3f, 1.0f); glClear(GL_COLOR_BUFFER_BIT); 
    
        // 注意：在更新uniform之前，一定要激活着色器程序（查询uniform的位置不需要，但是改变uniform的值一定要激活着色器程序）
        glUseProgram(shaderProgram);
        timeValue = glfwGetTime();
         float greenValue = static_cast<float>(sin(timeValue) / 2.0 + 0.5); 
         //着色器程序是运行在GPU上的程序，改变uniform的值需要知道uniform在程序中的位置。
         int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor"); 
         glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f); 
          glDrawArrays(GL_TRIANGLES, 0, 3); 



          glfwSwapBuffers(window);
           glfwPollEvents(); } 

           glDeleteVertexArrays(1, &VAO);
            glDeleteBuffers(1, &VBO);
             glDeleteProgram(shaderProgram);

              glfwTerminate(); 
              return 0; } 

 void processInput(GLFWwindow *window) { 
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS) glfwSetWindowShouldClose(window, true); 
    } 

     void framebuffer_size_callback(GLFWwindow* window, int width, int height) { 

     glViewport(0, 0, width, height); }

```

## 一些重要的概念

### GLSL
==着色器(Shader)==是==运行在GPU上的小程序==。这些小程序为图形渲染管线的某个特定部分而运行。从基本意义上来说，着色器只是一种把输入转化为输出的程序。**着色器也是一种非常独立的程序，因为它们之间不能相互通信；它们之间唯一的沟通只有通过输入和输出**

着色器是使用一种叫**GLSL的类C语言**写成的。GLSL是为图形计算量身定制的，它包含一些针对向量和矩阵操作的有用特性。

==着色器的开头总是要声明版本，接着是输入和输出变量、uniform和main函数==

### 顶点属性与上限
顶点shader中，每个输入变量也叫**顶点属性(Vertex Attribute)。我们能声明的顶点属性是有上限的，它一般由硬件来决定**。OpenGL确保至少有16个包含4分量的顶点属性可用，但是有些硬件或许允许更多的顶点属性，你可以查询**GL_MAX_VERTEX_ATTRIBS**来获取具体的上限：

```cpp
int nrAttributes;
glGetIntegerv(GL_MAX_VERTEX_ATTRIBS, &nrAttributes);
std::cout << "Maximum nr of vertex attributes supported: " << nrAttributes << std::endl;
```

通常情况下它至少会返回16个，大部分情况下是够用了。

### opengl利用命名实现类似于函数重载


==由于opengl基于C语言实现，没有C++的重载语法，所以使用相似的命名代替重载，比如：==

**glUniform是一个典型例子。这个函数有一个特定的后缀，标识设定的uniform的类型。可能的后缀有：**


`f` 函数需要一个float作为它的值

`i` 函数需要一个int作为它的值

`ui` 函数需要一个unsigned int作为它的值

`3f` 函数需要3个float作为它的值

`fv` 函数需要一个float向量/数组作为它的值

每当你打算配置一个OpenGL的选项时就可以简单地根据这些规则选择适合你的数据类型的重载函数。




GLSL中包含C等其它语言大部分的默认基础数据类型：`int`、`float`、`double`、`uint`和`bool`。

#### opengl中向量的定义

`vecn`  包含`n`个float分量的默认向量

`bvecn` 包含`n`个bool分量的向量

`ivecn` 包含`n`个int分量的向量

`uvecn` 包含`n`个unsigned int分量的向量

`dvecn` 包含`n`个double分量的向量

大多数时候我们使用`vecn`，因为float足够满足大多数要求了。


#### 向量支持的一些小语法

##### 向量的初始化构造
==我们也可以把一个向量作为一个参数传给不同的向量构造函数==，以减少需求参数的数量：

```cpp
vec2 vect = vec2(0.5, 0.7);
vec4 result = vec4(vect, 0.0, 0.0);
//向量可以直接作为参数传给更高维的向量的构造函数
vec4 otherResult = vec4(result.xyz, 1.0);
```

#### 向量的维度交换
向量这一数据类型也允许一些有趣而灵活的分量选择方式，叫做==重组(Swizzling)==。==用x,y,z，w代指向量的各个维度：==

```cpp
vec2 someVec;
vec4 differentVec = someVec.xyxx;
vec3 anotherVec = differentVec.zyw;
vec4 otherVec = someVec.xxxx + anotherVec.yxzy;
```


### 输入与输出
GLSL定义了`in`和`out`关键字专门来实现这个目的。每个着色器使用这两个关键字设定输入和输出，只要一个输出变量与下一个着色器阶段的输入匹配，它就会传递下去。

#### 特例
但是有两个特例：
**1.顶点着色器的输入不能随意的定义为 in （会影响顶点着色器读取顶点数据的效率）。需要加布局限定符 （layout lacation =0）**

**2. 片段着色器必须定义一个vec4的输出作为最终颜色，不然opengl会使用默认的黑色或白色。**


### uniform 

==Uniform是一种从CPU中的应用向GPU中的着色器发送数据的方式,可以把它理解为一种全局变量，在cpu的shader代码中定义好uniform后，可以在GPU运行的任何时候，或渲染管线的任何位置使用该变量。==


uniform的使用示例，随着时间改变颜色：

```
float timeValue = glfwGetTime();
float greenValue = (sin(timeValue) / 2.0f) + 0.5f;
int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
glUseProgram(shaderProgram);
glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);
```

#### uniform使用注意：
1.改变uniform的值，必须要知道该uniform变量在着色器程序的位置（使用**glGetUniformLocation**）
2.查询uniform的位置时，可以不激活着色器程序，但是改变uniform的之前必须激活着色器程序（**glUseProgram**）


### 使用着色器程序类简化流程

利用一个着色器程序类，类中包含 功能：
1.    读取并编译着色器代码（从本地读取shader代码的txt文档，ifstream转为string，和const string，创建着色器，编译）
2.   使用着色器程序（传入程序ID,绑定类中的片段着色器和顶点着色器，激活程序）
3.   支持修改shader中的uniform值（完成 查询uniform位置+设置uniform值的操作）


比较简单，代码见工程。

```cpp

// shader_s类
//预处理指令(Preprocessor Directives)。
// //这些预处理指令会告知你的编译器只在它没被包含过的情况下才包含和编译这个头文件，
//即使多个文件都包含了这个着色器头文件。它是用来防止链接冲突的。
#ifndef SHADER_H
#define SHADER_H

#include <glad/glad.h>

#include <string>
#include <fstream>
#include <sstream>
#include <iostream>

class Shader
{
public:
	unsigned int ID;
	
	Shader(const char* vertexPath, const char* fragmentPath)
	{
		// C++标准io流的读取文件，转为string
		std::string vertexCode;
		std::string fragmentCode;
		std::ifstream vShaderFile;
		std::ifstream fShaderFile;
		//exceptions（std::ifstream::failbit） 流读取失败的异常抛出
		vShaderFile.exceptions(std::ifstream::failbit | std::ifstream::badbit);
		fShaderFile.exceptions(std::ifstream::failbit | std::ifstream::badbit);

		//包含异常抛出的文件处理操作
		try
		{
			// 
			vShaderFile.open(vertexPath);
			fShaderFile.open(fragmentPath);
			// 打开文件，将读取内容<<入 字符流。
			std::stringstream vShaderStream, fShaderStream;
			//
			vShaderStream << vShaderFile.rdbuf();
			fShaderStream << fShaderFile.rdbuf();
			// 读完，关闭文件
			vShaderFile.close();
			fShaderFile.close();
			// 将字符流保存于string中
			
			vertexCode = vShaderStream.str();
			fragmentCode = fShaderStream.str();
		}
		//上述只要发生异常，则抛出
		catch (std::ifstream::failure& e)
		{
			std::cout << "ERROR::SHADER::FILE_NOT_SUCCESFULLY_READ: " << e.what() << std::endl;
		}
		//string转为 const 
		const char* vShaderCode = vertexCode.c_str();
		const char* fShaderCode = fragmentCode.c_str();
		// 编译各类shader
		unsigned int vertex, fragment;
		vertex = glCreateShader(GL_VERTEX_SHADER);
		glShaderSource(vertex, 1, &vShaderCode, NULL);
		glCompileShader(vertex);
		checkCompileErrors(vertex, "VERTEX");
		fragment = glCreateShader(GL_FRAGMENT_SHADER);
		glShaderSource(fragment, 1, &fShaderCode, NULL);
		glCompileShader(fragment);
		checkCompileErrors(fragment, "FRAGMENT");
		// 创建shaderprogram，附加shader，并编译
		ID = glCreateProgram();
		glAttachShader(ID, vertex);
		glAttachShader(ID, fragment);
		glLinkProgram(ID);
		checkCompileErrors(ID, "PROGRAM");
		//资源回收
		glDeleteShader(vertex);
		glDeleteShader(fragment);
	}
	//激活程序
	void use()
	{
		glUseProgram(ID);
	}
	//修改 uniform
	void setBool(const std::string& name, bool value) const
	{
		glUniform1i(glGetUniformLocation(ID, name.c_str()), (int)value);
	}

	void setInt(const std::string& name, int value) const
	{
		glUniform1i(glGetUniformLocation(ID, name.c_str()), value);
	}

	void setFloat(const std::string& name, float value) const
	{
		glUniform1f(glGetUniformLocation(ID, name.c_str()), value);
	}

private:
	//类支持传入shader ID , 类型，检查shader或 program 的编译错误
	void checkCompileErrors(unsigned int shader, std::string type)
	{
		int success;
		char infoLog[1024];
		if (type != "PROGRAM")
		{
			glGetShaderiv(shader, GL_COMPILE_STATUS, &success);
			if (!success)
			{
				glGetShaderInfoLog(shader, 1024, NULL, infoLog);
				std::cout << "ERROR::SHADER_COMPILATION_ERROR of type: " << type << "\n" << infoLog << "\n -- --------------------------------------------------- -- " << std::endl;
			}
		}
		else
		{
			glGetProgramiv(shader, GL_LINK_STATUS, &success);
			if (!success)
			{
				glGetProgramInfoLog(shader, 1024, NULL, infoLog);
				std::cout << "ERROR::PROGRAM_LINKING_ERROR of type: " << type << "\n" << infoLog << "\n -- --------------------------------------------------- -- " << std::endl;
			}
		}
	}
};
#endif

```



### 练习
练习使用uniform和in,out，比较简单，省略。


# 纹理


## 代码：


==main.cpp==

**加入了纹理对象的生成与绑定，多级纹理生成（mipmap）,以及纹理单元绑定纹理对象**

```cpp


#include <glad/glad.h>
#include <GLFW/glfw3.h>
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
#include "shader_s.h"

#include <iostream>
#include <filesystem>
using namespace std;
void framebuffer_size_callback(GLFWwindow* window, int width, int height);
void processInput(GLFWwindow* window);


const unsigned int SCR_WIDTH = 800;
const unsigned int SCR_HEIGHT = 600;

int main()
{
	
	
	glfwInit();
	glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
	glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
	glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

#ifdef __APPLE__
	glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif


	GLFWwindow* window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "LearnOpenGL", NULL, NULL);
	if (window == NULL)
	{
		std::cout << "Failed to create GLFW window" << std::endl;
		glfwTerminate();
		return -1;
	}
	glfwMakeContextCurrent(window);
	glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);


	if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
	{
		std::cout << "Failed to initialize GLAD" << std::endl;
		return -1;
	}

	
	//之前写好的shader类，功能见上节
	Shader ourShader("4.2.texture.vs", "4.2.texture.fs");

	
	float vertices[] = {
		// positions          // colors           // 纹理坐标
		//注：纹理坐标与实际纹理无关，可以看做顶点取纹理图片的比例
		 0.5f,  0.5f, 0.0f,   1.0f, 0.0f, 0.0f,   1.0f, 1.0f, // top right
		 0.5f, -0.5f, 0.0f,   0.0f, 1.0f, 0.0f,   1.0f, 0.0f, // bottom right
		-0.5f, -0.5f, 0.0f,   0.0f, 0.0f, 1.0f,   0.0f, 0.0f, // bottom left
		-0.5f,  0.5f, 0.0f,   1.0f, 1.0f, 0.0f,   0.0f, 1.0f  // top left 
	};
	//用于初始化EBO的索引
	unsigned int indices[] = {
		0, 1, 3, 
		1, 2, 3  
	};
	unsigned int VBO, VAO, EBO;
	glGenVertexArrays(1, &VAO);
	glGenBuffers(1, &VBO);
	glGenBuffers(1, &EBO);

	glBindVertexArray(VAO);
	//VBO与EBO赋值
	glBindBuffer(GL_ARRAY_BUFFER, VBO);
	glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

	glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
	glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

	// 顶点属性指针配置
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)0);
	//
	glEnableVertexAttribArray(0);
	
	glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(3 * sizeof(float)));
	glEnableVertexAttribArray(1);
	
	glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(6 * sizeof(float)));
	glEnableVertexAttribArray(2);

	
	//

	unsigned int texture1, texture2;
//纹理object用法同其他一样，都是先产生ID,再绑定管线
	glGenTextures(1, &texture1);
	glBindTexture(GL_TEXTURE_2D, texture1);
	// glTexParameteri（）是针对于渲染管线的设置
	//可以以（x,y）设置纹理的重复属性
	//可以针对于纹理被放大，或纹理被缩小的情况设置插值方式（纹理过滤）
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);	
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
	
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
	
	int width, height, nrChannels;
	//兼容OpenGL的y轴在下
	stbi_set_flip_vertically_on_load(true); 
	
	
	unsigned char* data = stbi_load("resources/textures/container.jpg", &width, &height, &nrChannels, 0);
	if (data)
	{
		//（纹理种类，多级纹理的级别，希望纹理被存储的格式，期望的宽，期望的高，默认0，读取原图的格式，读取原图的数据类型，内容）
		glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
		//生成多级纹理（属于纹理被缩小的情况）
		//纹理过大和纹理过小都会有走样，原则上我们应该使用尽可能大的纹理
		//因为mipmap通过生成多级缩小纹理，使得纹理与物体始终处于对等关系。
		glGenerateMipmap(GL_TEXTURE_2D);
	}
	else
	{
		std::cout << "Failed to load texture" << std::endl;
	}
	//释放纹理字符存储的内存
	stbi_image_free(data);
	// 另一个纹理

	glGenTextures(1, &texture2);
	glBindTexture(GL_TEXTURE_2D, texture2);
	//老四样
	//纹理本身的重复
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);	
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
	//纹理的采样方式和放大缩小情况
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
	
	//同上，读取纹理2
	data = stbi_load("resources/textures/awesomeface.png", &width, &height, &nrChannels, 0);
	if (data)
	{
		
		glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);
		glGenerateMipmap(GL_TEXTURE_2D);
	}
	else
	{
		std::cout << "Failed to load texture" << std::endl;
	}
	stbi_image_free(data);

	//改变采样器涉及到修改uniform，注意激活着色器程序

	ourShader.use(); 
	//因为一个片段着色器中定义了两个纹理，所以在实际渲染之前，我们需要告诉GPU每个纹理对应的位置，
	// //即纹理单元
	//1.绑定两个纹理到对应的纹理单元，2.然后定义哪个uniform采样器对应哪个纹理单元

	//首先，修改uniform采样器的值，说明哪一个采样器属于哪一个纹理单元
	glUniform1i(glGetUniformLocation(ourShader.ID, "texture1"), 0);
	
	ourShader.setInt("texture2", 1);



	
	while (!glfwWindowShouldClose(window))
	{
	
		processInput(window);

	
		glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
		glClear(GL_COLOR_BUFFER_BIT);

		// 其次，glActiveTexture激活对应的纹理单元，将纹理绑定到对应的纹理单元

		//如果只绑定了纹理到对应的纹理单元，没有处理采样器，
		//虽然GPU中有纹理，但是采样器无法根据纹理坐标取出纹理，所以最终还是没有纹理
		//绑定纹理到对应的纹理单元时，opengl默认单元0
		glActiveTexture(GL_TEXTURE0);//可以省略
		glBindTexture(GL_TEXTURE_2D, texture1);
		glActiveTexture(GL_TEXTURE1);
		glBindTexture(GL_TEXTURE_2D, texture2);

		
		ourShader.use();
		glBindVertexArray(VAO);
		glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

		
		glfwSwapBuffers(window);
		glfwPollEvents();
	}


	glDeleteVertexArrays(1, &VAO);
	glDeleteBuffers(1, &VBO);
	glDeleteBuffers(1, &EBO);

	
	glfwTerminate();
	return 0;
}


void processInput(GLFWwindow* window)
{
	if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
		glfwSetWindowShouldClose(window, true);
}


void framebuffer_size_callback(GLFWwindow* window, int width, int height)
{
	
	glViewport(0, 0, width, height);
}


```


==片段着色器==

**定义uniform类型的纹理采样器，用于在后续运行的代码中根据纹理单元（意为位置），读入不同的纹理，再使用texture（）函数根据顶点坐标采样出具体的像素颜色**
### 关于texture函数

GLSL内建的texture函数来采样纹理的颜色，它第一个参数是纹理采样器，第二个参数是对应的纹理坐标。texture函数是**状态使用函数，会使用** **之前设置的纹理参数**对相应的颜色值进行采样。（代码中由**状态设置函数，glTexParameteri**设置）

//==传入的是顶点对应的纹理坐标，是间断的，但是texture函数输出颜色是连续的，因为顶点之间的点的纹理坐标通过重心坐标插值得到，然后用采样器对纹理采样==
```cpp
#version 330 core
out vec4 FragColor;

in vec3 ourColor;
in vec2 TexCoord;

// 两种纹理，对应两个采样器（注意在代码中定义采样器处理的纹理单元）
uniform sampler2D texture1;
uniform sampler2D texture2;


//卡通渲染的shader实现
void main()
{
	
	FragColor = mix(texture(texture1, TexCoord), texture(texture2, TexCoord), 0.2);

	******OPENGL渲染管线搭建*******
	program.use("VetexShader",GL_TEXTURE0,FALSE，GL_SIMPLE_MSAA);//打开shader的执行
	program.update("FragmentShader",GL_FRAGMENT_SIMPLE,TRUE，GL_TIME_DATE);//更新片段着色器
	    program.upgrade("model",VetexShader.reverse,FragementShader.transform);//移动MVP矩阵，赋值世界坐标系中的物体
	program.pixel("FragmentShader",GL_BUFFER_RESIZE,GL_TEXTRUE_SIMPLE_MSAA,true);//设置像素的片段着色器结构
	********shader中的卡通渲染*********
	//具体思想是逐像素的使用DRP算法，对于结果做双线性插值。
	//卡通的边缘本例使用开销比较大的描边算法
	//渐变的“软阴影”使用视锥的内外层硬阴影实现
	//卡渲的风格化像素颜色选取参考论文如下
	//Xin Chen, Di He, and Ling Pei. Bds b1i multipath channel statistical model comparison between static and dynamic scenarios. Satellite Navigation, 1(1):1–16, 2020.
	//后续计划基于PBR的流程实现卡渲，而不是现在的类卡渲方案。
	






}
```








==顶点着色器==

加入更多顶点属性输入，将顶点对应的纹理坐标读入

```cpp
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aTexCoord;

out vec3 ourColor;
out vec2 TexCoord;

void main()
{
	gl_Position = vec4(aPos, 1.0);
	ourColor = aColor;
	TexCoord = vec2(aTexCoord.x, aTexCoord.y);
}


```


## 纹理

为了能够把纹理映射(Map)到三角形上，我们需要指定三角形的每个顶点各自对应纹理的哪个部分。这样每个顶点就会关联着一个纹理坐标(Texture Coordinate)，用来标明该从纹理图像的哪个部分采样（译注：采集片段颜色）。之后在图形的其它片段上进行片段插值(Fragment Interpolation)。



### 纹理坐标的重复方式
纹理坐标的范围通常是从(0, 0)到(1, 1)，但是opengl定义了一种超范围的重复方式：



**GL_REPEAT**对纹理的默认行为。重复纹理图像。

**GL_MIRRORED_REPEAT**  与GL_REPEAT一样，但每次重复图片是镜像放置的。

**GL_CLAMP_TO_EDGE**   纹理坐标会被约束在0到1之间，超出的部分会重复纹理坐标的边缘，产生一种边缘被拉伸的效果。

**GL_CLAMP_TO_BORDER**  超出的坐标不对纹理进行采样，直接对应一种用户定义的颜色。

当纹理坐标超出默认范围时，每个选项都有不同的视觉效果输出。我们来看看这些纹理图像的例子：

![](https://learnopengl-cn.github.io/img/01/06/texture_wrapping.png)
通过**gltexParameteri()**定义S和T轴（横轴和纵轴）
```cpp
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_MIRRORED_REPEAT); glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_MIRRORED_REPEAT);
```

## 纹理过滤

纹理坐标不依赖于分辨率(Resolution)，它可以是任意浮点值，所以OpenGL需要知道怎样将纹理像素(**Texture Pixel，也叫Texel**，译注1)映射到纹理坐标，当物体像素远大于纹理像素时，此时纹理被放大/缩小，纹理坐标会取到纹理像素的间隔，此时有两种纹理过滤的方法。
**GL_NEAREST**（也叫邻近过滤，Nearest Neighbor Filtering）是OpenGL默认的纹理过滤方式。当设置为GL_NEAREST的时候，OpenGL会选择中心点最接近纹理坐标的那个像素。下图中你可以看到四个像素，加号代表纹理坐标。左上角那个纹理像素的中心距离纹理坐标最近，所以它会被选择为样本颜色：==邻近过滤有锯齿感==

![](https://learnopengl-cn.github.io/img/01/06/filter_nearest.png)

**GL_LINEAR**（也叫线性过滤，(Bi)linear Filtering）它会基于纹理坐标附近的纹理像素，计算出一个插值，近似出这些纹理像素之间的颜色。一个纹理像素的中心距离纹理坐标越近，那么这个纹理像素的颜色对最终的样本颜色的贡献越大。下图中你可以看到返回的颜色是邻近像素的混合色：==线性过滤效果好，但也会是纹理变模糊==

![](https://learnopengl-cn.github.io/img/01/06/filter_linear.png)

同样使用glTexParameter*函数为放大和缩小指定过滤方式。这段代码看起来会和纹理环绕方式的设置很相似：

```
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```



## 多级渐远纹理

纹理像素与实际像素比例失衡时，会有各种走样。 ==通常情况下，我们会给一张高分辨率的纹理，而应对纹理像素》》像素的情况，就有了多级渐远纹理（mipmap）==



==宏的前一种指选取多级纹理的level的策略，后一种值指根据纹理坐标对纹理采样的策略。==

GL_NEAREST_MIPMAP_NEAREST

使用最邻近的多级渐远纹理来匹配像素大小，并使用邻近插值进行纹理采样

GL_LINEAR_MIPMAP_NEAREST

使用最邻近的多级渐远纹理级别，并使用线性插值进行采样

GL_NEAREST_MIPMAP_LINEAR

在两个最匹配像素大小的多级渐远纹理之间进行线性插值，使用邻近插值进行采样

GL_LINEAR_MIPMAP_LINEAR

在两个邻近的多级渐远纹理之间使用线性插值，并使用线性插值进行采样

就像纹理过滤一样，我们可以使用glTexParameteri将过滤方式设置为前面四种提到的方法之一：

```cpp
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```

==一个常见的错误是，将放大过滤的选项设置为多级渐远纹理过滤选项之一。这样没有任何效果，因为多级渐远纹理主要是使用在纹理被缩小的情况下的==(mipmap中只有被压缩过的纹理)





## 纹理的使用

同其他对象，都是创建，在绑定管线。在将数据载入管线（实际上，视为载入绑定的对象）
```
unsigned int texture;
glGenTextures(1, &texture);
```

```
glBindTexture(GL_TEXTURE_2D, texture);
```

现在纹理已经绑定了，我们可以使用前面载入的图片数据生成一个纹理了。纹理可以通过glTexImage2D来生成：

```
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
glGenerateMipmap(GL_TEXTURE_2D);
```


**-   第一个参数指定了纹理目标(Target)。设置为GL_TEXTURE_2D意味着会生成与当前绑定的纹理对象在同一个目标上的纹理（任何绑定到GL_TEXTURE_1D和GL_TEXTURE_3D的纹理不会受到影响）。
**-   第二个参数为纹理指定多级渐远纹理的级别，如果你希望单独手动设置每个多级渐远纹理的级别的话。这里我们填0，也就是基本级别。
**-   第三个参数告诉OpenGL我们希望把纹理储存为何种格式。我们的图像只有`RGB`值，因此我们也把纹理储存为`RGB`值。
**-   第四个和第五个参数设置最终的纹理的宽度和高度。我们之前加载图像的时候储存了它们，所以我们使用对应的变量。
**-   下个参数应该总是被设为`0`（历史遗留的问题）。
**-   第七第八个参数定义了源图的格式和数据类型。我们使用RGB值加载这个图像，并把它们储存为`char`(byte)数组，我们将会传入对应值。
**-   最后一个参数是真正的图像数据。


存入纹理后最好清理缓存。


```
stbi_image_free(data);
```



## 纹理单元

对于一个物体使用多个纹理，我们引入纹理单元（类似于纹理的位置）。使用注意：
1. 在片段着色器中定义的采样器的值实际上对应于纹理单元。

```cpp
ourShader.use();  // 不要忘记在设置uniform变量之前激活着色器程序！ 
glUniform1i(glGetUniformLocation(ourShader.ID, "texture1"), 0); // 手动设置 
ourShader.setInt("texture2", 1); // 或者使用着色器类设置
```
2.将纹理对象绑定对应的纹理单元，好让纹理单元的纹理进入渲染管线。
```cpp
glActiveTexture(GL_TEXTURE0); 
glBindTexture(GL_TEXTURE_2D, texture1);
glActiveTexture(GL_TEXTURE1); glBindTexture(GL_TEXTURE_2D, texture2);
```




# 变换


变换知识省略，说一下leanopenglcn中用的矩阵运算库GLM 
 **OpenGL Mathematics**




```cpp
glm::mat4 trans;
trans = glm::rotate(trans, glm::radians(90.0f), glm::vec3(0.0, 0.0, 1.0));
trans = glm::scale(trans, glm::vec3(0.5, 0.5, 0.5));
```





知识不涉及opengl的渲染管线，章节内容详见GAMES101。

## 关于旋转矩阵的顺序问题


==细节：1.旋转矩阵代码是右乘（A1A2）,但是旋转变换是左乘（A2A1）,程序和变换是反的。
2.都是先旋转后平移（反应到代码是先平移后旋转），为了防止平移后，物体移动了，但是旋转轴没有移动（还在原地）。==

也可以不这么写，只需要把平移后的旋转轴也跟着平移即可


# 坐标系统



## 坐标概述



![coordinate_systems](https://learnopengl-cn.github.io/img/01/08/coordinate_systems.png)

最重要的几个分别是**模型(Model)、观察(View)、投影(Projection)三个矩阵**。我们的顶点坐标起始于**局部空间(Local Space)**，在这里它称为局部坐标(Local Coordinate)，它在之后会变为**世界坐标(World Coordinate)**，**观察坐标(View Coordinate)，裁剪坐标(Clip Coordinate)，并最后以屏幕坐标(Screen Coordinate)** 的形式结束。


==注： 视空间是以相机为原点的空间，乘以投影矩阵，成功变成了（-1,1）的方体（先截成棱台，在压缩棱台为方体），这个方体叫裁切空间。==

==裁切空间经过视口变换，送入光栅器光栅化。==


透视投影矩阵具体推导见：GAME101  #GAMES系列 #GAME101 


要创建一个正射投影矩阵，我们可以使用GLM的内置函数`glm::ortho`：

```cpp
glm::ortho(0.0f, 800.0f, 0.0f, 600.0f, 0.1f, 100.0f);
```

前两个参数指定了平截头体的左右坐标，第三和第四参数指定了平截头体的底部和顶部。通过这四个参数我们定义了近平面和远平面的大小，然后第五和第六个参数则定义了近平面和远平面的距离。这个投影矩阵会将处于这些x，y，z值范围内的坐标变换为标准化设备坐标。


## 关于深度测试

opengl中讲的很浅，并没有讲**画家算法和Z-buffer**的引入。

这一部分可以详见 #GAME101 

glEnable和glDisable函数允许我们启用或禁用某个OpenGL功能。这个功能会一直保持启用/禁用状态，直到另一个调用来禁用/启用它。

现在我们想启用深度测试，需要开启GL_DEPTH_TEST：

```
glEnable(GL_DEPTH_TEST);
```

因为我们使用了深度测试，我们也想要在每次渲染迭代之前清除深度缓冲（否则前一帧的深度信息仍然保存在缓冲中）。就像清除颜色缓冲一样，我们可以通过在glClear函数中指定DEPTH_BUFFER_BIT位来清除深度缓冲：

```
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); //每一次渲染循环清理深度和颜色buffer
```






# 相机运动


==如何在世界坐标空间下快速的经过变换，将所有的物体都放置于相机的视角中（视空间）？==

我们可以 使用**lookat 矩阵**点乘 任意的向量，将向量置于视空间中。 


## 关于视空间和相机的表示

当我们讨论摄像机/观察空间(Camera/View Space)的时候，是在讨论**以摄像机的视角作为场景原点时场景中所有的顶点坐标**：观察矩阵把所有的世界坐标变换为相对于摄像机位置与方向的观察坐标。要定义一个摄像机，我们需要它在世界空间中的**位置、观察的方向、一个指向它右侧的向量以及一个指向它上方的向量**。细心的读者可能已经注意到我们实际上创建了一个三个单位轴相互垂直的、以摄像机的位置为原点的坐标系。

![](https://learnopengl-cn.github.io/img/01/09/camera_axes.png)

### 摄像机位置

摄像机在世界坐标下的位置：


```cpp
glm::vec3 cameraPos = glm::vec3(0.0f, 0.0f, 3.0f);
```

### 摄像机方向向量（摄像机Z轴）

就是摄像机看物体的方向，因为使用的是左手系，这里的观察方向与实际方向相反，如果是右手系，则会一致。

```
glm::vec3 cameraTarget = glm::vec3(0.0f, 0.0f, 0.0f);
glm::vec3 cameraDirection = glm::normalize(cameraPos - cameraTarget);
```

==方向向量(Direction Vector)并不是最好的名字，因为它实际上指向从它到目标向量的相反方向（译注：注意看前面的那个图，蓝色的方向向量大概指向z轴的正方向，与摄像机实际指向的方向是正好相反的）。==

==方向向量实际指出了相机的Z轴，因此是左右手相反==



### 摄像机右轴（X轴 ）

为了算出摄像机的右轴，我们需要定义一个世界坐标系的**UP向量** ，与已知的Z轴叉乘得到右轴。


```cpp
glm::vec3 up = glm::vec3(0.0f, 1.0f, 0.0f); 
glm::vec3 cameraRight = glm::normalize(glm::cross(up, cameraDirection));
```




### 摄像机上轴（y轴）

利用X轴与Z轴叉乘得到。
```cpp
glm::vec3 cameraUp = glm::cross(cameraDirection, cameraRight);
```

### 补充


up向量选取的意义：因为实际场景中，相机不太可能会上下翻转（roll），up向量始终是有效的。而选取其他方向就会因为相机的侧方或左右（yaw，pitch），而使得参与叉乘计算第二个轴的向量方向相反。



## lookat 矩阵

由之前得到的X,Y，Z轴，以及相机的位置 ，就可以得到任意一个向量向摄像机坐标变换的矩阵，记为**Lookat矩阵**：

定义摄像机空间的位置坐标，我们可以创建我们自己的LookAt矩阵了：
![[Pasted image 20221216224228.png]]

其中R是右向量，U是上向量，D是方向向量P是摄像机位置向量。注意，==位置向量是相反的，因为我们最终希望把世界平移到与我们自身移动的相反方向。==




### 在GLM中创建 lookat矩阵

```cpp
glm::mat4 view;
view = glm::lookAt(glm::vec3(0.0f, 0.0f, 3.0f), 
           glm::vec3(0.0f, 0.0f, 0.0f), 
           glm::vec3(0.0f, 1.0f, 0.0f));
```

glm::LookAt函数需要一个**位置、目标和上向量**。


## 基于相机位置，与视方向的键盘移动



```cpp

view = glm::lookAt(cameraPos, cameraPos + cameraFront, cameraUp);
void processInput(GLFWwindow *window)
{
    ...
    float cameraSpeed = 0.05f; // adjust accordingly
    if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS)
        cameraPos += cameraSpeed * cameraFront;
    if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS)
        cameraPos -= cameraSpeed * cameraFront;
    if (glfwGetKey(window, GLFW_KEY_A) == GLFW_PRESS)
        cameraPos -= glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;//相机运动的切线
    if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS)
        cameraPos += glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;
}
```

当我们按下**WASD**键的任意一个，摄像机的位置都会相应更新。如果我们希望向前或向后移动，我们就把位置向量加上或减去方向向量。如果我们希望向左右移动，我们使用叉乘来创建一个**右向量**(Right Vector)，并沿着它相应移动就可以了(**这样算出来的是相机运动的实时切线**)。这样就创建了使用摄像机时熟悉的横移(Strafe)效果。

## 鼠标运动改变视野

### 关于欧拉角
欧拉角(Euler Angle)是可以表示3D空间中任何旋转的3个值，由莱昂哈德·欧拉(Leonhard Euler)在18世纪提出。一共有3种欧拉角：俯仰角(Pitch)、偏航角(Yaw)和滚转角(Roll)，下面的图片展示了它们的含义：

![](https://learnopengl-cn.github.io/img/01/09/camera_pitch_yaw_roll.png)



==对于我们的摄像机系统来说，我们只关心俯仰角和偏航角，所以我们不会讨论滚转角。==

### 欧拉角转换为相机lookat矩阵的方法（重点）

因为鼠标移动 Xoff Yoff 改变的是视野中的 欧拉角（pitch ，yaw）,所以必须找到欧拉角与lookat矩阵的关系：


```cpp
direction.x = cos(glm::radians(pitch)) * cos(glm::radians(yaw)); // 译注：direction代表摄像机的前轴(Front)，这个前轴是和本文第一幅图片的第二个摄像机的方向向量是相反的,也就是相机的真正的观察方向。
direction.y = sin(glm::radians(pitch));
direction.z = cos(glm::radians(pitch)) * sin(glm::radians(yaw));
```


==原地转动欧拉角，通过改变carmeraFront（视方向）来改变lookat矩阵，（相机位置，世界UP向量均不变）==



## 相机缩放

通过改变 投影矩阵的FOV实现

注意 FOV范围最好是 （1,45）





# 光照 ：颜色



现在我们已经向每个顶点添加了一个法向量并更新了顶点着色器，我们还要更新顶点属性指针。注意，灯使用同样的顶点数组作为它的顶点数据，然而灯的着色器并没有使用新添加的法向量。我们不需要更新灯的着色器或者是属性的配置，但是我们必须至少修改一下顶点属性指针来适应新的顶点数组的大小
（这里的灯和box共用的是一个顶点数据，所以虽然灯不需要法线，但是仍然要改一下顶点属性指针的格式）

因此我们需要一个在世界空间中的顶点位置。我们可以通过把顶点位置属性乘以模型矩阵（不是观察和投影矩阵）来把它变换到世界空间坐标

（这里不同于GAMES101在视空间，是在世界坐标空间中计算光照）

## 解决不等比缩放与法向量矩阵

有待改进的地方：光照是在世界空间中算的，但是法向量却是顶点的局部空间中直接读入，这并不准，需要变化到世界空间。但是我们并不关心法向量的位置，变换时只考虑缩放和旋转。

我们就要从矩阵中移除位移部分，只选用模型矩阵左上角3×3的矩阵（注意，我们也可以把法向量的w分量设置为0，再乘以4×4矩阵（model * vec4(normal,0)）；这同样可以移除位移）。对于法向量，我们只希望对它实施缩放和旋转变换。

其次，如果模型矩阵执行了不等比缩放，顶点的改变会导致法向量不再垂直于表面了。因此，我们不能用这样的模型矩阵来变换法向量。下面的图展示了应用了不等比缩放的模型矩阵对法向量的影响：

![](https://learnopengl-cn.github.io/img/02/02/basic_lighting_normal_transformation.png)

每当我们应用一个不等比缩放时（注意：等比缩放不会破坏法线，因为法线的方向没被改变，仅仅改变了法线的长度，而这很容易通过标准化来修复），法向量就不会再垂直于对应的表面了，这样光照就会被破坏。

==修复这个行为的诀窍是使用一个为法向量专门定制的模型矩阵。这个矩阵称之为法线矩阵(Normal Matrix)，M是3x3左上model矩阵的逆的转置。==

**在不进行不等比缩放的行为下，M的值与原矩阵左上相等，这就是为什么在仅做平移和旋转时，可以直接单纯去掉平移量就可以保持垂直关系不变。**



注：
使用法向量矩阵涉及到矩阵求逆，开销比较大。最好留在CPU完成。
```cpp

//这里把法向量矩阵的计算放在了GPU的顶点着色器中，不太性能优先。
#version 330 core
layout (location = 0)
in vec3 aPos; 
layout (location = 1)
in vec3 aNormal;
out vec3 FragPos; 
out vec3 Normal; 
uniform mat4 model; 
uniform mat4 view;
uniform mat4 projection;
void main() 
{ 
FragPos = vec3(model * vec4(aPos, 1.0));
Normal = mat3(transpose(inverse(model))) * aNormal;
gl_Position = projection * view * vec4(FragPos, 1.0); 
}
```

## 关于逐顶点计算光照与逐像素计算光照

在光照着色器的早期，开发者曾经在顶点着色器中实现冯氏光照模型。在顶点着色器中做光照的优势是，相比片段来说，顶点要少得多，因此会更高效，所以（开销大的）光照计算频率会更低。然而，顶点着色器中的最终颜色值是仅仅只是那个顶点的颜色值，**片段的颜色值是由插值光照颜色所得来的。**结果就是这种光照看起来不会非常真实，除非使用了大量顶点。**

![](https://learnopengl-cn.github.io/img/02/02/basic_lighting_gouruad.png)

在顶点着色器中实现的冯氏光照模型叫做Gouraud着色(Gouraud Shading)，而不是冯氏着色(Phong Shading)。记住，由于插值，这种光照看起来有点逊色。冯氏着色能产生更平滑的光照效果。


## 材质：在GLSL中使用uniform的结构体
GLSL中一个结构体在设置uniform时并无任何区别，结构体只是充当uniform变量们的一个命名空间。所以如果想填充这个结构体的话，我们必须设置每个单独的uniform，但要以结构体名为前缀：

```cpp

//片段着色器
#version 330 core
struct Material {
vec3 ambient; 
vec3 diffuse;
vec3 specular;
float shininess; 
}; 

uniform Material material;




//CPU中使用
lightingShader.setVec3("material.ambient",  1.0f, 0.5f, 0.31f);
lightingShader.setVec3("material.diffuse",  1.0f, 0.5f, 0.31f);
lightingShader.setVec3("material.specular", 0.5f, 0.5f, 0.5f);
lightingShader.setFloat("material.shininess", 32.0f);
```

我们将环境光和漫反射分量设置成我们想要让物体所拥有的颜色，而将镜面分量设置为一个中等亮度的颜色，我们不希望镜面分量过于强烈。我们仍将反光度保持为32。

## 使用传统的bling-fong光照模型的材质属性表

http://devernay.free.fr/cours/opengl/materials.html


![[Pasted image 20221220030002.png]]


## 随时间变化的光

我们可以利用sin和glfwGetTime函数改变光源的环境光和漫反射颜色，从而很容易地让光源的颜色随着时间变化：

```cpp
glm::vec3 lightColor;
lightColor.x = sin(glfwGetTime() * 2.0f);
lightColor.y = sin(glfwGetTime() * 0.7f);
lightColor.z = sin(glfwGetTime() * 1.3f);

glm::vec3 diffuseColor = lightColor   * glm::vec3(0.5f); // 降低影响
glm::vec3 ambientColor = diffuseColor * glm::vec3(0.2f); // 很低的影响

lightingShader.setVec3("light.ambient", ambientColor);
lightingShader.setVec3("light.diffuse", diffuseColor);
```

这里虽然改了光的颜色，但是光源的颜色没有改，所以还是白色。
![[Pasted image 20221220032211.png]]




## 使用漫反射贴图补充漫反射向量和环境光向量

==注：
贴图是为了补充高光的细节，贴图采样结果 * 对应的分量，

我们也移除了环境光材质颜色向量，因为环境光颜色在几乎所有情况下都等于漫反射颜色，所以我们不需要将它们分开储存：

```cpp
struct Material {
    sampler2D diffuse;
    vec3      specular;
    float     shininess;
}; 
...
in vec2 TexCoords;
```

如果你非常固执，仍想将环境光颜色设置为一个（漫反射值之外）不同的值，你也可以保留这个环境光的`vec3`，但整个物体仍只能拥有一个环境光颜色。如果想要对不同片段有不同的环境光值，你需要对环境光值单独使用另外一个纹理。

==其实就是默认环境光和漫反射颜色相同==


### 实际使用
注意我们将在片段着色器中再次需要纹理坐标，所以我们声明一个额外的输入变量。接下来我们只需要从纹理中采样片段的漫反射颜色值即可：

```cpp
vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));  // 强度系数*漫反射分量*纹理贴图采样颜色值（采样器，纹理坐标）
```

**不要忘记将环境光的材质颜色设置为漫反射材质颜色同样的值。**

```
vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
```

这就是使用漫反射贴图的全部步骤了。

**还要在顶点着色器赋值一下纹理坐标：**
现在包含了顶点位置、法向量和立方体顶点处的纹理坐标。让我们更新顶点着色器来以顶点属性的形式接受纹理坐标，并将它们传递到**片段着色器**中：

```cpp
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;
...
out vec2 TexCoords;

void main()
{
    ...
    TexCoords = aTexCoords;
}
```

记得去更新两个VAO的顶点属性指针来匹配新的顶点数据，并加载箱子图像为一个纹理。
==注：
同之前使用纹理的步骤相似，为了支持多层纹理（最多16层）：
1.需要将顶点着色器中的**采样器Simple2D**  uniform  赋值（ =0  对应纹理单元0）
2.在代码中，绑定对应的纹理单元GL_TEXTURE0到管线，随后将纹理贴图0送入管线

在绘制箱子之前，我们希望将要用的纹理单元赋值到material.diffuse这个uniform采样器，并绑定箱子的纹理到这个纹理单元：

```cpp
lightingShader.setInt("material.diffuse", 0);
...
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, diffuseMap);
```

使用了漫反射贴图之后，细节再一次得到惊人的提升，这次箱子有了光照开始闪闪发光


## 高光贴图代替高光向量

我们利用一张高光贴图存储不同部位的高光信息。高光贴图的白色代表有高光，黑色代表无高光。 （==彩色的高光贴图自带的不均匀的RGB通道会影响最终画面结果带颜色==）

![[Pasted image 20221221222511.png]]

==注：
使用**Photoshop**或**Gimp**之类的工具，将漫反射纹理转换为镜面光纹理还是比较容易的，将图像转为黑白，在修改高光即可。




```cpp
lightingShader.setInt("material.specular", 1);
...
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, specularMap);  //纹理单元1使用高光贴图
```



```cpp
struct Material {
    sampler2D diffuse;
    sampler2D specular;
    float     shininess;
};
```

最后我们希望采样镜面光贴图，来获取片段所对应的镜面光强度：

```cpp
vec3 ambient  = light.ambient  * vec3(texture(material.diffuse, TexCoords));
vec3 diffuse  = light.diffuse  * diff * vec3(texture(material.diffuse, TexCoords));  
vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords)); // 类似于漫反射贴图 
FragColor = vec4(ambient + diffuse + specular, 1.0);
```


==注：
所有的贴图都是针对于同一物体，归属于纹理单元0,1,2.所以共用同一套纹理坐标。

通过使用镜面光贴图我们可以可以对物体设置大量的细节，比如物体的哪些部分需要有**闪闪发光**的属性，我们甚至可以设置它们对应的强度。镜面光贴图能够在漫反射贴图之上给予我们更高一层的控制。

## 多类光源

### 关于点光源衰减模拟

![[Pasted image 20221222233717.png]]

**我们可以将环境光分量保持不变，让环境光照不会随着距离减少，但是如果我们使用多于一个的光源，所有的环境光分量将会开始叠加，所以在这种情况下我们也希望衰减环境光照。**


### 关于聚光的模拟（锥形光）

![[Pasted image 20221222234112.png]]

```cpp
lightingShader.setVec3("light.position", camera.Position); lightingShader.setVec3("light.direction", camera.Front); lightingShader.setFloat("light.cutOff", glm::cos(glm::radians(12.5f)));
```

**传入的切光角是余弦值，而不是直接的角度。是因为我们用 LightDir点积SpotDir 得到角度的余弦值，如果比角度，会计算一个开销很大的反余弦，所以我们选择直接比较余弦**

会看到一个聚光，它仅会照亮聚光圆锥内的片段。看起来像是这样的：

![](https://learnopengl-cn.github.io/img/02/05/light_casters_spotlight_hard.png)



==但这仍看起来有些假，主要是因为聚光有一圈硬边。 现实中聚光没有这么明显的硬边缘，类似于硬阴影

#### 利用内外边缘平滑聚光的硬阴影
![[Pasted image 20221222235631.png]]


**利用实际角度与内外边缘的余弦值的插值 来实现平缓的边缘过渡。**

```cpp
float theta = dot(lightDir, normalize(-light.direction)); 
float epsilon = light.cutOff - light.outerCutOff;
float intensity = clamp((theta - light.outerCutOff) / epsilon, 0.0, 1.0);
... // 将不对环境光做出影响，让它总是能有一点光 
diffuse *= intensity; 
specular *= intensity;
...
```

## 多光源模拟实战




# 模型导入

## obj格式详解及补充

https://en.wikipedia.org/wiki/Wavefront_.obj_file


## assimp 的使用

Assimp很棒的一点在于，它抽象掉了加载不同文件格式的所有技术细节，只需要一行代码就能完成所有的工作：

```cpp
Assimp::Importer importer;
const aiScene *scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs);
```




assimp有很多后处理参数，具体见[https://assimp.sourceforge.net/lib_html/postprocess_8h.html]




==注：
因为我们的vertex,和纹理坐标Texcrod那一套是用glm的vec2/vec3写的，assimp库的基本结构比如 sence中的mesh，并没有定义对应的拷贝赋值运算符，所以需要手动写赋值，将asimp的结构转换为GLM的结构。==

## 灵活的使用结构体拓展属性指针

由于有了构造器，我们现在有一大列的网格数据用于渲染。在此之前我们还必须配置正确的缓冲，并通过顶点属性指针定义顶点着色器的布局。现在你应该对这些概念都很熟悉了，但我们这次会稍微有一点变动，使用结构体中的顶点数据：

```cpp
void setupMesh()
{
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);
    glGenBuffers(1, &EBO);

    glBindVertexArray(VAO);
    glBindBuffer(GL_ARRAY_BUFFER, VBO);

    glBufferData(GL_ARRAY_BUFFER, vertices.size() * sizeof(Vertex), &vertices[0], GL_STATIC_DRAW);  

    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, indices.size() * sizeof(unsigned int), 
                 &indices[0], GL_STATIC_DRAW);
//使用结构体Vertex封装各个属性，使得代码泛用性更强了
    // 顶点位置
    glEnableVertexAttribArray(0);   
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)0);
    // 顶点法线
    glEnableVertexAttribArray(1);   
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)offsetof(Vertex, Normal));
    // 顶点纹理坐标
    glEnableVertexAttribArray(2);   
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)offsetof(Vertex, TexCoords));

    glBindVertexArray(0);
}  
```



==C++结构体有一个很棒的特性，它们的内存布局是连续的(Sequential)。这点正是配置VBO的属性指针所需要的==

==同时，C++的结构体还提供一些标准库算法，如 sizeof()  , ofsetof(s，m)



同理，如果约定 纹理采样器uniform的命名遵循一定的规则 如，specular_texture0等，就可以写出 任意数量的纹理单元绑定任意数量的采样器，填充纹理。


```cpp
void Draw(Shader shader) 
{
    unsigned int diffuseNr = 1;
    unsigned int specularNr = 1;
    for(unsigned int i = 0; i < textures.size(); i++)
    {
             // 在绑定之前激活相应的纹理单元
        glActiveTexture(GL_TEXTURE0 + i);
  
        // 获取纹理序号（diffuse_textureN 中的 N）
        string number;
        string name = textures[i].type;
        if(name == "texture_diffuse")
            number = std::to_string(diffuseNr++);
        else if(name == "texture_specular")
            number = std::to_string(specularNr++);
//绑定纹理单元对应的采样器 ，命名格式遵循texture_specular0
        shader.setInt(("material." + name + number).c_str(), i);
  //纹理的id绑定到渲染管线。      
        glBindTexture(GL_TEXTURE_2D, textures[i].id);
    }
    glActiveTexture(GL_TEXTURE0);

    // 绘制网格
    glBindVertexArray(VAO);
    glDrawElements(GL_TRIANGLES, indices.size(), GL_UNSIGNED_INT, 0);
    glBindVertexArray(0);
}
```

我们首先计算了每个纹理类型的N-分量，并将其拼接到纹理类型字符串上，来获取对应的uniform名称。接下来我们查找对应的采样器，将它的位置值设置为当前激活的纹理单元，并绑定纹理。


### assimp没有直接对于GLM的向量的拷贝赋值函数


获取顶点数据非常简单，我们定义了一个Vertex结构体，我们将在每个迭代之后将它加到vertices数组中。我们会遍历网格中的所有顶点（使用`mesh->mNumVertices`来获取）。在每个迭代中，我们希望使用所有的相关数据填充这个结构体。顶点的位置是这样处理的：

```cpp
glm::vec3 vector; 
vector.x = mesh->mVertices[i].x;
vector.y = mesh->mVertices[i].y;
vector.z = mesh->mVertices[i].z; 
vertex.Position = vector;
```

注意我们为了传输Assimp的数据，我们定义了一个`vec3`的临时变量。使用这样一个临时变量的原因是Assimp对向量、矩阵、字符串等都有自己的一套数据类型，它们并不能完美地转换到GLM的数据类型中。**没有定义对应的拷贝赋值函数**


### assimp的抽象结构：

==为了高效的读写，node和mesh只保存数据的索引，真正的数据存储于sence中

顶点数据的索引存储于node中，但是我们需要根据索引从sence中取出具体的顶点数据，`scene->mMeshes[node->mMeshes[i]]`如下：

```cpp
void processNode(aiNode *node, const aiScene *scene)
{
    // 处理节点所有的网格（如果有的话）
    for(unsigned int i = 0; i < node->mNumMeshes; i++)
    {
        aiMesh *mesh = scene->mMeshes[node->mMeshes[i]]; 
        meshes.push_back(processMesh(mesh, scene));         
    }
    // 接下来对它的子节点重复这一过程
    for(unsigned int i = 0; i < node->mNumChildren; i++)
    {
        processNode(node->mChildren[i], scene);
    }
}
```

我们

### 索引
获取完顶点，还要存储图元（索引数据）
：

```cpp
for(unsigned int i = 0; i < mesh->mNumFaces; i++)
{
    aiFace face = mesh->mFaces[i];
    for(unsigned int j = 0; j < face.mNumIndices; j++)
        indices.push_back(face.mIndices[j]);
}
```


## 关于纹理的导入

**和节点存储顶点数据一样，mesh中只存储包含材质的索引，不存储材质数据本身。我们需要从sence中根据索引去除数据得到。**
网格材质索引位于它的mMaterialIndex属性中，我们同样可以用它来检测一个网格是否包含有材质：

```cpp
if(mesh->mMaterialIndex >= 0)
{
//根据mesh中的材质索引，从sence中取出材质数据
    aiMaterial *material = scene->mMaterials[mesh->mMaterialIndex];
   //下一步，是将assimp的材质数据，转换为自己使用的texture数据，也就是分别存储材质的id,类型 
   //loadMaterialTextures  是自己写的工具函数，从材质中获取纹理。这个函数将会返回一个Texture结构体的vector，该vector就是所要的材质容器
    vector<Texture> diffuseMaps = loadMaterialTextures(material, 
                                        aiTextureType_DIFFUSE, "texture_diffuse");
    textures.insert(textures.end(), diffuseMaps.begin(), diffuseMaps.end());
    vector<Texture> specularMaps = loadMaterialTextures(material, 
                                        aiTextureType_SPECULAR, "texture_specular");
    textures.insert(textures.end(), specularMaps.begin(), specularMaps.end());
}
```



**loadMaterialTextures函数遍历了给定纹理类型的所有纹理位置，获取了纹理的文件位置，并加载并和生成了纹理，将信息储存在了一个Vertex结构体中：

```cpp
vector<Texture> loadMaterialTextures(aiMaterial *mat, aiTextureType type, string typeName)
{
    vector<Texture> textures;
    for(unsigned int i = 0; i < mat->GetTextureCount(type); i++)
    {
        aiString str;
        //GetTextureCount，GetTexture是assimp的函数，作用是得到材质的本地相对路径
        mat->GetTexture(type, i, &str);
        Texture texture;
        //TextureFromFile是来自stb_image.h的函数，作用是（相对路径，根路径）读取材质，并生成一个纹理的id
        texture.id = TextureFromFile(str.C_Str(), directory);
        texture.type = typeName;
        //texture的纹理路径也是默认相对路径
        texture.path = str;
        textures.push_back(texture);
    }
    return textures;
}
```





==注意：在网络上找到的某些模型会对纹理位置使用绝对(Absolute)路径，这就不能在每台机器上都工作了。在这种情况下，你可能会需要手动修改这个文件，来让它对纹理使用本地路径（如果可能的话）。


### 关于纹理加载的优化（实现纹理重用）

**在实际读写中，stb_image.h的 TextureFromFile（）加载纹理，生成纹理id的函数其实是一个开销很大的操作**, 而且现实中有很多纹理重用现象(**比如，一个箱子的六面同时使用一张纹理**)，所以可以定义一个加载过的纹理vector **textures_loaded**，如果出现纹理重用，则直接vector拷贝，比调用 TextureFromFile（）重新加载纹理要高效很多。


代码：

```cpp
vector<Texture> loadMaterialTextures(aiMaterial *mat, aiTextureType type, string typeName)
{
    vector<Texture> textures;
    for(unsigned int i = 0; i < mat->GetTextureCount(type); i++)
    {
        aiString str;
        mat->GetTexture(type, i, &str);
        bool skip = false;
        for(unsigned int j = 0; j < textures_loaded.size(); j++)
        {
        //textures_loaded是一个声明在模型model类内部的私有容器，用于存储已经加载完成的vector
        //（本质上，该优化是一个利用额外的空间换取时间的过程）
            if(std::strcmp(textures_loaded[j].path.data(), str.C_Str()) == 0)
            {
                textures.push_back(textures_loaded[j]);
                skip = true; 
                break;
            }
        }
        if(!skip)
        {   // 如果纹理还没有被加载，则加载它
            Texture texture;
            texture.id = TextureFromFile(str.C_Str(), directory);
            texture.type = typeName;
            texture.path = str.C_Str();
            textures.push_back(texture);
            //加载新的纹理，同时更新textures_loaded集
            textures_loaded.push_back(texture);
        }
    }
    return textures;
}
```



# 光照系统再谈

## 深度测试





深度缓冲，就像颜色缓冲(Color Buffer)（储存所有的片段颜色：视觉输出）一样，在每个片段中储存了信息，并且（通常）和颜色缓冲有着一样的宽度和高度。

**深度缓冲是由窗口系统自动创建的，它会以16、24或32位float的形式储存它的深度值。在大部分的系统中，深度缓冲的精度都是24位的。

当深度测试(Depth Testing)被启用的时候，OpenGL会将一个片段的深度值与深度缓冲的内容进行对比。OpenGL会执行一个深度测试，如果这个测试通过了的话，深度缓冲将会更新为新的深度值(深度以片段为单位存储)。如果深度测试失败了，片段将会被丢弃。（其实就是以像素为单位的画家算法）

==一般来说，深度depth 的计算是在片段着色器和模板测试之后，

屏幕空间坐标与通过OpenGL的glViewport所定义的视口密切相关，并且可以直接使用GLSL内建变量gl_FragCoord从片段着色器中直接访问。gl_FragCoord的x和y分量代表了片段的屏幕空间坐标（其中(0, 0)位于左下角）。gl_FragCoord中也包含了一个z分量，它包含了片段真正的深度值。z值就是需要与深度缓冲内容所对比的那个值。

现在大部分的GPU都提供一个叫做**提前深度测试(Early Depth Testing)** 的硬件特性。提前深度测试允许深度测试在片段着色器之前运行。只要我们清楚一个片段永远不会是可见的（它在其他物体之后），我们就能提前丢弃这个片段。从而节约片段着色器的计算开销。

==当使用提前深度测试时，片段着色器的一个限制是你不能写入片段的深度值。（因为如果运行中有写入更新深度值，理论上来说，我们就不可能做提前深度测试）


深度测试默认是禁用的，所以如果要启用深度测试的话，我们需要用GL_DEPTH_TEST选项来启用它：

```cpp
glEnable(GL_DEPTH_TEST);
```

如果你启用了深度缓冲，你还应该在每个渲染迭代之前使用GL_DEPTH_BUFFER_BIT来清除深度缓冲，否则你会仍在使用上一次渲染迭代中的写入的深度值：

```cpp
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
```

可以想象，在某些情况下你会需要对所有片段都执行深度测试并丢弃相应的片段，但**不**希望更新深度缓冲。基本上来说，你在使用一个只读的(Read-only)深度缓冲。OpenGL允许我们禁用深度缓冲的写入，只需要设置它的深度掩码(Depth Mask)设置为`GL_FALSE`就可以了：

```cpp
glEnable(GL_DEPTH_TEST);//启用深度测试
glDepthMask(GL_FALSE);//禁止写入深度缓存，（实现提前深度测试）
```


### 深度测试函数

opengl 通过深度函数（Depth Function）控制深度测试的通过情况
这个函数接受下面表格中的比较运算符：

函数                                描述

GL_ALWAYS               永远通过深度测试

GL_NEVER                 永远不通过深度测试

GL_LESS                      在片段深度值小于缓冲的深度值时通过测试

GL_EQUAL                在片段深度值等于缓冲区的深度值时通过测试

GL_LEQUAL              在片段深度值小于等于缓冲区的深度值时通过测试

GL_GREATER             在片段深度值大于缓冲区的深度值时通过测试

GL_NOTEQUAL         在片段深度值不等于缓冲区的深度值时通过测试

GL_GEQUAL                 在片段深度值大于等于缓冲区的深度值时通过测试

**opengl 渲染管线默认的深度函数是GL_LESS，它将会丢弃深度值大于等于当前深度缓冲值的所有片段。**

```cpp
glEnable(GL_DEPTH_TEST);
glDepthFunc(GL_ALWAYS);
```
### 深度测试精度问题

在opengl的投影矩阵中，棱台的（x，y）与NDC 的（x,y）是严格的线性对应关系。

但是，==视空间的Z值与深度测试的深度值并不是线性关系==，**因为深度测试的精度要求在越近的地方越高**
==因此该对应的关系应该与Z成反比

![[Pasted image 20230122040654.png]]

![[Pasted image 20230122040947.png]]



### （测试）可视化深度缓冲为灰度图比较线性与非线性的区别

#### 非线性的深度值


内建gl_FragCoord向量的z值包含了那个特定片段的深度值。**直接将片段着色器的深度值作为（R,G，B）输出，得到深度的灰度图**


```cpp
void main()
{
    FragColor = vec4(vec3(gl_FragCoord.z), 1.0);
}
```



![](https://learnopengl-cn.github.io/img/04/01/depth_testing_func_less.png)



![](https://learnopengl-cn.github.io/img/04/01/depth_testing_visible_depth.png)

这很清楚地展示了**物体非常靠近近平面（nearer）时，可以看出灰度的渐变，但是在较远距离后，迅速变为白色（深度为1）,显然图片中反应的深度值与实际的距离（Z值）并不是线性的关系**








#### 线性的深度值


**要从非线性的深度值(0,1)中，还原出 视椎体的原本线性的Z值(近平面，远平面)需要结合投影矩阵逆转

```cpp
#version 330 core
out vec4 FragColor;

float near = 0.1; 
float far  = 100.0; 

float LinearizeDepth(float depth) 
{
    float z = depth * 2.0 - 1.0; // 将（0,1）的深度值还原为NDC的（-1,1）
    return (2.0 * near * far) / (far + near - z * (far - near));   //单独对深度X投影矩阵的倒数 
}

void main()
{             
    float depth = LinearizeDepth(gl_FragCoord.z) / far; // Z值变为（0,1）的线性范围
    FragColor = vec4(vec3(depth), 1.0);
}
```



![](https://learnopengl-cn.github.io/img/04/01/depth_testing_visible_linear.png)

**可以看到，全是黑色，也就是0（因为对于整个视空间范围（near,far）来说，我们的物体还很近）**


### 一些避免深度冲突的方法

当场景中物体的三角面比较靠近时，会互相覆盖像素，争夺片段着色器，称为==深度冲突。

经过以上尝试，有一些比较容易想到的避免深度冲突的方法：
第一个也是最重要的技巧是**永远不要把多个物体摆得太靠近，以至于它们的一些三角形会重叠**。通过在两个物体之间设置一个用户无法注意到的偏移值，你可以完全避免这两个物体之间的深度冲突。在箱子和地板的例子中，我们可以将箱子沿着正y轴稍微移动一点。箱子位置的这点微小改变将不太可能被注意到，但它能够完全减少深度冲突的发生。然而，这需要对每个物体都手动调整，并且需要进行彻底的测试来保证场景中没有物体会产生深度冲突。

第二个技巧是**尽可能将近平面设置远一些**。在前面我们提到了精度在靠近**近**平面时是非常高的，所以如果我们将**近**平面远离观察者，我们将会对整个平截头体有着更大的精度。然而，将近平面设置太远将会导致近处的物体被裁剪掉，所以这通常需要实验和微调来决定最适合你的场景的**近**平面距离。

另外一个很好的技巧是牺牲一些性能，**使用更高精度的深度缓冲**。大部分深度缓冲的精度都是24位的，但现在大部分的显卡都支持32位的深度缓冲，这将会极大地提高精度。所以，牺牲掉一些性能，你就能获得更高精度的深度测试，减少深度冲突。


## 模板测试

### 模板测试与模板缓冲

**当片段着色器处理完一个片段之后，模板测试(Stencil Test)会开始执行**，和深度测试一样，它也可能会丢弃片段。

一个模板缓冲中，（通常）每个模板值(Stencil Value)是8位的。所以每个像素/片段一共能有256种不同的模板值。 ==也就是说，模板缓冲本质是以片段为基础的，256种不同值的“特殊片段”。==  

**个人理解：因为片段具有三维空间的特性（以像素为单位一次性渲染空间中所有的点，并且还没做深度测试，还有不同的Zbuffer），所以模板测试也是有三维的特征的，可以看做加在三维空间中的mask.** 

**每个窗口库都需要为你配置一个模板缓冲。GLFW自动做了这件事**


模板缓冲操作允许我们在渲染片段时将模板缓冲设定为一个特定的值。通过在渲染时修改模板缓冲的内容，我们**写入**了模板缓冲。在**同一个（或者接下来的）渲染迭代**中，我们可以**读取**这些值，来决定丢弃还是保留某个片段。使用模板缓冲的时候你可以尽情发挥，但大体的步骤如下：

-   启用模板缓冲的写入。
-   渲染物体，**根据渲染的物体**更新模板缓冲的内容。
-   禁用模板缓冲的写入。（这一步为了避免之后渲染的物体覆盖模板缓冲）
-   渲染（其它）物体，这次**根据模板缓冲的内容**丢弃特定的片段。

所以，通过使用模板缓冲，我们可以**根据场景中已绘制的其它物体的片段**，来决定是否丢弃特定的片段。


### 模板函数

和深度测试一样，我们对模板缓冲应该通过还是失败，以及它应该如何影响模板缓冲，也是有一定控制的。一共有两个函数能够用来配置模板测试：**glStencilFunc（定义通过方式，参考值，以及位掩码mask）和glStencilOp(定义不通过模板，通过模板但不通过深度，两者都通过后的处理方式)**

==关于 glStencilFunc的mask的作用：==
所有的参考值和模板缓冲值都会先和mask做AND运算。
GL_LESS 通过，当且仅当 满足: ( stencil & mask ) ref < ( stencil & mask )。GL_GEQUAL通过，当且仅当( stencil & mask ) >= ( ref & mask )。
==mask一般为0xFF,如果置0，相当于  stencil和ref 的16进制某一位先天为0.==


glStencilFunc(GLenum func, GLint ref, GLuint mask)一共包含三个参数：

-   `func`：设置模板测试函数(Stencil Test Function)。这个测试函数将会应用到已储存的模板值上和glStencilFunc函数的`ref`值上。可用的选项有：GL_NEVER、GL_LESS、GL_LEQUAL、GL_GREATER、GL_GEQUAL、GL_EQUAL、GL_NOTEQUAL和GL_ALWAYS。它们的语义和深度缓冲的函数类似。
-   `ref`：设置了模板测试的参考值(Reference Value)。模板缓冲的内容将会与这个值进行比较。
-   `mask`：设置一个掩码，它将会与参考值和储存的模板值在测试比较它们之前进行与(AND)运算。初始情况下所有位都为1。

l例如，函数被设置为：

```cpp
glStencilFunc(GL_EQUAL, 1, 0xFF)
```

这会告诉OpenGL，只要一个片段的模板值等于(`GL_EQUAL`)参考值1，片段将会通过测试并被绘制，否则会被丢弃。



glStencilOp(GLenum sfail, GLenum dpfail, GLenum dppass)一共包含三个选项，我们能够设定每个选项应该采取的行为：

-   `sfail`：模板测试失败时采取的行为。
-   `dpfail`：模板测试通过，但深度测试失败时采取的行为。
-   `dppass`：模板测试和深度测试都通过时采取的行为。

每个选项都可以选用以下的其中一种行为：
各类操作参考：

GL_KEEP 保持当前储存的模板值

GL_ZERO 将模板值设置为0

GL_REPLACE 将模板值设置为glStencilFunc函数设置的`ref`值

GL_INCR 如果模板值小于最大值则将模板值加1

GL_INCR_WRAP 与GL_INCR一样，但如果模板值超过了最大值则归零

GL_DECR 如果模板值大于最小值则将模板值减1

GL_DECR_WRAP 与GL_DECR一样，但如果模板值小于0则将其设置为最大值

GL_INVERT 按位翻转当前的模板缓冲值

**默认情况下glStencilOp是设置为`(GL_KEEP, GL_KEEP, GL_KEEP)`的**

### 模板运用：描出物体轮廓

#### 算法分析


```cpp
glEnable(GL_DEPTH_TEST);  //初始开启深度测试，模板测试
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);  //先设置本轮模板测试的操作方式：本例为同时通过模板测试和深度测试的设1，效果是模板缓存的是本轮绘制的物体，值为1，同时由于考虑了深度测试，模板缓冲会有遮挡关系（被挡住的物体不会通过深度测试，从而不会replace模板值为1）

glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT); //循环之前清理缓存

//这里选择先绘制不需要描边的地板，注意绘制时必须关闭模板的写入
glStencilMask(0x00); // 记得保证我们在绘制地板的时候不会更新模板缓冲
normalShader.use();
DrawFloor()  


//开启模板写入，同时 模板函数设为总是通过，这意味着只要渲染过该物体的片段，都会通过模板测试（但是不一定通过深度测试）
glStencilFunc(GL_ALWAYS, 1, 0xFF); 
glStencilMask(0xFF); 
DrawTwoContainers();//渲染物体，保留物体的模板

//接下来使用刚才的模板决定是否丢弃下一次渲染的片段（此时一定要关闭模板的写入）
glStencilFunc(GL_NOTEQUAL, 1, 0xFF); 
glStencilMask(0x00); //此时一定要关闭模板的写入
glDisable(GL_DEPTH_TEST);//关掉深度测试，是为了在接下来的draw大的箱子阶段，我们不希望地板挡住箱子，从而不画出来
shaderSingleColor.use(); 
DrawTwoScaledUpContainers(); //此时，不等于1的片段，才能通过模板测试被保留，并画出，因为是更大的箱子，所以保留的就是边框
glStencilMask(0xFF);  //画完一定要还原（开启模板写入，否则模板不会被清0，之后所有的模板测试都是用的第一次的模板缓存）
glEnable(GL_DEPTH_TEST);  
```


#### 此方法依然存在的问题

##### 复杂物体轮廓上移

https://zhuanlan.zhihu.com/p/464740166
**是模型的坐标原点的问题**
教程给出的例子是实现一个正方体周围的轮廓，而作为练习，我和许多人一样，想到在之前练习中已经加载的人型模型的基础上实现轮廓。严格来说，这并不是一个问题，只是“需求不同”。但是讲到轮廓，大多数人第一反应会是贴合于外形的一条曲线；然后参照着教程正方体里的例子，应用到人型模型上，得到的图像中，“轮廓”有一个明显的上移。

![[Pasted image 20230205050647.png]]

教程里，对模型本身和轮廓都使用了相同的顶点着色器，并且在传入 uniform 的 model 变换矩阵时，将轮廓的 model 矩阵多叠加一次scale(1.1f, 1.1f, 1.1f)。从这张上移的图其实可以猜测的出这样子做 scaling 为什么会带来这个问题。**模型的“基准点”其实在两脚之间，所以直接以 model 变换矩阵的方式去做，会以此基准点做 scaling，轮廓整体会向外长大一轮，这也解释了为什么模型内部是没有轮廓的。

至于为什么正方体的实现是正常的“轮廓”，是一个巧合，首先正方体的基准点在正方点的中心；同时正方体是实心的，这一点也很重要。

沿顶点的法线方向移动可以初步解决坐标系的问题
```cpp
vec4(0.1*Normal, 0);
```
![[Pasted image 20230205051154.png]]


**但是会引入新的问题：比如面移动了，但并没有放大，这样会使mesh之间连接断裂。

**也可以使用 glPolygonMode 函数并增大线宽来实现


## 帧缓冲
