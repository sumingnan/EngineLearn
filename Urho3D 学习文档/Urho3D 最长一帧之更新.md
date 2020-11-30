#### Urho3D 最长一帧之更新

在Urho3D的更新事件是在Engine::Update调用的，代码很简洁，发送四个事件，代码如下：

```c++
// Logic update event
    using namespace Update;

    VariantMap& eventData = GetEventDataMap();
    eventData[P_TIMESTEP] = timeStep_;
    SendEvent(E_UPDATE, eventData);

    // Logic post-update event
    SendEvent(E_POSTUPDATE, eventData);

    // Rendering update event
    SendEvent(E_RENDERUPDATE, eventData);

    // Post-render update event
    SendEvent(E_POSTRENDERUPDATE, eventData);
```

分别对应 引擎更新事件、引擎更新后事件、渲染更新事件、渲染后更新事件，下面分别介绍这些事件所做的事情。

##### E_UPDATE 引擎更新事件

1. 材质属性动画更新事件

   shader属性动画，用来做着色器中的逐帧属性动画（值动画）；

2. 场景更新事件

   

3. 精灵更新事件

4. BillBoard 广告板更新事件

5. 贴花更新事件

##### E_POSTUPDATE 引擎更新后事件

1. 控制台的更新事件
2. 调试框hub更新事件
3. UI更新事件
4. UI元素的更新事件

##### E_RENDERUPDATE 引擎渲染更新事件

1. 音频Audio的更新事件
2. 八叉树的更新事件
3. 渲染器的更新事件
4. UI的更新事件

##### E_POSTRENDERUPDATE 引擎渲染后更新事件

1. 骨骼动画组建的更新事件
2. 广告板的更新事件
3. 贴合更新事件
4. IK更新事件