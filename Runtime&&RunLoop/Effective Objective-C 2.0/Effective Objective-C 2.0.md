# Effective Objective-C 2.0 

## 第一章



## 第二章

1. 理解“属性”这一概念

属性（property）是用来封装对象中属性的。Objective-C通常会把其所需要的数据保存各种**实例变量**。看如下的一段代码：

```objective-c
@interface EOCPerson: NSObject {
  @public
    NSString *_firstName;
  	NSString *_lastName;
  @private
    NSString *_someInternalData;
}
```

从Java，C++转过来的程序员比较熟悉这种写法，这些语言中可以定义实例变量的作用域。而编写Objective-C的代码却很少这样做。这样写有个问题：**对象布局在编译器（compile time）就固定了。只要碰到访问_firstName变量的代码，编译器就把他替换成“偏移量”（offset），这个偏移量是“硬编码”（hardCode），表示该变量距离放对象的内存区域的起始地址有多远。**这样看表面上没有问题，但是如果又添加一个实例变量，比如添加一个_dateOfBirth实例变量

```objective-c
@interface EOCPerson: NSObject {
  @public
    NSDate *_dateOfBirth;
    NSString *_firstName;
  	NSString *_lastName;
  @private
    NSString *_someInternalData;
}
```

原来的_firstName 的偏移量现在指向了 _dateOfBirth。

![2-1](/Users/X-Liang/Documents/面试-Invocation/Runtime&&RunLoop/Effective Objective-C 2.0/2-1.png)

如果使用编译器确定的偏移量，那么在修改完类定义后要重新编译才行，否则就会出错。