---
title: ARC(Automatic Reference Counting)简记
date: 2018-04-16 10:29:09
tags: [c, c++]
categories: Objective-C
---

## ARC概述
　　在Objective-C中采用Automatic Reference Counting(ARC)机制，让编译器([clang][3])和运行时库协助([objc4][2])来进行内存管理。

### 内存管理的思考方式

* 自己生成的对象，自己持有
* 非自己生成的对象，自己也能持有
* 不再需要自己持有的对象时释放
* 非自己持有的对象无法释放

**对象操作与Objective-C方法的对应：**

| 对象操作       | Objective-C方法                   |
|----------------|-----------------------------------|
| 生成并持有对象 | alloc/new/copy/mutableCopy 等方法 |
| 持有对象       | retain方法                        |
| 释放对象       | release方法                       |
| 废弃对象       | dealloc方法                       |

>“自己”：对象的使用环境

**苹果采用散列表(引用计数表)来管理引用计数**

```c++
static struct {
    CFLock_t lock;
    CFBasicHashRef table;
//    uint8_t padding[64 - sizeof(CFBasicHashRef) - sizeof(CFLock_t)];
} __NSRetainCounters[NUM_EXTERN_TABLES];

CF_EXPORT uintptr_t __CFDoExternRefOperation(uintptr_t op, id obj) {
    if (nil == obj) HALT;
    uintptr_t idx = EXTERN_TABLE_IDX(obj);
    uintptr_t disguised = DISGUISE(obj);
    CFLock_t *lock = &__NSRetainCounters[idx].lock;
    CFBasicHashRef table = __NSRetainCounters[idx].table;  // 取得对象对应的散列表
    uintptr_t count;
    switch (op) {
    case 300:   // increment
    case 350:   // increment, no event
        __CFLock(lock);
    CFBasicHashAddValue(table, disguised, disguised);
        __CFUnlock(lock);
        if (__CFOASafe && op != 350) __CFRecordAllocationEvent(__kCFObjectRetainedEvent, obj, 0, 0, NULL);
        return (uintptr_t)obj;
    case 400:   // decrement
        if (__CFOASafe) __CFRecordAllocationEvent(__kCFObjectReleasedEvent, obj, 0, 0, NULL);
    case 450:   // decrement, no event
        __CFLock(lock);
        count = (uintptr_t)CFBasicHashRemoveValue(table, disguised);
        __CFUnlock(lock);
        return 0 == count;
    case 500:
        __CFLock(lock);
        count = (uintptr_t)CFBasicHashGetCountOfKey(table, disguised);
        __CFUnlock(lock);
        return count;
    }
    return 0;
}
```
[CF-1153.18/CFRuntime.c][1]

**CFBasicHashRef**

```c++
CF_PRIVATE CFBasicHashRef CFBasicHashCreate(CFAllocatorRef allocator, CFOptionFlags flags, const CFBasicHashCallbacks *cb) {
    size_t size = sizeof(struct __CFBasicHash) - sizeof(CFRuntimeBase);
    if (flags & kCFBasicHashHasKeys) size += sizeof(CFBasicHashValue *); // keys
    if (flags & kCFBasicHashHasCounts) size += sizeof(void *); // counts
    if (flags & kCFBasicHashHasHashCache) size += sizeof(uintptr_t *); // hashes
    CFBasicHashRef ht = (CFBasicHashRef)_CFRuntimeCreateInstance(allocator, CFBasicHashGetTypeID(), size, NULL);
    if (NULL == ht) return NULL;

    ht->bits.finalized = 0;
    ht->bits.hash_style = (flags >> 13) & 0x3;
    ht->bits.fast_grow = (flags & kCFBasicHashAggressiveGrowth) ? 1 : 0;
    ht->bits.counts_width = 0;
    ht->bits.strong_values = (flags & kCFBasicHashStrongValues) ? 1 : 0;
    ht->bits.strong_keys = (flags & kCFBasicHashStrongKeys) ? 1 : 0;
    ht->bits.weak_values = (flags & kCFBasicHashWeakValues) ? 1 : 0;
    ht->bits.weak_keys = (flags & kCFBasicHashWeakKeys) ? 1 : 0;
    ht->bits.int_values = (flags & kCFBasicHashIntegerValues) ? 1 : 0;
    ht->bits.int_keys = (flags & kCFBasicHashIntegerKeys) ? 1 : 0;
    ht->bits.indirect_keys = (flags & kCFBasicHashIndirectKeys) ? 1 : 0;
    ht->bits.num_buckets_idx = 0;
    ht->bits.used_buckets = 0;
    ht->bits.deleted = 0;
    ht->bits.mutations = 1;

    if (ht->bits.strong_values && ht->bits.weak_values) HALT;
    if (ht->bits.strong_values && ht->bits.int_values) HALT;
    if (ht->bits.strong_keys && ht->bits.weak_keys) HALT;
    if (ht->bits.strong_keys && ht->bits.int_keys) HALT;
    if (ht->bits.weak_values && ht->bits.int_values) HALT;
    if (ht->bits.weak_keys && ht->bits.int_keys) HALT;
    if (ht->bits.indirect_keys && ht->bits.strong_keys) HALT;
    if (ht->bits.indirect_keys && ht->bits.weak_keys) HALT;
    if (ht->bits.indirect_keys && ht->bits.int_keys) HALT;

    uint64_t offset = 1;
    ht->bits.keys_offset = (flags & kCFBasicHashHasKeys) ? offset++ : 0;
    ht->bits.counts_offset = (flags & kCFBasicHashHasCounts) ? offset++ : 0;
    ht->bits.hashes_offset = (flags & kCFBasicHashHasHashCache) ? offset++ : 0;

#if DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
    ht->bits.hashes_offset = 0;
    ht->bits.strong_values = 0;
    ht->bits.strong_keys = 0;
    ht->bits.weak_values = 0;
    ht->bits.weak_keys = 0;
#endif

    ht->bits.__kret = CFBasicHashGetPtrIndex((void *)cb->retainKey);
    ht->bits.__vret = CFBasicHashGetPtrIndex((void *)cb->retainValue);
    ht->bits.__krel = CFBasicHashGetPtrIndex((void *)cb->releaseKey);
    ht->bits.__vrel = CFBasicHashGetPtrIndex((void *)cb->releaseValue);
    ht->bits.__kdes = CFBasicHashGetPtrIndex((void *)cb->copyKeyDescription);
    ht->bits.__vdes = CFBasicHashGetPtrIndex((void *)cb->copyValueDescription);
    ht->bits.__kequ = CFBasicHashGetPtrIndex((void *)cb->equateKeys);
    ht->bits.__vequ = CFBasicHashGetPtrIndex((void *)cb->equateValues);
    ht->bits.__khas = CFBasicHashGetPtrIndex((void *)cb->hashKey);
    ht->bits.__kget = CFBasicHashGetPtrIndex((void *)cb->getIndirectKey);

    for (CFIndex idx = 0; idx < offset; idx++) {
        ht->pointers[idx] = NULL;
    }

#if ENABLE_MEMORY_COUNTERS
    int64_t size_now = OSAtomicAdd64Barrier((int64_t) CFBasicHashGetSize(ht, true), & __CFBasicHashTotalSize);
    while (__CFBasicHashPeakSize < size_now && !OSAtomicCompareAndSwap64Barrier(__CFBasicHashPeakSize, size_now, & __CFBasicHashPeakSize));
    int64_t count_now = OSAtomicAdd64Barrier(1, & __CFBasicHashTotalCount);
    while (__CFBasicHashPeakCount < count_now && !OSAtomicCompareAndSwap64Barrier(__CFBasicHashPeakCount, count_now, & __CFBasicHashPeakCount));
    OSAtomicAdd32Barrier(1, &__CFBasicHashSizes[ht->bits.num_buckets_idx]);
#endif

    return ht;
}
```
[CF-1153.18/CFBasicHash.c][1]

### 所有权修饰符
　　Objective-C 中为了处理对象，将类型定义为 id 类型或各种对象类型
　　id 类型用于隐藏类型的类名，相当于C语言中的 void *。

* **__strong：** 表示对对象的“强引用”。持有强引用的变量，在超出其作用域时强引用失效，所以自动地释放自己持有的对象，对象的所有者不存在，因此废弃该对象。该修饰符是 id 类型和对象类型默认的所有权修饰符。
* **__weak：** 表示对对象的“弱引用”。持有弱引用的变量，在超出其作用域时，对象即被释放。在持有某对象的弱引用时，若该对象被废弃，则此弱引用将自动失效且被赋值为 nil (空弱引用)。
* **__unsafe_unretained：** 附有该修饰符的变量不属于编译器的内存管理对象。既不持有对象的强引用也不持有对象的弱引用，只是表示对象，若该对象被废弃，其为悬垂指针。
* **__autoreleasing：** 

**属性声明的属性与所有权修饰符的对应关系**

| 属性声明的属性           | 所有权修饰符                    |
|-------------------|---------------------------------|
| assign            | __unsafe_unretained             |
| copy              | __strong (赋值的是被复制的对象) |
| retain            | __strong                        |
| strong           | __strong                          |
| weak             | __weak                          |
| unsafe_unretained | __unsafe_unretained             |

>内存泄漏就是应当废弃的对象在超出其生存周期后继续存在。相互引用(循环引用)容易发生内存泄漏。

>野指针是指向“垃圾”内存（不可用内存）的指针。不是NULL指针。

>悬垂指针是指指向曾经存在的对象，但该对象已经不再存在了。

>附有__strong 和__weak 修饰符的变量类似于C++中的智能指针 std::shared_ptr 和 std::weak_ptr。std::shared_ptr 可通过引用计数来持有C++ 类实例，std::weak_ptr 可避免循环引用。
 
### autorelease

　　类似于C语言中的自动变量(局部变量)的特性。若某自动变量超出其作用域，改自动变量将被自动废弃。

**autorelease 的具体用法：**

* 生成并持有 NSAutoreleasePool 对象
* 调用已分配对象的 autorelease 实例方法
* 废弃 NSAutoreleasePool 对象
```objective-c
/* ARC无效 */
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
// 等同于 objc_autoreleasePoolPush();

id obj = [[NSObject alloc] init];
[obj autorelease];
// 等同于 objc_autorelease(obj);

[pool drain];
// 等同于 objc_autoreleasePoolPop(pool);

/* ARC有效 */
@autoreleasepool { // 显式
    id __autoreleasing obj = [[NSObject alloc] init];
}

@autoreleasepool { // 非显式
    // 非自己生成并持有的对象
    id __strong obj = [NSMutableArray array];
}
```
　　ARC有效时，指定“@autoreleasepool 块”来代替“NSAutoreleasePool 类对象生成，持有以及废弃”。通过将对象赋值给附加了\_\_autoreleasing 修饰符的变量来代替调用 autorelease 方法。对象赋值给附有 \_\_autoreleasing 修饰符的变量等价于在ARC无效时调用对象的 autorelease 方法，即对象被注册到 autoreleasepool 。

　　非显式地使用 \_\_autoreleasing 也可以。这是由于编译器会检查**方法名**是否以 alloc/new/copy/mutableCopy 开始，如果不是则自动将返回的对象注册到 autoreleasepool 。

```objective-c
+ (id)array {
    id obj = [[NSMutableArray alloc] init];
    return obj;
}
```
　　以上为取得非自己生成并持有的对象时被调用方法的源代码示例。因为没有显式指定所有权修饰符所以 id obj 同附有 \_\_strong 修饰符的 id \_\_strong obj 是完全一样的。由于 return 使得对象变量超出其作用域，所以该强引用对应的自己持有的对象会被自动释放，但该对象作为函数的返回值，编译器会自动将其注册到 autoreleasepool 中。

　　id 的指针(id *obj)或对象的指针(NSObject **obj)在没有显式指定时会被附加上 \_\_autoreleasing 修饰符。使用附有 \_\_autoreleasing 修饰符的变量作为对象取得参数，都会注册到 autoreleasepool ， 并取得非自己生成并持有的对象。 如下：
```objective-c
- (BOOL)performOperationWithError:(NSError * __autoreleasing *)error {
    *error = [[NSError alloc] initWithDomain:@"Domain" code:0 userInfo:nil];
    return NO;
}

// 调用方法
NSError __strong *error = nil;
BOOL result = [obj performOperationWithError: &error];

编译器转化上述代码为下：

NSError __strong *error = nil;
NSError __autoreleasing *tmp = error;
BOOL result = [obj performOperationWithError: &tmp];
error = tmp;
```
>在显式地指定 \_\_autoreleasing 修饰符时，必须注意对象变量要为自动变量(包括局部变量，函数，以及方法参数)。

　　在访问附有 \_\_weak 修饰符的变量时，实际上必定要访问注册到 autoreleasepool 的对象。为什么？
```objective-c
id __strong obj0 = [[NSObject alloc] init];
id __weak obj1 = obj0;
NSLog(@"class = %@", [obj1 class]);

编译器转化上述代码为下：
id __weak obj1 = obj0;
id __autoreleasing tmp = obj1;
NSLog(@"class = %@", [tmp class]);
```
　　这是因为 __weak 修饰符只持有对象的弱引用，而在访问引用对象的过程中，该对象有可能被废弃。如果把要访问的对象注册到 autoreleasepool 中，那么在 @autoreleasepool 块结束之前都能确保该对象存在。

　　在Cocoa框架中，相当于主循环的 NSRunLoop 或者在其他程序可运行的地方，对 NSAutoreleasePool 对象进行生成，持有和废弃处理。

　　Cocoa框架中有很多类方法用于返回 autorelease 的对象。比如：
```objective-c
id array = [NSMutableArray arrayWithCapacity:1];
id array2 = [[[NSMutableArray alloc] initWithCapacity:1] autorelease];
```

>无论ARC是否有效，调试用的非公开函数 \_\_objc\_\_autoreleasePoolPrint() 都可使用。利用这一函数可调试注册到 autoreleasepool 上的对象。

**autorelease 的实现**

```c++
class AutoreleasePoolPage 
{
    id *add(id obj)
    {
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        return ret;
    }

    void releaseAll() 
    {
        releaseUntil(begin());
    }

    static inline id autorelease(id obj)
    {
        assert(obj);
        assert(!obj->isTaggedPointer());
        id *dest __unused = autoreleaseFast(obj);
        assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
        return obj;
    }

    static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }

    static inline void pop(void *token) 
    {
        AutoreleasePoolPage *page;
        id *stop;

        if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
            // Popping the top-level placeholder pool.
            if (hotPage()) {
                // Pool was used. Pop its contents normally.
                // Pool pages remain allocated for re-use as usual.
                pop(coldPage()->begin());
            } else {
                // Pool was never used. Clear the placeholder.
                setHotPage(nil);
            }
            return;
        }

        page = pageForPointer(token);
        stop = (id *)token;
        if (*stop != POOL_BOUNDARY) {
            if (stop == page->begin()  &&  !page->parent) {
                // Start of coldest page may correctly not be POOL_BOUNDARY:
                // 1. top-level pool is popped, leaving the cold page in place
                // 2. an object is autoreleased with no pool
            } else {
                // Error. For bincompat purposes this is not 
                // fatal in executables built with old SDKs.
                return badPop(token);
            }
        }

        if (PrintPoolHiwat) printHiwat();

        page->releaseUntil(stop);

        // memory: delete empty children
        if (DebugPoolAllocation  &&  page->empty()) {
            // special case: delete everything during page-per-pool debugging
            AutoreleasePoolPage *parent = page->parent;
            page->kill();
            setHotPage(parent);
        } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
            // special case: delete everything for pop(top) 
            // when debugging missing autorelease pools
            page->kill();
            setHotPage(nil);
        } 
        else if (page->child) {
            // hysteresis: keep one empty child if page is more than half full
            if (page->lessThanHalfFull()) {
                page->child->kill();
            }
            else if (page->child->child) {
                page->child->child->kill();
            }
        }
    }

    static void init()
    {
        int r __unused = pthread_key_init_np(AutoreleasePoolPage::key, 
                                             AutoreleasePoolPage::tls_dealloc);
        assert(r == 0);
    }
}
```
[objc4-706/runtime/NSObject.mm][2]

## ARC规则

* 不能使用 retain/release/retainCount/autorelease。在ARC下，内存管理是编译器的工作。
* 不能使用 NSAllocateObject/NSDeallocateObject
* 须遵守内存管理的方法命名规则。alloc/new/copy/mutableCopy，以这些名称开始的方法在返回对象时，必须返回给调用方所应当持有的对象。以 init 开始的方法必须是实例方法，并且必须要返回对象。返回的对象应为 id 类型或该方法声明类的对象类型，抑或是该类的超类或子类型。该返回对象不注册到 autoreleasepool 中。 基本上只是对 alloc 方法返回值的对象进行初始化处理并返回该对象。
* 不要显式调用 dealloc。 无论 ARC 是否有效，只要对象的所有者都不持有该对象，该对象就被废弃。对象被废弃时，不管 ARC 是否有效，都会调用对象的 dealloc 方法。
* 使用@autoreleasepool 块代替 NSAutoreleasePool 
* 不能使用区域(NSZone)
* 对象型变量不能作为C语言结构体(struct/union)的成员。C语言的规范上没有方法来管理结构体成员的生存周期。要把对象型变量加入到结构体成员中时，可强制转换为 void *，或者时附加 \_\_unsafe\_unretained 。 附有该修饰符的变量不属于编译器的内存管理对象。
* 显式转换“ id ”和“ void * ”。

**\_\_bridge 转换**  单纯的赋值转换。注意转换为 void * 的 \_\_bridge 转换其安全性与赋值给 \_\_unsafe\_unretained 修饰符相近，甚至会更低。如果不注意赋值对象的所有者，就会因悬垂指针而导致程序崩溃。
```objective-c
id obj = [[NSObject alloc] init];
void *p = (__bridge void *)obj;
id o = (__bridge id)p;
```

**\_\_bridge\_retained 转换** 可使要转换赋值的变量也持有所赋值的对象。
```objective-c
id obj = [[NSObject alloc] init];
void *p = (__bridge_retained void *)obj;
```

**\_\_bridge\_transfer 转换** 被转换的变量所持有的对象在该变量被赋值给转换目标变量后随之释放。
```objective-c
id obj = [[NSObject alloc] init];
void *p = (__bridge_retained void *)obj;

id obj1 = (__bridge_transfer id)p;
```

　　以下函数可用于 Objective-C 对象与 Core Foundation 对象之间的相互变换，即 Toll-Free-Bridge 转化。由于这种转换不需要使用额外的CPU资源，因此也被称为“免费桥”。
```objective-c
CFTypeRef CFBridgingRetain(id  _Nullable X) {
    return (__bridge_retained CFTypeRef)X;
}

id CFBridgingRelease(CFTypeRef  _Nullable X) {
        return (__bridge_transfer id)X;
}
```
　　使用方法：
```objective-c
NSMutableArray *obj = [[NSMutableArray alloc] initWithCapacity:1];
CFMutableArrayRef cfobject = CFBridgingRetain(obj);
CFShow(cfobject);
CFRelease(cfobject);

CFMutableArrayRef cfobject = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
id obj = CFBridgingRelease(cfobject);
```

## ARC实现

* **__strong：**表示对对象的“强引用”。引用计数为1。持有强引用的变量，在超出其作用域时强引用失效，所以自动地释放自己持有的对象，对象的所有者不存在，因此废弃该对象。该修饰符是 id 类型和对象类型默认的所有权修饰符。

```objective-c
id __strong obj = [[NSObject alloc] init];

// 编译器的模拟代码：
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_release(obj);
```

　　使用 alloc/new/copy/mutableCopy 以外的方法

```objective-c
+ (id)array {
    return [[NSMutableArray alloc] init];
}
// NSMutableArray 类的 array 类方法 通过编译器转换后的模拟代码：
+ (id)array {
    id obj = objc_msgSend(NSMutableArray, @selector(alloc));
    objc_msgSend(obj, @selector(init));
    return objc_autoreleaseReturnValue(obj);
}

id __strong obj = [NSMutableArray array];
// 编译器的模拟代码：
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleasedReturnValue(obj)
objc_release(obj);
```
　　objc_autoreleaseReturnValue() 函数会检查使用该函数的方法或者函数调用方的执行命令列表，如果方法或者函数的调用方调用了方法或者函数后紧接着调用 objc_retainAutoreleasedReturnValue() 函数，那么就不将返回的对象注册到 autoreleasepool 中，而是直接传递到方法或者函数的调用方。
而返回的对象则存储在 TLS 中， Thread Local Storage（TLS）线程局部存储，目的很简单，将一块内存作为某个线程专有的存储，以 key-value 的形式进行读写。在返回的对象身上调用 objc_autoreleaseReturnValue() 方法时， runtime 将这个返回的对象 obj 储存在 TLS 中，然后直接返回这个 obj （不调用autorelease）；同时，在外部接收这个返回的对象的方法 objc_retainAutoreleasedReturnValue() 里发现 TLS 中正好存了这个对象，那么直接返回这个 obj （不调用retain）。于是乎，调用方和被调方利用 TLS 做中转，很有默契的免去了对返回值的内存管理。这就是在ARC下，runtime 对 autorelease 返回值的优化策略。

>Thread-local storage（线程局部存储）指向 hot page ，即最新添加的 autoreleased 对象所在的那个 page 。???

* **__weak：** 表示对对象的“弱引用”。弱引用并不持有对象，不会改变赋值给附有 \_\_weak 修饰符的变量的引用计数。持有弱引用的变量，在超出其作用域时，对象即被释放。在持有某对象的弱引用时，若该对象被废弃，则此弱引用将自动失效且被赋值为 nil (空弱引用)。**使用** \_\_weak 修饰符的变量，即是使用注册到 autoreleasepool 中的对象，该对象的引用计数会加1。

```objective-c
id __strong obj = [[NSObject alloc] init];

id __weak obj1 = obj;
NSLog(@"%@", obj1);

// 编译器的模拟代码：
id obj1;
objc_initWeak(&obj1, obj);
id tmp = objc_loadWeakRetained(&obj1);
objc_autorelease(tmp);
NSLog(@"%@", tmp);
objc_destroyWeak(&obj1);

id objc_initWeak(id *location, id newObj) {
    return storeWeak(location, (objc_object*)newObj);
}

void objc_destroyWeak(id *location) {
    (void)storeWeak(location, nil);
}
```
　　storeWeak() 函数把第二参数的赋值对象的地址作为键值key，将第一参数的附有 \_\_weak 修饰符的变量的地址作为键值 value 注册到 \_\_weak 表中。如果第二参数(key)为 nil， 则把变量的地址(value)从 weak 表中删除。

　　weak 表与引用计数表相同，作为散列表被实现。如果使用 weak 表，将废弃对象的地址作为键值key进行检索，就能高速地获取对应的附有 \_\_weak 修饰符的变量的地址。另外，由于一个对象可同时赋值给多个附有 \_\_weak 修饰符的变量中，所以对于一个键值key，可注册多个变量的地址。
　　
　　objc_loadWeakRetained() 函数取出附有 \_\_weak 修饰符变量所引用的对象并 retain。objc_autorelease() 函数将对象注册到 autoreleasepool 中 。如果大量使用附有 \_\_weak修饰符的变量，注册到 autoreleasepool 中的对象也会大量的增加，因此在使用附有 \_\_weak修饰符的变量时，最好先暂时赋值给附有 __strong 修饰符的变量后使用。

>对象废弃执行的动作

1. objc_release
2. 当引用计数为0执行dealloc
3. _objc_rootDealloc
4. object_dispose
5. objc_destructInstance
6. objc_clear_deallocating

>对象被弃用时最后调用objc_clear_deallocating 函数执行的动作

1. 从 weak 表中获取废弃对象的地址为键值key的记录
2. 将包含在记录中的所有附有 __weak 修饰符变量的地址，赋值为 nil 
3. 从weak 表中删除该记录
4. 从引用计数表中删除废弃对象的地址为键值的记录

　　如果大量使用附有 \_\_weak 修饰符的变量，则会消耗相应的CPU资源，良策是只在需要避免循环引用时使用 \_\_weak 修饰符。

* **__unsafe_unretained：** 附有该修饰符的变量不属于编译器的内存管理对象。既不持有对象的强引用也不持有对象的弱引用，只是表示对象，若该对象被废弃，其为悬垂指针。

* **__autoreleasing：** 将对象赋值给有 __autoreleasing 修饰符的变量，等同于 ARC 无效时调用对象 autorelease 方法，会将对象注册到 autoreleasepool 中，对象的引用计数加1。

```objective-c
@autoreleasepool {
    id __autoreleasing obj = [[NSObject alloc] init];
}
/* 编译器的模拟代码 */
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_autorelease(obj);
objc_autoreleasePoolPop(pool);

@autoreleasepool {
    id __autoreleasing obj = [NSMutableArray array];
}
// 编译器的模拟代码：
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleasedReturnValue(obj);
objc_autorelease(obj);
objc_autoreleasePoolPop(pool);
```


[1]: https://opensource.apple.com/tarballs/CF/
[2]: https://opensource.apple.com/tarballs/objc4/
[3]: http://clang.org/
[2]: http://clang.org/
