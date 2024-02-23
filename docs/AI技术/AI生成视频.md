
AI生成视频的基本功能功能点
文字生成视频
图片生成视频
大运动的AI视频
视频局部重绘
视频整体重绘
视频画幅扩展


测评点：
基本功能
清晰度
细节度
视频逼真程度
镜头稳定性
语义指令理解

Latent Diffusion
vae编码器
视频压缩网络Video Compression Network

AI生成视频训练process
compression -> spacetime patches

需要学习的
视频压缩算法，论文《High-Resolution Images Synthesis with Latent Diffusion》
视频训练算法，论文《Vision Transformer (ViT)》
视频特征拆分，

![whiteboard_exported_image.png](..%2F..%2F..%2F..%2FDownloads%2Fwhiteboard_exported_image.png)
视频经过视频编码器的处理，先后生成【时空潜空间特征】并被进一步分解成【时空图像块】，最终转化成了一维向量，输入至Transformer Diffusion的组建中，进行扩散模型的加噪去噪过程。

在解码时则反过来，利用Transformer解码器，将处理后的向量内容转化为【时空图像块】再拼合成【时空潜空间特征】 ，最终，利用vae模型将潜空间特征转化为最终的视频。



以上就是关于Sora的训练过程的核心揭秘了。