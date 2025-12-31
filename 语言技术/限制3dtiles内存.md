# 限制3dtiles内存

在 Cesium 中，没有一个单一的参数叫“最大文件大小”，因为 3D Tiles 是流式加载的（按需加载）。导致浏览器崩溃通常是因为**显存（VRAM）耗尽**或者**内存溢出**。

要解决崩溃问题并限制资源占用，你需要通过配置 `Cesium3DTileset` 的参数来控制**缓存大小**和**加载精度**。

以下是具体的解决方案，按重要性排序：

## 1. 限制显存缓存大小 (`maximumMemoryUsage`)

这是最直接对应你需求的参数。它决定了 Cesium 在开始卸载不可见或低优先级的瓦片之前，允许 3D Tiles 占用的显存大小（单位是 MB）。

*   **默认值**：512 (MB)。
*   **如何调整**：虽然默认是 512，但如果你的场景非常大，或者你的显卡显存较小，Cesium 的垃圾回收机制可能来不及清理。
    *   **尝试降低**：如果崩溃是因为瞬间内存峰值过高，尝试设为 **256** 或更低，强迫 Cesium 更频繁地卸载旧瓦片。
    *   **注意**：如果你设得太高（例如 4096），虽然能减少闪烁，但更容易导致浏览器崩溃。

```javascript
const tileset = await Cesium.Cesium3DTileset.fromUrl('path/to/tileset.json', {
    // 限制缓存为 256MB，强迫更积极地卸载旧瓦片
    maximumMemoryUsage: 256, 
});
viewer.scene.primitives.add(tileset);
```

## 2. 降低加载精度 (`maximumScreenSpaceError`)

这是最有效的防止崩溃的方法。它控制了加载瓦片的精细程度。

*   **原理**：数值越大，加载的瓦片越粗糙（层级越低），加载的数据量越少。数值越小，细节越丰富，数据量越大。
*   **默认值**：16。
*   **建议**：如果模型太大导致崩溃，将此值调大（例如 **32**, **64**, 甚至 **128**）。这会显著减少同屏渲染的三角形数量和纹理大小。

```javascript
const tileset = await Cesium.Cesium3DTileset.fromUrl('path/to/tileset.json', {
    maximumMemoryUsage: 256,
    
    // 默认是16，设为64会显著降低细节，但能大幅减少内存占用，防止崩溃
    maximumScreenSpaceError: 64, 
});
```

## 3. 启用“跳级加载”优化 (`skipLevelOfDetail`)

如果你的数据层级很深，Cesium 默认会逐级加载（L1 -> L2 -> L3...），这会占用大量带宽和内存。开启跳级加载可以直接加载最终需要的层级。

```javascript
const tileset = await Cesium.Cesium3DTileset.fromUrl('path/to/tileset.json', {
    maximumMemoryUsage: 256,
    maximumScreenSpaceError: 64,
    
    // 开启跳级加载优化
    skipLevelOfDetail: true,
    baseScreenSpaceError: 1024,
    skipScreenSpaceErrorFactor: 16,
    skipLevels: 1,
    
    // 立即卸载不可见的瓦片（节省显存）
    immediatelyLoadDesiredLevelOfDetail: false,
    loadSiblings: false 
});
```

## 4. 动态调整分辨率 (`resolutionScale`)

如果上述方法仍然卡顿或崩溃，可以通过降低 Viewer 的渲染分辨率来减轻 GPU 压力。

```javascript
// 0.5 表示分辨率降低一半，性能大幅提升，虽然画面会变糊
viewer.resolutionScale = 0.5; 

// 开启自动调整（可选）
viewer.scene.requestRenderMode = true; 
```

## 5. 如果依然崩溃：数据源问题

如果在调整了 `maximumMemoryUsage` 和 `maximumScreenSpaceError` 后依然崩溃，通常说明**数据本身有问题**：

1.  **根节点过大**：有些 3D Tiles 制作不规范，Root Tile（根节点）本身就有几百 MB 的几何体或纹理。这种情况下，无论你怎么设置参数，只要加载就会崩。你需要使用工具（如 Cesium ion 或 3d-tiles-tools）重新切片（Tiling）该数据。
2.  **纹理未压缩**：检查纹理是否是 WebP 或 CRN/KTX2 格式。如果全是巨大的 PNG/JPG，显存极易爆炸。

## 综合配置代码示例

建议你使用以下配置进行测试，这组配置是针对“巨型模型防崩溃”的保守设置：

```javascript
try {
    const tileset = await Cesium.Cesium3DTileset.fromUrl('http://your-server/tileset.json', {
        // --- 核心防崩溃设置 ---
        maximumMemoryUsage: 256,       // 限制显存缓存 (MB)
        maximumScreenSpaceError: 64,   // 降低细节精度 (越大越粗糙，但越安全)
        
        // --- 性能优化设置 ---
        skipLevelOfDetail: true,       // 开启跳级加载
        baseScreenSpaceError: 1024,
        skipScreenSpaceErrorFactor: 16,
        skipLevels: 1,
        
        // --- 激进的内存回收 ---
        cullWithChildrenBounds: true,  // 优化视锥体剔除
        dynamicScreenSpaceError: true, // 动态调整SSE，运动时降低质量
        dynamicScreenSpaceErrorDensity: 0.00278,
        dynamicScreenSpaceErrorFactor: 4.0,
        dynamicScreenSpaceErrorHeightFalloff: 0.25
    });

    viewer.scene.primitives.add(tileset);
    
    // 定位到模型
    viewer.zoomTo(tileset);

} catch (error) {
    console.error(`加载 3D Tiles 失败: ${error}`);
}
```

**操作建议：**
先将 `maximumScreenSpaceError` 设得很大（比如 128），如果能加载成功不崩溃，再慢慢减小这个数值（128 -> 64 -> 32），直到找到画质和稳定性的平衡点。