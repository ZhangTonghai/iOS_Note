---
title: KVO && KVC
date: 2020-06-08 11:27
---

### 1、KVO实现原理
* 为需要observed的对象动态创建一个子类，以NSKVONotifying_ 最为前缀
* 将对象的 isa 指针指向新的子类
* 重写所有要观察属性的setter方法，统一会走一个方法
* 内部是会调用 willChangeValueForKey 和 didChangevlueForKey


![](https://upload-images.jianshu.io/upload_images/861390-c6f11ee35006dd1b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1024/)

### 2、如何手动关闭KVO
```Objective-C
+(BOOL)automaticallyNotifiesObserversForKey:(NSString *)key{
    if ([key isEqualToString:@"name"]) {
        return NO;
    }else{
        return [super automaticallyNotifiesObserversForKey:key];
    }
}

-(void)setName:(NSString *)name{
    
    if (_name!=name) {
        
        [self willChangeValueForKey:@"name"];
        _name=name;
        [self didChangeValueForKey:@"name"];
    }
      
}
```
### 3、通过KVC修改属性会触发KVO吗
会

### 4、哪些情况下使用KVO会崩溃，如何避免

1、dealloc 没有移除 kvo 观察者，解决方案：创建一个中间对象，将其作为某个属性的观察者，然后dealloc的时候去做移除观察者，而调用者是持有中间对象的，调用者释放了，中间对象也释放了，dealloc 也就移除观察者了；
2、多次重复移除同一个属性，移除了未注册的观察者
3、被观察者提前被释放，被观察者在 dealloc 时仍然注册着 KVO，导致崩溃。 例如：被观察者是局部变量的情况（iOS 10 及之前会崩溃） 比如 weak ；
4、添加了观察者，但未实现 observeValueForKeyPath:ofObject:change:context:方法，导致崩溃；
5、添加或者移除时 keypath == nil，导致崩溃；


### 5、KVO的优缺点

**优点：**

* 运用了设计模式：观察者模式
* 支持多个观察者观察同一属性，或者一个观察者监听不同属性。
* 开发人员不需要实现属性值变化了发送通知的方案，系统已经封装好了，大大减少开发工作量；
* 能够对非我们创建的对象，即内部对象的状态改变作出响应，而且不需要改变内部对象（SDK对象）的实现；
* 能够提供观察的属性的最新值以及先前值；
* 用key paths来观察属性，因此也可以观察嵌套对象；
* 完成了对观察对象的抽象，因为不需要额外的代码来允许观察值能够被观察

**缺点：**

* 观察的属性键值硬编码（字符串），编译器不会出现警告以及检查；
* 由于允许对一个对象进行不同属性观察，所以在唯一回调方法中，会出现地狱式 if-else if - else 分支处理情况；
