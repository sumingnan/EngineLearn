# Urho3D 学习之事件消息系统

##### 简介

​		Urho3D 的事件消息机制是注册制，整个Urho3D都是使用事件来驱动整个引擎的更新和渲染，只要你继承于**Urho3D::Object**类就可以注册事件和发送事件。

##### 实现

​		在Urho3D里面，主要实现的载体类有：**Urho3D::Object**、**Urho3D::Context**这两个类，前者用来实现消息发送等功能封装实现，后者是全局唯一类（可以认为是单例）用来保存事件发送、接收的实例队列。

​		在Urho3D中，任何类（只要继承于**Urho3D::Object**）就可以注册消息，主要有两种类型：

1. 接收来自特定发送者的消息：

```c++
SubscribeToEvent(model, E_RELOADFINISHED, URHO3D_HANDLER(AnimatedModel, HandleModelReloadFinished));
```

第一个参数：此发送者发送消息才会响应；

第二个参数：消息类型

第三个参数：回调函数

主要实现代码：

```c++
void Object::SubscribeToEvent(Object* sender, StringHash eventType, EventHandler* handler)
{
    // If a null sender was specified, the event can not be subscribed to. Delete the handler in that case
    if (!sender || !handler)
    {
        delete handler;
        return;
    }

    handler->SetSenderAndEventType(sender, eventType);
    // Remove old event handler first
    EventHandler* previous;
    EventHandler* oldHandler = FindSpecificEventHandler(sender, eventType, &previous);
    if (oldHandler)	//如果已有注册
    {
        eventHandlers_.Erase(oldHandler, previous);//将旧的删除
        eventHandlers_.InsertFront(handler);//插入新的消息函数
    }
    else
    {
        eventHandlers_.InsertFront(handler);
        context_->AddEventReceiver(this, sender, eventType);//加到context对象的消息队列中
    }
}
```

2. 接收所有发送该事件的消息接收者：就是不管谁发送该消息，我都接收并处理该事件

   ```c++
   SubscribeToEvent(E_RENDERUPDATE, URHO3D_HANDLER(Audio, HandleRenderUpdate));
   ```

   以上第一个参数：事件类型

   第二个参数：消息回调函数

   具体的实现代码如下：

   ```c++
   void Object::SubscribeToEvent(StringHash eventType, EventHandler* handler)
   {
       if (!handler)
           return;
   
       handler->SetSenderAndEventType(nullptr, eventType);
       // Remove old event handler first
       EventHandler* previous;
       EventHandler* oldHandler = FindSpecificEventHandler(nullptr, eventType, &previous);
       if (oldHandler)	//如果已有注册
       {
           eventHandlers_.Erase(oldHandler, previous);
           eventHandlers_.InsertFront(handler);
       }
       else
       {
           eventHandlers_.InsertFront(handler);
           context_->AddEventReceiver(this, eventType);//添加到消息队列中
       }
   }
   ```

   

   那当我们发送消息会发生什么呢？

   ```c++
   // Rendering update event
   SendEvent(E_RENDERUPDATE, eventData);
   ```

   以上就是用来发送一个渲染更新的事件，那么发送这事件到底做了什么呢？代码如下：

   ```c++
   void Object::SendEvent(StringHash eventType, VariantMap& eventData)
   {
     //主线程判断
       if (!Thread::IsMainThread())
       {
           URHO3D_LOGERROR("Sending events is only supported from the main thread");
           return;
       }
   
       if (blockEvents_)
           return;
   
       // Make a weak pointer to self to check for destruction during event handling
       WeakPtr<Object> self(this);
       Context* context = context_;
       HashSet<Object*> processed;
   
       context->BeginSendEvent(this, eventType);
   
       // Check first the specific event receivers
       // Note: group is held alive with a shared ptr, as it may get destroyed along with the sender
     //首先去获取那些需要特定发送者的消息
       SharedPtr<EventReceiverGroup> group(context->GetEventReceivers(this, eventType));
       if (group)
       {
           group->BeginSendEvent();
   
           const unsigned numReceivers = group->receivers_.Size();
           for (unsigned i = 0; i < numReceivers; ++i)
           {
               Object* receiver = group->receivers_[i];
               // Holes may exist if receivers removed during send
               if (!receiver)
                   continue;
   					//执行回调函数
               receiver->OnEvent(this, eventType, eventData);
   
               // If self has been destroyed as a result of event handling, exit
               if (self.Expired())
               {
                   group->EndSendEvent();
                   context->EndSendEvent();
                   return;
               }
   
               processed.Insert(receiver);
           }
   
           group->EndSendEvent();
       }
   
       // Then the non-specific receivers
     	// 然后去获取那些未设置特定发送者的消息队列
       group = context->GetEventReceivers(eventType);
       if (group)
       {
           group->BeginSendEvent();
   
           if (processed.Empty())	//如果没有特定发送消息队列
           {
               const unsigned numReceivers = group->receivers_.Size();
               for (unsigned i = 0; i < numReceivers; ++i)
               {
                   Object* receiver = group->receivers_[i];
                   if (!receiver)
                       continue;
   							//执行回调函数
                   receiver->OnEvent(this, eventType, eventData);
   
                   if (self.Expired())
                   {
                       group->EndSendEvent();
                       context->EndSendEvent();
                       return;
                   }
               }
           }
           else
           {
               // If there were specific receivers, check that the event is not sent doubly to them 如果有特定发送者，检查一下是否有重复，防止重复发送
               const unsigned numReceivers = group->receivers_.Size();
               for (unsigned i = 0; i < numReceivers; ++i)
               {
                   Object* receiver = group->receivers_[i];
                   if (!receiver || processed.Contains(receiver))
                       continue;
   
                   receiver->OnEvent(this, eventType, eventData);
   
                   if (self.Expired())
                   {
                       group->EndSendEvent();
                       context->EndSendEvent();
                       return;
                   }
               }
           }
   
           group->EndSendEvent();
       }
   
       context->EndSendEvent();
   }
   ```

   

   ##### 总结

   可以看到，其实主要的实现都是在基类的Object中，所以在Urho3D中，Object和Context，几乎每个对象都有持有。



