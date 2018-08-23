# README

## 一. load 方法什么时候调用?

- `load`方法会在`runtime`加载`类`、`分类`时调用.
- 每个`类`、`分类`的`load`方法,在程序运行过程中`只调用一次`.

## 二.单个类的`load`方法调用及其分类的`load`方法调用

- 假设有这么一个类.`TYPerson`.

```objc
// TYPerson.h 文件
@interface TYPerson : NSObject
+ (void)test;
@end

// TYPerson.m 文件
@implementation TYPerson
+ (void)load {
    NSLog(@"%s",__func__);
}

+ (void)test {
    NSLog(@"%s",__func__);
}
@end
```

- 它有2个分类,分别为: 
    - TYPerson+category_First`

    ```objc
    @implementation TYPerson (category_First)
    + (void)load {
        NSLog(@"%s",__func__);
    }
    + (void)test {
        NSLog(@"%s",__func__);
    }
    @end
    ```
    
    - `TYPerson+category_Second`

    ```objc
    @implementation TYPerson (category_Second)
    + (void)load {
        NSLog(@"%s",__func__);
    }
    
    + (void)test {
        NSLog(@"%s",__func__);
    }
    @end
    ```

    
- 调用 `[ TYPerson test]`方法.那么打印如下:

```lldb
+[TYPerson load]
+[TYPerson(category_Second) load]
+[TYPerson(category_First) load]
+[TYPerson(category_First) test]
```

- 我们知道,类及其它的分类,如果有相同方法的实现,那么会优先调用分类的方法(后编译的分类的方法会优先调用).
- 那么为什么`load`方法不同?为什么不是同`test`方法类似,`只调用分类里面的 load 方法呢?`

### 1. load 方法的底层实现

通过`runtime`源码,窥探`load`方法的底层实现.

因为`load`方法是在`运行时加载类及其分类的时候调用`.所以从`load_images`这个方法看起.

![](https://lh3.googleusercontent.com/knvq-5Lrmhi3Q1yYXjY255tkQkmizKhsXxarbX4Tl5RXRr1P2TWnzZ2r0MArXZO6-QfCIabvKUkU-J6wTa1kjo530xH6ki_pGHmKo2XZcZAG_z4o_GJu-pYNyt72JpqiFG0KTdpb7pBBTQwhxpIosf2YWY3E1vcZ2OTWYzEKIionmb8ag2ln2prndE-MHcDIXnAhadfAIYfD9RZxCwgB42nG1DzLmrkVvx9rkdQL5EhNoJlKB0xJOU3qUZ1bTOlIyfCeGINCKYDhmro5YCJ8Rck4bubl2vy83cgyD1lNp-FS1rsZpbaQTr3CE26Qsbo8I47xn9H2smyusN5tM8aucvVH-odoYZYJVDMzdGg6UZR3dpno8k_KRqRsxqYtR-_CDLVgLZLuYLlr8nG3WNPVRWq3d78feWGqCX1fqd45lrQPQKEMpWpV0WB0Cd0DLpzC2oifiE0lPvrfeZJG7NIXiIIiD2wQXUAE2crGQR7ewI7Fkxlx7ERlaOrsu1b0rR8jKJP0laZ1EA5htKu_WwE6r8gBM3j_B4jiUpPomUKeELLnUYhK-GOmGIw5bcV0AGmlswPBasHlqaHN2NOsEIVk1qy7NKVZqYMmllw15w=w956-h444-no)

- 看`load_images`方法实现

```c++
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    // 检查有没有 +load 方法
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    // 发现 +load 方法
    {
        rwlock_writer_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    // 调用 +load 方法
    call_load_methods();
}
```

- 看`call_load_methods()`方法,看起如何调用`load`方法.
    - 看这个方法上分的注释,写的很清楚
    - 调用所有挂起的`类和分类的 +load`方法.
    - 先调用`类的+ load 方法,直到类的+ load 方法调用完毕,再调用分类的+ load方法`. 


```c++
/***********************************************************************
* call_load_methods
* Call all pending class and category +load methods.
* Class +load methods are called superclass-first. 
* Category +load methods are not called until after the parent class's +load.
* 
* This method must be RE-ENTRANT, because a +load could trigger 
* more image mapping. In addition, the superclass-first ordering 
* must be preserved in the face of re-entrant calls. Therefore, 
* only the OUTERMOST call of this function will do anything, and 
* that call will handle all loadable classes, even those generated 
* while it was running.
*
* The sequence below preserves +load ordering in the face of 
* image loading during a +load, and make sure that no 
* +load method is forgotten because it was added during 
* a +load call.
* Sequence:
* 1. Repeatedly call class +loads until there aren't any more
* 2. Call category +loads ONCE.
* 3. Run more +loads if:
*    (a) there are more classes to load, OR
*    (b) there are some potential category +loads that have 
*        still never been attempted.
* Category +loads are only run once to ensure "parent class first" 
* ordering, even if a category +load triggers a new loadable class 
* and a new loadable category attached to that class. 
*
* Locking: loadMethodLock must be held by the caller 
*   All other locks must not be held.
**********************************************************************/
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    // 先调用类的 load 方法
    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        // 类的 load 方法调用完毕,再调用分类的 load 方法
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
``` 

#### 1.1. 首先看`类的 + load`方法如何被调用的?

**先看结论: 通过函数地址,直接找到类的+ load 方法进行调用.(每一个类的,循环遍历,直到调用完成)**

- 看`call_class_loads`如何调用类的方法
    - 首先从分离当前可加载列表,得到可加载的类(需要调用 +load 方法的类列表)

    ```c++
    struct loadable_class *classes = loadable_classes;
    
    /*** -------------loadable_class--------------------***/
    // List of classes that need +load called (pending superclass +load)
    // This list always has superclasses first because of the way it is constructed
    static struct loadable_class *loadable_classes = nil;
    static int loadable_classes_used = 0;
    static int loadable_classes_allocated = 0;
    ``` 
    
    - `loadable_class` 的结构

    ```c++
    struct loadable_class {
       Class cls;  // may be nil
       IMP method; // 其实就是 +load 方法
    };
    ```
    
    - 对可加载的类循环遍历.取出 cls 和 method,然后通过 cls 调用它的 method(即 +load 方法).
    - `load_method_t` 其实就是 `typedef void(*load_method_t)(id, SEL);` 是一个`指向函数的指针,里面存放的是函数的地址`

    ```c++
    for (i = 0; i < used; i++) {
        // 取出 cls
        Class cls = classes[i].cls;
        // 取出 method(其实就是 + load 方法)
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        // 通过函数地址,直接调用 cls 这个类的方法(即 +load 方法)
        (*load_method)(cls, SEL_load);
    }
    ```

完整实现

```c++
/***********************************************************************
* call_class_loads
* Call all pending class +load methods.
* If new classes become loadable, +load is NOT called for them.
*
* Called only by call_load_methods().
**********************************************************************/
static void call_class_loads(void)
{
    int i;
    
    // Detach current loadable list.
    // 分离当前可加载列表
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // Call all +loads for the detached list.
    // 调用所有的 load 方法
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, SEL_load);
    }
    
    // Destroy the detached list.
    if (classes) free(classes);
}
```

#### 1.2. 其次看分类(category)的 `load`方法如何被调用?

**结论: 同类的 load 方法调用类似,都是通过函数地址直接去 load 方法**

看 `call_load_methods`这个方法中的

```c++
// 2. Call category +loads ONCE
more_categories = call_category_loads();
```

- `call_category_loads()` 方法的实现
    - 同调用类的方法实现类似
    - 首先分类可加载的类.得到分类的数组 

    ```c++
    struct loadable_category *cats = loadable_categories;
    ```
    
    - 每个 `loadable_category`中都包含`分类和方法(就是+ load 方法)`

    ```c++
    struct loadable_category {
        Category cat;  // may be nil
        IMP method;
    };
    ```
    
    - 然后循环变量,取出每个`分类和方法`
    - 最后通过函数地址直接调用方法

    ```c++
    (*load_method)(cls, SEL_load);
    ```

完整实现

```c++
/***********************************************************************
* call_category_loads
* Call some pending category +load methods.
* The parent class of the +load-implementing categories has all of 
*   its categories attached, in case some are lazily waiting for +initalize.
* Don't call +load unless the parent class is connected.
* If new categories become loadable, +load is NOT called, and they 
*   are added to the end of the loadable list, and we return TRUE.
* Return FALSE if no new categories became loadable.
*
* Called only by call_load_methods().
**********************************************************************/
static bool call_category_loads(void)
{
    int i, shift;
    bool new_categories_added = NO;
    
    // Detach current loadable list.
    struct loadable_category *cats = loadable_categories;
    int used = loadable_categories_used;
    int allocated = loadable_categories_allocated;
    loadable_categories = nil;
    loadable_categories_allocated = 0;
    loadable_categories_used = 0;

    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Category cat = cats[i].cat;
        load_method_t load_method = (load_method_t)cats[i].method;
        Class cls;
        if (!cat) continue;

        cls = _category_getClass(cat);
        if (cls  &&  cls->isLoadable()) {
            if (PrintLoading) {
                _objc_inform("LOAD: +[%s(%s) load]\n", 
                             cls->nameForLogging(), 
                             _category_getName(cat));
            }
            (*load_method)(cls, SEL_load);
            cats[i].cat = nil;
        }
    }

    // Compact detached list (order-preserving)
    shift = 0;
    for (i = 0; i < used; i++) {
        if (cats[i].cat) {
            cats[i-shift] = cats[i];
        } else {
            shift++;
        }
    }
    used -= shift;

    // Copy any new +load candidates from the new list to the detached list.
    new_categories_added = (loadable_categories_used > 0);
    for (i = 0; i < loadable_categories_used; i++) {
        if (used == allocated) {
            allocated = allocated*2 + 16;
            cats = (struct loadable_category *)
                realloc(cats, allocated *
                                  sizeof(struct loadable_category));
        }
        cats[used++] = loadable_categories[i];
    }

    // Destroy the new list.
    if (loadable_categories) free(loadable_categories);

    // Reattach the (now augmented) detached list. 
    // But if there's nothing left to load, destroy the list.
    if (used) {
        loadable_categories = cats;
        loadable_categories_used = used;
        loadable_categories_allocated = allocated;
    } else {
        if (cats) free(cats);
        loadable_categories = nil;
        loadable_categories_used = 0;
        loadable_categories_allocated = 0;
    }

    if (PrintLoading) {
        if (loadable_categories_used != 0) {
            _objc_inform("LOAD: %d categories still waiting for +load\n",
                         loadable_categories_used);
        }
    }

    return new_categories_added;
}
```

#### 1.3. 为什么分类的 `+ (void)test`方法会优先调用而`+ (void)load`方法会类和分类都调用解答.

- `[TYPerson test]`这个`+ (void)test`方法调用的本质就是`objc_msgSend([TYPerson class], @selector(test))`.
    - 首先会通过 `class 对象的 isa` 找到 `meta-class`. 找到其类方法的方法列表.
    - 然后按顺序遍历查找方法,那么就一定会先找到的分类,看起有没有`+ (void)test`方法,有就调用,没有继续往下找.

- 而 `+ (void)load`方法.本质是通过`拿到所有可加载的类及分类,然后通过函数地址,直接调用其方法(load 方法)`.
- 所以二者不同.


## 三.类之间有继承的时候, load 方法

现有一个类`TYStudent`继承自`TYPerson`.然后有两个`TYStudent`类的分类:`TYStudent+category_First`和`TYStudent+category_Second`.每个类及分类中都有`+(void)load`方法的实现.运行程序,打印如下:

```c
+[TYPerson load]
+[TYStudent load]
+[TYPerson(category_Second) load]
+[TYStudent(category_Second) load]
+[TYStudent(category_First) load]
+[TYPerson(category_First) load]
```

- 每个类及其分类的`load`方法都会被调用.

### 1.`load`方法的调用顺序?

通过方法`call_load_methods`方法可以得知:

- 会先反复调用`class`的`load`方法,直到所有 load 方法都调用完毕.
- 然后调用一次分类的 load 方法

```c++
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

### 2.`class`中 `load 方法`的调用顺序?

**结论**

- `有继承关系的类`: 先调用父类的 load 方法,再调用子类的 load 方法.
- `无继承关系的类`: 优先调用`先被编译`的类的 load 方法


**证明**

看上面`call_load_methods`方法.调用`class`中`load`方法那段.

```c++
do {
        // 1. Repeatedly call class +loads until there aren't any more
       while (loadable_classes_used > 0) {
           call_class_loads();
       }
    
       // 2. Call category +loads ONCE
       more_categories = call_category_loads();
    
       // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);
```

看`call_class_loads`方法的实现.

```c++
static void call_class_loads(void)
{
    int i;
    
    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // Call all +loads for the detached list.
    // 这里是循环遍历取出 classes 这个数组中的 class, 然后拿到它的方法(load)进行调用.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, SEL_load);
    }
    
    // Destroy the detached list.
    if (classes) free(classes);
}
```

通过`call_class_loads`这个方法实现,可以看出.它内部是通过`循环遍历取出 classes 这个数组中的 class. 然后按顺序拿到它的方法(即load方法)进行调用`.那么只要知道这个`classes`中类的顺序,就可以知道 class 中 load 方法的调用顺序了.

回到`load_images`方法.看`prepare_load_methods`这个方法.这是发现 load 方法并做准备操作的一步.

```c++
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        rwlock_writer_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
```

查看`prepare_load_methods`方法的实现:发现`schedule_class_load`方法.其含义是`prepare_load_methods Schedule +load for classes in this image, any un-+load-ed  superclasses in other images, and any categories in this image`,做一些准备,规划工作为类.

```c++
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertWriting();

    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
    // prepare_load_methods Schedule +load for classes in this image, any un-+load-ed superclasses in other images, and any categories in this image
    schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
```

继续查看`schedule_class_load`的实现.发现`add_class_to_loadable_list`方法.

```c++
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    // 确保先调用父类的 load 方法
    schedule_class_load(cls->superclass);

    // 将类加到这个数组中,有父类的话先添加父类进去,然后再加子类
    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
```

查看`add_class_to_loadable_list`方法的实现:

```c++
void add_class_to_loadable_list(Class cls)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = cls->getLoadMethod();
    if (!method) return;  // Don't bother if cls has no +load method
    
    if (PrintLoading) {
        _objc_inform("LOAD: class '%s' scheduled for +load", 
                     cls->nameForLogging());
    }
    
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}
```

**所以类中load 方法的调用顺序是先调用父类的.再调用子类的**

### 3.category 中 load 方法的调用顺序?

**结论**

- category 中的 load 方法不用管继承这种,只管`编译顺序`.`先被编译的先被调用`.

证明方法同 class 类似.


### 4.load 方法可不可以继承?

首先,我们一般不会去`主动`调用 `load方法`.但是如果问 load 方法可不可以继承,答案是:`子类确实可以调用父类的 load 方法.`

如有两个类: `TYStudent 继承自 TYPerson`.

- TYPerson 中实现了 load 方法
- TYStudent 中没有实现 load 方法
- TYStudent 主动调用 load 方法 `[TYStudent load]`
- 打印结果如下:

```c
// 这三个是 TYPerson 及其分类由 runtime 加载时主动调用的 load 方法
+[TYPerson load]
+[TYPerson(category_Second) load]
+[TYPerson(category_First) load]
------

// 这个是我们手动调用 load 方法的实现
+[TYPerson(category_First) load]
```

手动调用 load 方法.`[TYStudent load]`.其实质就是`objc_msgSend([TYStudent class], @selector(load))`.

- 那么会首先通过`TYStudent` 的`isa`指针找到`TYStudent的元类对象`,然后去这里找`+(void)load`方法.
- 因为在`TYStudent`的元类对象中找不到 load 方法.那么就会通过`superclass`指针去找其`父类的元类对象`,即`TYPerson`的元类对象,发现这里确实有load 方法,但是因为`TYPerson`的分类的方法会在方法数组的前面,所以优先调用分类的 load 方法.所以如上的打印结果.

