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
 



