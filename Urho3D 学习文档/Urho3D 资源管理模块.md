### Urho3D 资源管理模块/文件系统模块

​		Urho3D的资源管理模块相对其他引擎来说还是比较简单，最主要的两个类是：**ResourceCache**、**Resource**，首先ResourceCache用来管理所有资源的加载、删除、重新加载的管理类，它是作为引擎的一个子系统存在的；而Resource是资源的一个基类，它有两个基本的函数是：BeginLoad和EndLoad，不同的资源加载都是通过加载这两个虚函数来实现不同的加载方式。

​		Urho3D的文件系统模块，也是一个子系统，叫**FileSystem**文件系统，主要用来文件、目录的操作和处理控制，比如说创建目录、复制、重命名、删除文件等。而在文件操作的一个基本类是：**File**，主要用来服务于文件系统，用来实现不同系统下的文件读取、文件存储等操作。

​		以下分别来说明这两大模块的设计思路以及之间的关系：

#### Urho3D 资源管理系统

​		在Urho3D中，任何类型的资源都必须继承于Resource，根据类型不同，可分别重载其中的BeginLoad和EndLoad方法，首先其中的BeginLoad就是用来解析资源的方法，它可以**在主线程加载，也可以在子线程加载（预步加载方法）**，而EndLoad就是用来加载完之后所需要做的事情，比如说，模型数据加载完毕，这个时候可能需要创建响应的buffer，**和BeginLoad不同的是，EndLoad必须在主线程加载**.

##### 预步加载资源

​		在Urho3D有一个专门用来处理预步加载的类叫：**BackgroundLoader**，在ResourceCache里面持有这个对象（也只有ResourceCache才能持有）。如果你想预步加载，那你可以使用**ResourceCache::BackgroundLoadResource**，如果加载完毕，会发送完成事件，调用成功，会加到预步加载队列里，如果加载失败（例如已经存在），则返回false，这个接口不一定是主线程才能调用。

在这个**BackgroundLoader类**里面，相映的流程图如下：

![backGroundResource](image/backGroundResource.png)

**常用的资源类型**

Urho3D::Animation：动画资源类

Urho3D::Material：材质类

Urho3D::Model：模型类

Urho3D::ParticleEffect：粒子影响器类

Urho3D::Shader：着色器资源类

Urho3D::Texture：纹理资源类

Urho3D::Texture2D：2D纹理资源类

Urho3D::Texture2DArray：2D纹理组资源类

Urho3D::Texture3D：3D纹理资源

Urho3D::TextureCube：立方体纹理资源

Urho3D::Image：图片资源类

Urho3D::JSONFile：json资源类

Urho3D::XMLFile：XML资源类

##### ResourceCache的设计

​		在Urho3D里面，ResourceCache管理着所有的资源的创建、更新、释放和删除以及资源内存占用空间大小的统计，在Urho3D上是作为一个子系统来创建（可以认为是一个单例）。

​		ResourceCache既可以对单个资源的管理、也可以对以类型的资源管理，主要对外开放的接口有以下几种：

1. 添加/删除资源搜索目录或者添加资源包：AddResourceDir/AddPackageFile/RemoveResourceDir/RemovePackageFile
2. 删除资源
   1. 删除某个资源：ReleaseResource(StringHash type, **const** String& name, **bool** force = **false**);
   2. 删除某个类型的资源：ReleaseResources(StringHash type, **bool** force = **false**);
   3. 删除某个类型下的带有特定名称的资源：ReleaseResources(StringHash type, **const** String& partialName, **bool** force = **false**);
   4. 删除所有带有特定名称的资源：ReleaseResources(**const** String& partialName, **bool** force = **false**);
   5. 删除所有资源：ReleaseAllResources(**bool** force = **false**);
3. 重载资源/重载资源包括依赖资源/开启自动更新资源/设置某类型资源的最大空间占用（如果超过，则需要移除最旧的资源）：ReloadResource/ReloadResourceWithDependencies/SetMemoryBudget/SetAutoReloadResources
4. 获取文件：GetFile
5. 获取资源，并放入资源缓冲区中：GetResource
6. 获取临时资源，不放入资源缓冲区：GetTempResource
7. 以多线程加载资源，加载成功，会发送事件：BackgroundLoadResource
8. 资源路由功能：这个主要用来处理资源的额外处理，比如这边的**ScriptResourceRouter**，它对于后缀是as的文件会处理成asc文件。添加/删除路由：AddResourceRouter/RemoveResourceRouter

#### Urho3D 文件系统

Urho3D的文件系统，主要是用来处理IO，主要提供以下几个功能：

1. 设置当前工作目录
2. 创建目录
3. 执行系统命令/执行系统程序
4. 执行系统打开文件
5. 拷贝文件
6. 重命名文件
7. 删除文件
8. 注册路径
9. 设置文件的修改时间
10. 获取当前工作目录
11. 检查文件夹的处理权限（可读/可写/可读写）
12. 获取一个文件的最近修改时间
13. 检查是否存在该文件
14. 检查是否存在该文件夹
15. 扫描特定后缀文件的文件夹
16. 获取程序目录/用户定义目录/临时工作目录
17. 分离一个完整的目录成：文件路径名、文件名、文件后缀名
18. 获取路径/获取文件名/获取后缀

##### 文件流

**File**：主要用来存储文件的操作，主要有提供以下一些方法

1. 从文件中读取数据
2. 跳到文件中的某个位置
3. 往文件中写入数据
4. 获取文件名
5. 打开/关闭文件
6. 获取打开文件模式
7. 判断是否打开
8. 获取文件句柄

##### 内存流

**MemoryBuffer**：主要是用来操作内存中的数据流

1. 读取/写入 数据
2. 跳动内存块中的某个位置
3. 获取数据指针
4. 判断是否是只读

##### 和ResourceCache的设计对接

​		在Urho3D的ResourceCache自由管理中，和FS(fileSystem)有关联的就是在资源加载、判断是否存在某个资源，而在ResourceCache中会保存着用户定义的资源搜索路径，所以在查找资源时，会使用预定义的资源搜索和文件名遍历查找文件。

​		总的来说，在Urho3D中，ResourceCache和FileSystem紧密合作，ResourceCache负责资源的管理、而FileSystem负责资源的读取、写入和删除等。

