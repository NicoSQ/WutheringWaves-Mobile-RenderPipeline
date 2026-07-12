# 鸣潮移动端渲染管线截帧分析  
## 截帧环境 
MuMu模拟器/小米14Ultra/画质高/分辨率1920x1080，基于Rendedoc学习下鸣潮移动端渲染管线。
## 渲染管线分析
### DepthPass
#### 角色阴影图
计算角色专属的高精度阴影图，独立于场景CSM阴影，移动端更加节省带宽。**阴影图大小512x512，精度D16**。用角色的LOD2模型做代理模型渲染，顶点变换（含骨骼Skinning）与光栅化开销直接减少一个数量级。  
<img width="2560" height="1313" alt="image" src="https://github.com/user-attachments/assets/5845f302-3d1b-4e4c-82e5-8c787a4c2105" />
存在历史帧的角色阴影图复用，结合后面LightPass的屏幕空间阴影Mask，这一帧只更新了一张NPC的角色阴影图。
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
按照不同材质属性，分别输出不同的GBuffer数据。这里可以详细信息可以参考一下**鸣潮在UE官方上的分享**，链接在下方（PS：官方分享到后面去掉了Toon模式的GBuffer，但是最新版截帧看好像还是存在的）。  
**https://www.bilibili.com/video/BV1BK411v7FY/?spm_id_from=333.337.search-card.all.click&vd_source=fadff3dd5afa82456b1c99decd7cc54a**  
<img width="2015" height="1028" alt="image" src="https://github.com/user-attachments/assets/2c89f4f0-aebc-4495-884d-b6239674f88b" />  
其中，GBufferA中B通道直接光模式按分享会说法是计算云层投影的。  
**GBufferA(R8G8B8A8)**  
<img width="2547" height="1173" alt="image" src="https://github.com/user-attachments/assets/9172144e-536f-4132-ab05-89a656889cab" />  
**GBufferB(R8G8B8A8)**  
<img width="2555" height="1169" alt="image" src="https://github.com/user-attachments/assets/f4638422-3dba-49d1-acce-34e5c9a0662c" />  
**GBufferC(R8G8B8A8_SRGB)**  
<img width="2552" height="1174" alt="image" src="https://github.com/user-attachments/assets/0ab552b1-a563-45e6-8425-3baf6f5321d1" />

**不同材质写入的Stencil值，后续LightPass根据不同模板值计算不同光照分支，以下仅记录当前帧**  
| 植被 | 建筑/普通物品 | 地形 | 近处角色 | 远处角色 |
|------|---------------|----- |--------- | --------- |
| 0010 0000 | 1010 0000 | 0011 0000 | 0000 1010 | 0000 0010 |
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

<img width="465" height="358" alt="image" src="https://github.com/user-attachments/assets/b5945d8c-ab5d-4e40-b1b7-65f0a53aa548" />  

**B通道：采样一张sRGB的256x256的噪波图计算云层投影阴影**
<img width="2556" height="1175" alt="image" src="https://github.com/user-attachments/assets/48ad490e-71f1-485b-abcd-a7325c674db3" />  

**A通道：存入屏幕空间AO（GTAO）**  
<img width="2553" height="1175" alt="image" src="https://github.com/user-attachments/assets/8801f111-f5d5-49fb-b337-ca689af47a16" />  
#### SSR屏幕空间反射  
按材质选择性反射，只让金属/水面进入SSR反射计算。这一帧几乎都是建筑，地面，大多数像素被跳过。  
详细待补充。

### 延迟光照着色
每个全屏光照DP通过不同Stencil参考值计算不同光照分支（**Stencil Func = Equal，Blend One One**），输出的光照计算RT，分辨率1920x1080，R11G11B10_FLOAT格式。  
<img width="1361" height="413" alt="image" src="https://github.com/user-attachments/assets/f15cb940-e27b-4c93-b8da-04cb2eec0f23" />  
**不同模板值绘制不同光照分支如下**  
| 植被光照结果  | 植被StencilTest  |
|---|---|
| <img width="534" height="298" alt="image" src="https://github.com/user-attachments/assets/50062fea-0de4-4e08-94bf-19f049720205" /> | <img width="534" height="298" alt="image" src="https://github.com/user-attachments/assets/46a1503d-245e-4b94-8a3d-36702106eaf8" />|  

| 建筑/普通物品光照结果 | 建筑/普通物品StencilTest  |
|---|---|
| <img width="534" height="298" alt="image" src="https://github.com/user-attachments/assets/354cf2d8-20b5-4eca-8357-cb7966572646" /> |  <img width="534" height="298" alt="image" src="https://github.com/user-attachments/assets/5e7b6d88-5c4c-452c-8f40-f70a96f85288" /> |  

| 地形光照结果 | 地形StencilTest  |
|---|---|
| <img width="534" height="298" alt="image" src="https://github.com/user-attachments/assets/9fe82884-8967-4fd3-bad7-a409f7a97fba" />|  <img width="534" height="298" alt="image" src="https://github.com/user-attachments/assets/12bf0c0b-3738-4085-a150-66e31c0e4d84" />|  

| 角色光照结果 | 角色StencilTest  |
|---|---|
| <img width="534" height="298" alt="image" src="https://github.com/user-attachments/assets/37ab34b8-621c-4c9e-b6eb-767189278a98" />| <img width="534" height="298" alt="image" src="https://github.com/user-attachments/assets/7cd472ae-705a-48a6-beb7-8e786d4386d0" />|  

#### 水面渲染/高度雾
光照部分最后阶段进行场景水面和高度雾的渲染。  
水面用输入一张全分辨率的场景深度和半分辨率的场景颜色作为输入，用来计算岸边软边过渡和水面镜面反射。  
水体的网格专门做了岸边过渡的UV处理，应该也是为了更好计算岸边白沫效果。  
<img width="2560" height="1313" alt="image" src="https://github.com/user-attachments/assets/d5b00f00-62ac-4d6a-8014-798c5a8d531e" />  
高度雾为输入四个顶点的全屏mesh计算，混合模式设置为One+SrcAlpha，比较典型基于深度的全屏高度雾计算叠加。  
<img width="2021" height="66" alt="image" src="https://github.com/user-attachments/assets/e57ead35-fcd6-40ea-ab8f-eb9b65041fd0" />  
<img width="2552" height="1138" alt="image" src="https://github.com/user-attachments/assets/bca7b65b-c568-4d62-be21-02af6e484939" />  

### Translucent  
#### 特效计算
接下来一段是标准的半透明/特效计算阶段，走前向渲染，基本基于两种混合模式。计算结果输出到HDR的场景光照颜色RT上。  
<img width="1467" height="67" alt="image" src="https://github.com/user-attachments/assets/1c99cccb-d177-4a38-be82-2b9331b19ea8" />  
<img width="1473" height="70" alt="image" src="https://github.com/user-attachments/assets/a6cacd44-2682-49d8-8611-367caaaa6109" />  

#### 云层计算
在半透最开始阶段，基于半椭球形的天空盒，首先渲染天空云层效果。  
**云层输入贴图**  
<img width="274" height="642" alt="image" src="https://github.com/user-attachments/assets/60fba410-aaea-4fae-9baa-9025c78e5c8f" />   
| 天空盒球体 | 云层渲染效果  |
|---|---|
| <img width="640" height="328" alt="image" src="https://github.com/user-attachments/assets/bb9c0e15-8341-4728-80db-7deec1404529" />  | <img width="640" height="328" alt="image" src="https://github.com/user-attachments/assets/7a498fe5-051f-4dbe-8747-31df56c74c46" /> |

### 屏幕后处理/2D UI合成
#### 整体链路
最后阶段进行全屏的屏幕后处理。链路包括：TAA抗锯齿计算 + Bloom合成 + 自动曝光 + 计算ColorGradingLUT + Tonemap最终合成（HDR->LDR）。  
**最终输出**  
<img width="2560" height="1313" alt="image" src="https://github.com/user-attachments/assets/7bf5089d-08b3-478f-b169-a77f6ba5a4ce" />
#### TAA抗锯齿 
传统TAA抗锯齿计算作抗锯齿计算。**SceneColor + 历史帧（上一帧TAA输出结果） + MotionVector（GBuffer阶段输出）+ 深度 + Mask 作为输入**，输出精度R11G11B11，分辨率为1920x1080的HDR屏幕图像，作为最后后处理合成的基础。  
上面链接的官方分享会有提到做过的TAAU的过程和一些效果优化方案。  
<img width="543" height="302" alt="image" src="https://github.com/user-attachments/assets/409a5803-6f7d-4071-a515-9b1df058d80d" />

| MotionVector输入 | TAA输出结果  |
|---|---|
| <img width="796" height="447" alt="image" src="https://github.com/user-attachments/assets/f8e315f4-5f3f-4a45-b482-649a355490fe" /> | <img width="796" height="447" alt="image" src="https://github.com/user-attachments/assets/3bfd7045-bb10-447b-9471-05ef623212e8" /> |  

#### Bloom
上一步TAA输出结果作为Bloom计算的输入。先降采样提取屏幕超过阈值的亮度，再搭建4级高斯模糊金字塔（480x270/240x135/120x68/30x17），每一级金字塔实现一次H，V方向上的高斯模糊计算，为后续Bloom合并做准备。  
**最终Bloom合成（各级金字塔合成一张480x270的Bloom RT）**  
<img width="2560" height="1313" alt="image" src="https://github.com/user-attachments/assets/00dd8edf-1501-4072-89f6-3ec512ac3863" />  

#### LUT/最终合成  
最终合成前，计算输出R10G10B10A2精度的LUT贴图。  
<img width="396" height="304" alt="image" src="https://github.com/user-attachments/assets/dab9fe35-bbb5-4b62-a7d5-266ebec63ec9" />  

**最终合成输入：TAA输出的HDR场景色 + Bloom合并结果 + LUT + 深度 + 曝光值，转换为最终的LDR屏幕图像（R11G11B10 ——> R8G8B8A8）**  
<img width="2560" height="1313" alt="image" src="https://github.com/user-attachments/assets/044664e0-dbaf-4e86-99bb-6f48a9a31a6f" />  

#### 2D UI叠加
Tonemap输出LDR后2D UI直接和RT叠加，节省一次移动端全屏RT写回和读取的带宽消耗。  
<img width="2551" height="1111" alt="image" src="https://github.com/user-attachments/assets/74e36ece-0680-437e-a4b4-605231c86b3b" />


































