+++
title = "Java17新Native API初体验"
date = 2022-01-15T22:16:02+08:00
draft = false
archives = 2022
categories = "java"
tags = ["native","java"]
+++

最近在使用Java编写程序，需要和c++进行交互。正好碰到了Java17新添加的Native API（类似JNI）。来自[JEP 412](https://openjdk.java.net/jeps/412)。

<!--more-->

于是咱本着舍我其谁，身先士卒，敢为人先，一个顶俩的精神，使用了这个还在孵化的API。

### 加载函数

废话不多说，上Code:
```java
// 需要先加载库
System.load("xxx.dll");

CLinker linker = CLinker.getInstance();

MethodHandle init = linker.downcallHandle(
        SymbolLookup.loaderLookup().lookup(/*你的函数名称*/).orElseThrow(),
        MethodType.methodType(/*你的函数定义*/),
        FunctionDescriptor.of(/*你的函数定义*/)
);

```
这样，就可以简单加载一个函数了。我们只需要对返回的函数Hanlde执行调用操作就行了哦。记得填上参数，参数和返回值类型同`MethodType`的那个参数。ヽ(✿ﾟ▽ﾟ)ノ

当然，在`MethodType.methodType`和`FunctionDescriptor.of`需要填写你的函数的定义。两个函数的调用要对应。例如:
```java
// 这段代码调用的c函数接口是:
// void* ();
// 即一个返回任意类型指针的函数
MethodType.methodType(MemoryAddress.class),
                FunctionDescriptor.of(CLinker.C_POINTER)

// 注意，MemoryAddress.class和CLinker.C_POINTER是对应关系（即指针类型）
// 如果修改，将会在运行时报错
```

下面有一个对应关系表，注意，如果返回void（即没有返回值），需要使用`MethodType.methodType(void.class,...)`和`FunctionDescriptor.ofVoid(...)`来替代。

下面是MethodType和FunctionDescriptor的对应关系表:

| Method Type | Function Descriptor | C Type |
| :----: | :-----: | :-----: |
| MemoryAddress.class | CLinker.C_POINTER | 任何类型的指针 |
| long.class | CLinker.C_LONG_LONG | unsigned long long |
| int.class | CLinker.C_INT | int |
| short.class | CLinker.C_SHORT | short |
| 以此类推，还有double,char等基本类型 | none | none |


### 解析数据结构
为了解析数据结构，我们需要一个Native函数返回一个MemoryAddress：指向数据结构。

一个例子:
```c++
struct Bitmap {
	uint32_t x;
	uint32_t y;
	uint32_t bitmap_left;
	uint32_t bitmap_top;
    // 指向一个矩形位图，大小由x和y决定。每位占用RGBA共4个字节。
	uint8_t* data;
};
```
parsing in java...
```java
// 数据结构定义
public class Bitmap {
    public int x;
    public int y;
    public int bitmapLeft;
    public int bitmapTop;
    public byte[] data;
}

// 解析
Bitmap parse(MemoryAddress ptr){
    // 将指针转化为可以访问的内存段
    MemorySegment segment
        = data.asSegment(/*经过手工计算，结构体的大小位16字节*/16,/*使用globalScope需要我们自己释放内存，如果想要JVM来释放，可以使用newImplicitScope，具体见Java文档*/ResourceScope.globalScope());

    // 根据ABI结构取出结构体的变量
    // MemoryAccess是一个辅助类，可以用来访问内存段
    var x = MemoryAccess.getIntAtIndex(segment,0);
    var y = MemoryAccess.getIntAtIndex(segment,1);
    var left = MemoryAccess.getIntAtIndex(segment,2);
    var top = MemoryAccess.getIntAtIndex(segment,3);
    var data = MemoryAccess.getAddressAtOffset(segment,16);

    // 复制数据
    Bitmap bitmap = new Bitmap();
    bitmap.x = x;
    bitmap.y = y;
    bitmap.bitmapLeft = left;
    bitmap.bitmapTop = top;
    bitmap.data = new byte[x * y * 4];

    // 复制Bitmap.data里的字节数组
    segment = data.asSegment(/*为什么要乘以4，因为c++部分的代码如此。此处填写内存段大小。*/(long) x * y * 4,ResourceScope.globalScope());

    // copy
    segment.asByteBuffer().get(bitmap.data);

    return bitmap;
}
```

### 后续
Java将继续孵化此API。*Java Forward Again!*
