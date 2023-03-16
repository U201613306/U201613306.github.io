# 写在前面



**做仿真时有遇到过海面模拟开销影响平台实时帧率输出的问题**，所以认真的查各类资料,想在Unreal内做FFT海面。资料是有的

[无敌的Y神技美系列](https://zhuanlan.zhihu.com/p/67481306)

![image-20230316190249609](C:\Users\xue\AppData\Roaming\Typora\typora-user-images\image-20230316190249609.png)

​                                                                               （图）劝退名言233



**我头铁的看了一下，由于暂时还没系统点DX的技能树，让我改改shaderToy的代码GLSL改HLSL部署到虚幻里还可以，让我用虚幻的连连看写写shader蓝图还可以。但是直接撸UE的DX代码就劝退了。** 果然unreal内的海面只能继续交给大佬的插件了233. 不过，不甘心的我 决定学习别人的代码用OpenGL做一版海面。

# 原理

傅里叶，傅里叶逆变换，快速傅里叶，快速傅里叶逆变换可以参考 

[链接:一小时学会傅里叶](https://zhuanlan.zhihu.com/p/31584464)

![image-20230316191909247](C:\Users\xue\AppData\Roaming\Typora\typora-user-images\image-20230316191909247.png)



![image-20230316192200088](C:\Users\xue\AppData\Roaming\Typora\typora-user-images\image-20230316192200088.png)



其实，就是 由右侧频谱 混合叠加，得到某一个位置（x,z）处的高度h （也就是y）

但是，解频域h(k,t)，我们还需要假定一个频谱 （一般是菲利普频谱）：

![image-20230316192521165](C:\Users\xue\AppData\Roaming\Typora\typora-user-images\image-20230316192521165.png)

















# 代码分析

