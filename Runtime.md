---
title: Runtime
date: 2020-06-08 09:51
---

# ##1、runtime的内存模型（isa、对象、类、metaclass、结构体的存储信息等）
OC中几乎所有的类都继承自 **NSObject** ，OC的动态性也是通过NSObject实现的。
**NSProxy** 是少数不继承自 NSObject 的类

NSObject 类定义：NSObject 里有一个指向 Class 的 isa
**Class 是一个 objc_class类型的结构体**
objc_class 代表类对象， objc_object 代表实例对象， objc_object 的 isa 指向 objc_class 
**结论**：实例对象的isa是指向该类的类对象的。类对象的 isa 只想元类对象

**元类（meta class）**

>  * 元类的isa均指向根元类，根元类指向自己
>  * 根元类继承根类（NSObject）

### **objc_class 和 objc_object**
```Objective-C
OC - 2.0
typedef struct objc_class *Class;
typedef struct objc_object *id;

@interface Object { 
    Class isa; 
}

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

struct objc_object {
private:
    isa_t isa;
}

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;
}
```

``` Objective-C
OC_1.0
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```
实例对象、类对象、元类对象经典关系图：
> 实例对象的实力放在存在类对象的方法列表中，类方法函数存在metaclass结构体中
![](https://upload-images.jianshu.io/upload_images/967869-f15ff2bc1baf88d2.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

### 2、为什么要设计metaclass
* 类对象、元类对象能够复用消息发送流程机制
* 单一职责原则

* 对象的实例方法调用时，通过对象的 isa 在类中获取方法的实现。
* 类对象的类方法调用时，通过类的 isa 在元类中获取方法的实现。

meta-class之所以重要，是因为它存储着一个类的所有类方法。每个类都会有一个单独的meta-class，因为每个类的类方法基本不可能完全相同

> objc_msgSend(void /* id self, SEL op, ... */ ) 复用消息通道，类方法也可以放在Class里，但发送消息时，需要增加一个参数

### 3、class_copyIvarList&class_copyPropertyList区别
**class_copyIvarList** 获取类对象中的所有实例变量信息
**class_copyPropertyList** 获取类对象中的属性信息

### 4、category如何被加载的，两个category的load方法的加载顺序，两个category的同名方法的加载顺序
category 的属性总是在前面的，baseClass的属性被往后偏移

### 5、category和extension区别，可以给NSObject添加Extension吗
**category:**
* 运行时添加分类属性/协议/方法
* 分类添加的方法会“覆盖”原类方法，因为方法查找的话是从头至尾，一旦查找到了就停止了
* 同名分类方法谁生效取决于编译顺序，image 读取的信息是倒叙的，所以编译越靠后的越先读入
* 名字相同的分类会引起编译报错；

**extension:**
* 编译时决议
* 只以声明的形式存在，多数情况下就存在于 .m 文件中；
* 不能为系统类添加扩展

### 6、消息转发机制，消息转发机制和其他语言的消息机制优劣对比
**优点：**更加灵活。
**缺点：**增加损耗。
### 7、在方法调用的时候，方法查询-> 动态解析-> 消息转发 之前做了什么
![](https://upload-images.jianshu.io/upload_images/9181332-ac568261ffa3cf90.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200)
从全局来看，消息转发机制共分为3大步骤：
1.Method resolution 方法解析处理阶段
第一阶段 咨询接收者是否可以动态添加方法
 *   +(BOOL)resolveInstanceMethod:(SEL)sel
 * +(BOOL)resolveClassMethod:(SEL)sel
2.Fast forwarding 快速转发阶段
第二阶段：询问是否有其他对象可以处理
 * - (id)forwardingTargetForSelector:(SEL)aSelector
3.Normal forwarding 常规转发阶段
* -(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
* -(void)forwardInvocation:(NSInvocation *)anInvocation

### 8、IMP、SEL、Method的区别和使用场景
* SEL：一个selector的指针
* IMP：IMP即Implementation，为指向函数实现的指针，如果我们能够获取到这个指针，则可以直接调用该方法，充分证实了它就是一个函数的指针
* Method：objc_method 类型的结构体

### 9、load和initialize方法的区别是什么？在继承关系中他们有什么区别
>  调用方式
    * load函数直接调用
    * initialize是通过objc_msgSend调用
>  调用时刻
    * load是在程序初始化的时候调用。(只调用一次）
    * initialize在类第一次接收到消息的时候调用
>  调用顺序
    **load**
先调用类中的load
先编译的类先调用load
在调用子类的load之前，会先调用父类的
后调用category中的load
先编译的先调用

        **initialize**
父类先于子类调用
category会覆盖本类中的initialize
子类没实现会调用父类的，所以父类的initialize可能调用多次

### 10、关联对象
关联对象都由AssociationsManager管理

```Objective-C
- (void)setName:(NSString *)name{
    objc_setAssociatedObject(self, "name", name, OBJC_ASSOCIATION_COPY);
}

- (NSString*)name{
    NSString *nameObject = objc_getAssociatedObject(self, "name");
    return nameObject;
}

```


### 11、怎么理解Objective-C是动态运行时语言
* 将数据类型的确定由编译时,推迟到了运行时
* 运行时机制使我们直到运行时才去决定一个对象的类别,以及调用该类别对象指定方法

