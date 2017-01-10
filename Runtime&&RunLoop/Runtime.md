# Runtime

1. Runtime 中最基本的结构体

   **obj**

   ```objective-c
   获得某个类的meta类
   id = objc_getMetaClass(class_getName(cls))	
   ```

   ​

   **id** ：OC中对象类型并不时编译时期就知道，而是在运行时查找的。而**id**可以表示Objective-C任意对象的类型

   ```objective-c
   struct objc_object {
     Class isa
   } *id;
   ```

   >每个对象结构体的首个成员是Class类的变量。该变量定义了对象所属的类。是一个isa指针

   **Class**

   ```objective-c
   typedef struct objc_class *Class;
   struct objc_class {
       Class isa                                 OBJC_ISA_AVAILABILITY; 
   #if !__OBJC2__
       Class super_class                         OBJC2_UNAVAILABLE; 
       const char *name                          OBJC2_UNAVAILABLE;
       long version                              OBJC2_UNAVAILABLE; 
       long info                                 OBJC2_UNAVAILABLE; 
       long instance_size                        OBJC2_UNAVAILABLE; 
       struct objc_ivar_list *ivars              OBJC2_UNAVAILABLE; 
       struct objc_method_list **methodLists     OBJC2_UNAVAILABLE;
       struct objc_cache *cache                  OBJC2_UNAVAILABLE; 
       struct objc_protocol_list *protocols      OBJC2_UNAVAILABLE; 
   #endif
   }
   ```

   方法

   ```objective-c
   Class class_getSuperclass(Class cls);  // 返回class的superclass
   BOOL class_isMetaClass(Class cls);     // 返回class是否是meta类
   ```

   ​

   **SEL**

   ```objective-c
   typedef struct objc_selector *SEL;	
   ```

   **Method**

   ```objective-c
   typedef struct objc_method *Method;
   struct objc_method {
       SEL method_name                    OBJC2_UNAVAILABLE;
       char *method_types                 OBJC2_UNAVAILABLE;
       IMP method_imp                     OBJC2_UNAVAILABLE;
   }
   ```

   方法

   ```objective-c
   SEL method_getName(Method m); // 获得 方法 的SEL(方法名)
   	得到 SEL 后如何获得 char *name 描述的方法名？？
       const char * sel_getName(SEL sel);

   IMP method_getImplementation(Method m); // 获得方法的实现

   const char * method_getTypeEncoding(Method m); // 返回 method的encoding描述字符串

   char * method_copyReturnType(Method m); // 返回 method的返回值的encoding描述字符串, 如果有返回值，必须free返回值

   unsigned int method_getNumberOfArguments(Method m); // 获得方法的参数个数
   	获得方法参数个数后常见的操作就是逐个变量，获得参数描述，那我该如何获得参数描述？？
       char * method_copyArgumentType(Method m, unsigned int index);// 获得第index个参数的encoding参数描述，之后要free掉
   ```

   **Property**

   ```objective-c
   objc_property_t: typedef struct objc_property *objc_property_t; // 他是由 objc_property 重命名过来的
   objc_property 是一个不透明的句柄
   ```

   方法

   ```objective-c
   const char * property_getName(objc_property_t property);
   objc_property_attribute_t * property_copyAttributeList(objc_property_t property, unsigned int *outCount);// 返回 objc_property_attribute_t 结构体数组


   ```

   [property 属性编码](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW1)

   **Ivar** 类中实例变量的类型

   ```objective-c
   typedef struct objc_ivar *Ivar;
   struct objc_ivar {
       char *ivar_name                   OBJC2_UNAVAILABLE; 
       char *ivar_type                   OBJC2_UNAVAILABLE; 
       int ivar_offset                   OBJC2_UNAVAILABLE; 
   #ifdef __LP64__
       int space                         OBJC2_UNAVAILABLE;
   #endif
   }	
   ```

   方法

   ```objective-c
   ivar_getName(ivar)	// 获得实例变量的名称	
   ivar_getTypeEncoding(ivar) // 获得实例变量的typeEncoding 
   ```

   [实例变量编码](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

2. 判断协议是否包含某个方法( 判断Selector是否属于某个一个Protocol )

   ```objective-c
   objc_method_description // 方法描述
   struct objc_method_description {
       SEL name;		// 方法名
       char *types;	// 方法类型字符串
   };
   ```

   判断某个Protocol是否包含某个Selector有个方法

   ```objective-c
   struct objc_method_description protocol_getMethodDescription(Protocol *p, SEL aSel, BOOL isRequiredMethod, BOOL isInstanceMethod);
   1. Protocol *p: 指定的 Protocol
   2. SEL aSel: 指定的方法(@selector(...))
   3. BOOL isRequiredMethod: 指定的 SEL 是否是一个必须实现的方法
   4. BOOL isInstanceMethod: 指定的 SEL 是否是一个对象方法
    
   如果 Protocol 包含 指定的 SEL 返回一个 方法描述结构体 objc_method_description， 否则返回 { NULL, NULL }
   ```

   具体实现：

   ```objective-c

   struct objc_method_description MethodDescriptionForSELInProtocol(Protocol *protocol, SEL sel) {
       struct objc_method_description description = protocol_getMethodDescription(protocol, sel, YES, YES);
       if (description.types) {
           return description;
       }
       description = protocol_getMethodDescription(protocol, sel, NO, YES);
       if (description.types) {
           return description;
       }
       return (struct objc_method_description){NULL, NULL};
   }
   ```

   如何实现一个协议分发器？？

   ​

   ​

3. ​

