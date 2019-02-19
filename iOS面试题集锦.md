1、OC中self和super的理解

	@implementation Son : Father
	- (id)init{
  		self = [super init];
  		if(self){
  	  		NSLog(@"%@",NSStringFromClass([self class]))
 		 }
	}
	
答案：[self class]和[super class]，接受消息的对象都是当前的 son *这个对象。
	
self是类的隐藏参数，指向当前调用方法的这个类的实例，super是一个Magic Keyword,它本质是一个编译器标识符，和self是指向的同一个消息接受者！它们两个的不同点在于super会告诉编译器，调用class这个方法时，要去父类的方法，而不是本类里的。

当使用self调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；而当使用super时，则从父类的方法列表中开始找，然后调用父类的这个方法。

这就是为什么说“不推荐在init方法中使用点语法”，如果想访问实例变量iVar应该用下划线（_iVar）,而非点语法（self.iVar）。点语法(self.iVar)的坏处就是子类有可能覆写setter。假设 Person有一个子类叫ChenPerson，这个子类专门表示那些姓“陈”的人。该子类可能会覆写lastName属性所对应的设置方法。

通过runtime验证一下super关键字的本质：

	$ clang -rewrite-objc test.m

将会转化为：

	NSLog((NSString *)&__NSConstantStringImpl__var_folders_gm_0jk35cwn1d3326x0061qym280000gn_T_main_a5cecc_mi_0, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("class")))); 
	
	NSLog((NSString *)&__NSConstantStringImpl__var_folders_gm_0jk35cwn1d3326x0061qym280000gn_T_main_a5cecc_mi_1, NSStringFromClass(((Class (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){ (id)self, (id)class_getSuperclass(objc_getClass("Son")) }, sel_registerName("class")))); 
	
从上面的代码中，我们可以发现在调用[self class]时，会转化成objc_msgSend函数。函数定义如下：

	id objc_msgSend(id self, SEL op, ...)
我们把self做为第一个参数传递进去。
而在调用[super class] 时，会转化成objc_msgSendSuper函数。函数定义如下：

	id objc_msgSendSuper(struct objc_super *super, SEL op,...)
第一个参数是objc_super这样一个结构体，定义如下：

	struct objc_super {
		__unsafe_unretained id receiver;
		__unsafe_unretained Class super_class;
	};
	
结构体有两个成员，第一个成员是reveiver,类似于上面的objc_msgSend函数第一个参数self。第二个成员是记录当前类的父类是什么。

所以，当调用[self class]时，实际先调用的是objc_msgSend函数，第一个参数是Son当前的这个实例，然后在Son这个类里面去找-(Class)class这个方法，没有，去父类Father里找，也没有，最后在NSObject类中发现这个方法。而-(Class) class的实现就是返回self的类别，故上述输出结果为Son。

objc Runtime开源代码对-(Class)class方法的实现：

	- (Class)class {
		return object_getClass(self);
	}
而当调用[super class]时，会转换成objc_msgSendSuper函数。第一步先构造objc_super结构体，结构体第一个成员就是self。第二个成员是(id)class_getSuperclass(objc_getClass("Son"))，实际该函数输出结果为Father。

第二步是去Father这个类里去找 -(Class)class，没有，然后去NSObject类去找，找到了。最后内部是使用objc_msgSend(objc_super->receiver,@selector(class))去调用，此时已经和[self class]调用相同了，故上述输出结果仍然返回Son。

2、objc runtime中关于Object & Class & Meta Class的细节

	@interface Sark : NSObject
	@end
	@implementation Sark
	@end
	int main(int argc, const char * argv[]) {
		@autoreleasepool {
			BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
			BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
			BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
			BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
			NSLog(@"%d %d %d %d",res1,res2,res3,res4);
		}
		return 0;
	}
	
结果：1 0 0 0
id 在objc.h中定义如下：

	/// A pointer to an instance of a class.(id是指向一个objc_object结构体的指针)
	typedef struct objc_object *id;

id这个struct的定义本身就带一个\*,所以我们在使用其他NSObject类型的实例时需要在前面加上\*,而使用id时不用。

objc_object在objc.h中定义如下：

	/// Represents an instance of a class
	struct objc_object {
		Class isa;
	}
	
Objective-C中的object在最后会被转换成C的结构体，而在这个struct中有一个isa指针，指向它的类别Class。

Class 在objc.h中定义：

	/// An opaque type that represents an Objective-C class.
	typedef struct objc_class *Class;
Class本身指向的也是一个C的struct objc_class。

继续看objc_class定义：

	struct objc_class {
		Class isa OBJC_ISA_AVAILABILITY;
		#if !__OBJC2__
		Class super_class				OBJC2_UNAVAILABLE;
		const char *name				OBJC2_UNAVAILABLE;
		long version					OBJC2_UNAVAILABLE;
		long info						OBJC2_UNAVAILABLE;
		long instance_size				OBJC2_UNAVAILABLE;
		struct objc_ivar_list *ivars				OBJC2_UNAVAILABLE;
		struct objc_method_list **methodLists		OBJC2_UNAVAILABLE;
		struct objc_cache *cache		OBJC2_UNAVAILABLE;
		struct objc_protocol_list *protocols		OBJC2_UNAVAILABLE;
		#endif
	}OBJC2_UNAVAILABLE;
	
该结构体中，isa指向所属Class, super_class指向父类别。
	
###参考：
[芒果iOS开发之高级面试题二
](https://blog.csdn.net/crazyzhang1990/article/details/50405627)

[刨根问底Objective－C Runtime(唐巧)
](https://blog.csdn.net/shznt/article/details/50481819)

 


