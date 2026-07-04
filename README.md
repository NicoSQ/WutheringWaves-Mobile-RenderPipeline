# WutheringWaves-Mobile-RenderPipeline
## 截帧环境 
MuMu模拟器/小米14Ultra/画质高/分辨率1920x1080  
（补充图片）
## 渲染管线分析
### DepthPass
#### 角色阴影图
计算角色专属的高精度阴影图，独立于场景CSM阴影，移动端更加节省带宽。**阴影图大小 512x512，精度 D16**。用角色的LOD2模型做代理模型渲染，减少提交vertexBuffer数据量。  
存在历史帧的角色阴影图复用，结合后面 LightPass 的屏幕空间阴影 Mask 结合部分，这一帧只更新了一张角色阴影图。
<img width="1913" height="858" alt="image" src="https://github.com/user-attachments/assets/7eb79249-a448-4f06-bc55-22579604d7ec" />
#### 主光源 CSM 阴影
场景主方向光的 CSM 级联阴影，2048x512，精度 D16，按距离分四层级联横向打包，每一级512x512。
CSM 级联阴影每帧交替更新，**这一帧只更新了最左边，距离最近的 Tile，后面三级距离远的级联相机变化慢，不用每帧重画，直接复用历史帧，减少更新写入带宽和 4 倍的 DP 消耗**。  
开始绘制
<img width="2557" height="1174" alt="image" src="https://github.com/user-attachments/assets/30fe1e40-c34f-4eb9-b078-906aa0b9d62f" />
结束绘制
<img width="2556" height="1206" alt="image" src="https://github.com/user-attachments/assets/8d89be28-fe8b-410a-8f6c-5b36bc0dc86d" />
### BasePass
#### 宏观
鸣潮移动端沿用延迟渲染管线，BasePass 将场景不透明物体几何渲染进 GBuffer，**一共输出 6 张 GBufferRT + 1 张 Depth**。  
其中，BasePass 部分也是 PSO 切换大头，**总共约 168 次渲染状态切换，263 DP 提交**。  
角色部分在 GBuffer 阶段有特殊标记，供后续 LightPass 阶段进行单独的 NPR 光照计算。 
<img width="1986" height="173" alt="image" src="https://github.com/user-attachments/assets/aeeaa973-803c-45c9-b781-9b1d8e4831fe" />
#### 植被的 DepthPrePass 
开始提前绘制场景所有植被的 PreDepth，减少后续绘制植被的 OverDraw（镂空直接来自叶片网格几何），**深度统一存入 D24S8 的深度图**。
<img width="2558" height="1346" alt="image" src="https://github.com/user-attachments/assets/c885e81b-7c36-49d4-b9e3-4021bc34bd1c" />
#### GBuffer Pass
按照不同材质属性，分别输出不同的 GBuffer。这里可以参考鸣潮在 UE 官方上的分享，链接点此处。  
（待补充）
### 构建 HZD + GPU Driven 遮挡剔除
对 BasePass 输出的深度图进行处理，把硬件深度转换为线性深度（R32F），**再降到半分辨率，构建一条分辨率从 512x512 —> 4x4 的 Hi-Z 深度金字塔，共 8 级 R16F 精度的深度图**，加速后续的屏幕空间效果计算。  
移动端优化：转化的线性深度图降低一半分辨率（960x540）处理，转为 R16F 精度存储。  
**第一级 Hi-Z 深度图（512x512）**
<img width="2549" height="1167" alt="image" src="https://github.com/user-attachments/assets/0a4786a2-eac2-4b65-a5ac-51b2544c6db8" />  
接下来做 GPU-Driven 遮挡剔除的前置处理。  
**首先CPU 提交物体包围盒数据到 GPU**。14 次 vkCmdCopyBufferToImage，将物体包围盒数据存入贴图，每帧提交 128x128 最多 16384 个物体实例的包围和数据，RGBA32F精度。
<img width="2547" height="1177" alt="image" src="https://github.com/user-attachments/assets/95bd474e-2b8c-48d3-9516-3935f6afa4ab" />  
**再把前置计算的 8 级 HZD 深度打包成 512x1024 的图集**。方便后续 GPU-Driven 遮挡剔除一次绑定访问所有 HZD 深度层级，减少读取带宽。  
<img width="2551" height="1172" alt="image" src="https://github.com/user-attachments/assets/e734047a-7fcc-4292-b46e-64eaca643f4b" />  
最后，**计算依据 HZB Atlas 的 GPU-Driven 的遮挡剔除**。仅依靠 GPU 驱动，用贴图内存储的物体包围盒数据 vs HZB 的深度测试，依靠一次 DP 消耗，提前判定哪些物体被遮挡，避免后续进入昂贵的着色，削减 DP 和 OverDraw。  
<img width="2558" height="1178" alt="image" src="https://github.com/user-attachments/assets/658a26fc-1d43-46ce-ba6f-7ea44d54639a" />





