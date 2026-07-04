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


