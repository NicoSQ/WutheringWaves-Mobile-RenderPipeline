# WutheringWaves-Mobile-RenderPipeline
## 截帧环境 
MuMu模拟器/小米14Ultra/画质高/分辨率1920x1080  
（补充图片）
## 渲染管线分析
### DepthPass
#### 角色阴影图
计算角色专属的高精度阴影图，独立于场景CSM阴影，移动端更加节省带宽。**阴影图大小512x512，精度D16**。用角色的LOD2模型做代理模型渲染，减少提交vertexBuffer数据量。  
存在历史帧的角色阴影图复用，结合后面LightPass的屏幕空间阴影Mask结合部分，这一帧只更新了一张NPC的角色阴影图。
<img width="1913" height="858" alt="image" src="https://github.com/user-attachments/assets/7eb79249-a448-4f06-bc55-22579604d7ec" />
#### 主光源CSM阴影
场景主方向光的CSM级联阴影，2048x512，精度D16，按距离分四层级联横向打包，每一级512x512。
CSM级联阴影每帧交替更新，**这一帧只更新了最左边，距离最近的Tile，后面三级距离远的级联相机变化慢，不用每帧重画，直接复用历史帧，减少更新写入带宽和4倍的DP消耗**。  
开始绘制
<img width="2557" height="1174" alt="image" src="https://github.com/user-attachments/assets/30fe1e40-c34f-4eb9-b078-906aa0b9d62f" />
结束绘制
<img width="2556" height="1206" alt="image" src="https://github.com/user-attachments/assets/8d89be28-fe8b-410a-8f6c-5b36bc0dc86d" />
### BasePass
#### 整体过程
鸣潮移动端沿用延迟渲染管线，BasePass将场景不透明物体几何渲染进GBuffer，**一共输出6张GBufferRT + 1张 Depth**。  
其中，BasePass部分也是 PSO 切换大头，**总共约168次渲染状态切换，263次DP提交**。  
角色部分在GBuffer阶段有特殊标记，供后续LightPass阶段进行单独的NPR光照计算。 
<img width="1986" height="173" alt="image" src="https://github.com/user-attachments/assets/aeeaa973-803c-45c9-b781-9b1d8e4831fe" />
#### 植被的DepthPrePass 
开始提前绘制场景所有植被的PreDepth，减少后续绘制植被的OverDraw（镂空直接来自叶片网格几何），**深度统一存入D24S8的深度图**。
<img width="2558" height="1346" alt="image" src="https://github.com/user-attachments/assets/c885e81b-7c36-49d4-b9e3-4021bc34bd1c" />
#### GBuffer Pass
按照不同材质属性，分别输出不同的GBuffer数据。这里可以参考鸣潮在 UE 官方上的分享，链接点此处。  
（待补充）
### 构建HZD + GPU Driven遮挡剔除
对BasePass输出的深度图进行处理，把硬件深度转换为线性深度（R32F），**再降到半分辨率，构建一条分辨率从512x512—>4x4的Hi-Z深度金字塔，共8 级R16F精度的深度图**，加速后续的屏幕空间效果计算。  
移动端优化：转化的线性深度图降低一半分辨率（960x540）处理，转为R16F精度存储。  
**第一级Hi-Z深度图（512x512）**
<img width="2549" height="1167" alt="image" src="https://github.com/user-attachments/assets/0a4786a2-eac2-4b65-a5ac-51b2544c6db8" />  
接下来做GPU-Driven遮挡剔除的前置处理。  
**首先CPU提交物体包围盒数据到GPU**。14次vkCmdCopyBufferToImage，将物体包围盒数据存入贴图，每帧提交128x128最多16384个物体实例的包围和数据，精度RGBA32F。
<img width="2547" height="1177" alt="image" src="https://github.com/user-attachments/assets/95bd474e-2b8c-48d3-9516-3935f6afa4ab" />  
**再把前置计算的8级HZD深度打包成512x1024的图集**。方便后续GPU-Driven遮挡剔除一次绑定访问所有HZD深度层级，减少读取带宽。  
<img width="2551" height="1172" alt="image" src="https://github.com/user-attachments/assets/e734047a-7fcc-4292-b46e-64eaca643f4b" />  
最后，**计算依据HZB Atlas的GPU-Driven的遮挡剔除**。仅依靠GPU驱动，用贴图内存储的物体包围盒数据与HZB Atlas的深度进行对比测试（越大的包围盒取越高一级的Atlas层级），用一次全屏DP消耗，提前判定哪些物体会被遮挡，避免后续进入昂贵的着色，削减DP和OverDraw。  
<img width="2558" height="1178" alt="image" src="https://github.com/user-attachments/assets/658a26fc-1d43-46ce-ba6f-7ea44d54639a" />
### LightPass
#### 屏幕阴影Mask合成
在正式利用GBuffer计算光照前，将这一帧**所有的阴影源合成到一张屏幕半分辨率（960x540），精度R8G8B8A8的多通道屏幕阴影RT上**。所有DP共用一张屏幕半分辨率的R32F的深度图作为输入，用来重建每个像素的世界空间位置。  
**R通道：存入主光级联阴影**  
<img width="2556" height="1174" alt="image" src="https://github.com/user-attachments/assets/df2d6d1f-d190-42f2-9524-971420aea51c" />  
**G通道：存入每个角色独立包围盒计算的独立阴影**  
<img width="2558" height="1164" alt="image" src="https://github.com/user-attachments/assets/81eaf83a-2dd5-461b-a8c3-858e74e9cea1" />
这里跟最开始DepthPass更新的角色阴影呼应，**这一帧一共绘制了5张角色阴影图，4张直接复用之前帧的结果，只重画了一张，按需更新**。  
除此之外，鸣潮移动端还做了**按重要性或者距离区分的阴影贴图分辨率分配+图集打包**，近处/重要/屏占比大的角色独占一张512，较远/次要/屏占比小的角色4个打包进一张512，两项优化，节省显存和带宽消耗。  
**角色阴影图Atlas**  
<img width="1021" height="1022" alt="image" src="https://github.com/user-attachments/assets/4711bc0c-ea80-479d-ab58-39801676d1df" />






















