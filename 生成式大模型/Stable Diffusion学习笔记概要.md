- **1.SD基础原理**
	- 在前向扩散过程中，SD模型持续对一张图像添加高斯噪声直至变成**随机噪声矩阵**。
	- 在反向扩散过程中，SD模型进行**去噪声过程**，将一个随机噪声矩阵逐渐去噪声直至生成一张图像。
	- ![image.png](../assets/image_1710166052241_0.png)
- **2. SD架构**
	- **Stable Diffusion模型整体上是一个End-to-End模型**，主要由VAE（变分自编码器，Variational Auto-Encoder），U-Net以及CLIP Text Encoder三个核心组件构成。
		- （1）**在Stable Diffusion中，VAE模型主要起到了图像压缩和图像重建的作用**；
		- （2）在Stable Diffusion中，**U-Net模型是一个关键核心部分，能够预测噪声残差**，并结合Sampling method（调度算法：PNDM，DDIM，K-LMS等）对输入的特征矩阵进行重构，**逐步将其从随机高斯噪声转化成图片的Latent Feature**。
		  collapsed:: true
			- 具体来说，在前向推理过程中，SD模型通过反复调用 U-Net，将预测出的噪声残差从原噪声矩阵中去除，得到逐步去噪后的图像Latent Feature，再通过VAE的Decoder结构将Latent Feature重建成像素级图像
		- （3）**CLIP模型是一个基于对比学习的多模态模型，主要包含Text Encoder和Image Encoder两个模型**。其中Text Encoder用来提取文本的特征，可以使用NLP中常用的text transformer模型作为Text Encoder；而Image Encoder主要用来提取图像的特征，可以使用CNN/vision transformer模型（ResNet和ViT）作为Image Encoder。
	- ![](https://pic1.zhimg.com/80/v2-a643ee39e80807d6b7236d15f1c289a8_720w.webp)
- **3. SD+Lora模型**
	- **3.1 LoRA与SD模型的训练**
		- **LoRA模型的训练逻辑**是首先冻结SD模型的权重，然后**在SD模型的U-Net结构中注入LoRA模块，并将其与CrossAttention模块结合，并只对这部分参数进行微调训练。**
		- ![](https://pic2.zhimg.com/80/v2-210423e6f135feb78a4088fc6d521161_720w.webp)
	- **3.2 LoRA模型的优势**
		- 与SD系列模型全参训练相比，**LoRA模型训练速度更快**。
		- **非常低的算力要求**。我们可以在2080Ti级别的算力设备上进行LoRA模型的训练。
		- 由于只是与SD模型的结合训练，**LoRA模型本身的参数量非常小，最小可至3M左右**。
		- LoRA模型能在小数据集上进行训练（1张以上即可），并且与不同SD模型都能较好兼容与迁移适配。
		- 训练时主模型参数保持不变，LoRA模型能更好的在主模型的能力上优化学习。
		- SD主模型之间切换需要将所有模型参数加载到内存，从而造成严重的I/O瓶颈。通过对权重更新的有效参数化，**LoRA模型之间的切换加载既高效又容易**。
- **4. Stable Diffusion经典应用场景**
	- **4.1 文本生成图像**
		- 输入：prompt
		- 输出：图像
		- ![文生图.png](../assets/文生图_1708928614821_0.png)
		- 其中Load Checkpoint模块代表对SD模型的主要结构进行初始化（VAE，U-Net），CLIP Text Encode表示文本编码器，可以输入prompt和negative prompt，来控制图像的生成，Empty Latent Image表示初始化的高斯噪声，KSampler表示调度算法以及SD相关生成参数，VAE Decode表示使用VAE的解码器将低维度的隐空间特征转换成像素空间的生成图像。
	- **4.2 图片生成图片**
		- 输入：图像 + prompt
		- 输出：图像
		- ![图生图.png](../assets/图生图_1708928602013_0.png)
		- 其中Load Checkpoint模块代表对SD模型的主要结构进行初始化（VAE，U-Net），CLIP Text Encode表示文本编码器，可以输入prompt和negative prompt，来控制图像的生成，Load Image表示输入的图像，KSampler表示调度算法以及SD相关生成参数，VAE Encode表示使用VAE的编码器将输入图像转换成低维度的隐空间特征，VAE Decode表示使用VAE的解码器将低维度的隐空间特征转换成像素空间的生成图像。
		- 与文字生成图片的过程相比，图片生成图片的预处理阶段，先把噪声添加到隐空间特征中。我们设置一个去噪强度（Denoising strength）控制加入多少噪音。如果它是0，就不添加噪音。如果它是1，则添加最大数量的噪声，使潜像成为一个完整的随机张量，如果将去噪强度设置为1，就完全相当于文本转图像，因为初始潜像完全是随机的噪声。
	- **4.3 图像inpainting**
		- 图像inpainting最初用在**图像修复**上，是一种图像修复技术，可以将图像中的水印、噪声、标志等瑕疵去除。
		  ```
		  传统的图像inpainting过程可以分为两步：
		    1. 找到图像中的瑕疵部分 
		    2. 对瑕疵部分进行重绘去除，并填充图像内容使得图像语义完整自然。
		  ```
		- ```
		  那么什么是图像编辑呢？
		  图像编辑是指对图像进行修改、调整和优化的过程。它可以包括对图像的颜色、对比度、亮度、饱和度等进行调整，以及修复图像中的缺陷、删除不需要的元素、添加新的图像内容等操作。
		  在SD中，主要是通过给定一个想要编辑的区域mask，并在这个区域mask圈定的范围内进行文本生成图像的操作，从而编辑mask区域的图像内容。
		  ```
		- SD中的图像inpainting流程如下所示：
			- ![图像inpainting流程.jpg](../assets/图像inpainting流程_1708928590939_0.jpg)
			- 图像inpainting整体上和图生图流程一致，不过为了保证mask以外的图像区域不发生改变，在去噪过程的每一步，我们利用mask将Latent Feature中不需要重建的部分都替换成原图最初的特征，只在mask部分进行特征的重建与优化。
		- 输入：图像 + mask + prompt
		- 输出：图像
		- ![图像inpainting.png](../assets/图像inpainting_1708928586365_0.png)
		- 其中Load Checkpoint模块代表对SD模型的主要结构进行加载（VAE，U-Net）。CLIP Text Encode表示SD模型的文本编码器，可以输入prompt和negative prompt，来引导图像的生成。Load Image表示输入的图像和mask。KSampler表示调度算法以及SD相关生成参数。VAE Encode表示使用VAE的Encoder将输入图像和mask转换成Latent Feature，VAE Decode表示使用VAE的Decoder将Latent Feature重建成像素级图像。
	- **4.4 使用controlnet辅助生成图片**
		- 输入：素描图 + prompt
		- 输出：图像
		- ![使用controlnet辅助生成图片.png](../assets/使用controlnet辅助生成图片_1708928575749_0.png)
		- 其中Load Checkpoint模块代表对SD模型的主要结构进行初始化（VAE，U-Net），CLIP Text Encode表示文本编码器，可以输入prompt和negative prompt，来控制图像的生成，Load Image表示输入的ControlNet需要的预处理图，Empty Latent Image表示初始化的高斯噪声，Load ControlNet Model表示对ControlNet进行初始化，KSampler表示调度算法以及SD相关生成参数，VAE Decode表示使用VAE的解码器将低维度的隐空间特征转换成像素空间的生成图像。
	- **4.5 超分辨率重建**
		- 输入：prompt/（图像 + prompt）
		- 输入：图像
		- ![超分辨率重建.png](../assets/超分辨率重建_1708928568770_0.png)
		- 其中Load Checkpoint模块代表对SD模型的主要结构进行初始化（VAE，U-Net），CLIP Text Encode表示文本编码器，可以输入prompt和negative prompt，来控制图像的生成，Empty Latent Image表示初始化的高斯噪声，Load Upscale Model表示对超分辨率重建模型进行初始化，KSampler表示调度算法以及SD相关生成参数，VAE Decode表示使用VAE的解码器将低维度的隐空间特征转换成像素空间的生成图像，Upscale Image表示将生成的图片进行超分。