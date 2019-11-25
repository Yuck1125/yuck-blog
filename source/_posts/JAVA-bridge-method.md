---
title: 泛型桥接方法验证
date: 2019-11-25 11:30:08
tags: "JAVA"
categories: "技术"
---
### 简单验证

```
public interface SuperClass<T> {
    public T Apply(T t) ;
}


public class SonClass implements SuperClass<String> {

    public String Apply(String str) {
        return str;
    }
}
```

运行`javap -c -v SuperClass.class SonClass.class`查看编译后的结果 可以看到SonClass额外生成了一个`Object Apply (Object obj)`的桥接方法
```
.
.
.
{
  public abstract T Apply(T);
    descriptor: (Ljava/lang/Object;)Ljava/lang/Object;
    flags: ACC_PUBLIC, ACC_ABSTRACT
    MethodParameters:
      Name                           Flags
      t
    Signature: #8                           // (TT;)TT;
}
Signature: #9                           // <T:Ljava/lang/Object;>Ljava/lang/Object;
SourceFile: "SuperClass.java"


.
.
.
{
  public com.example.bridge.demo.SonClass();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/example/bridge/demo/SonClass;

  public java.lang.String Apply(java.lang.String);
    descriptor: (Ljava/lang/String;)Ljava/lang/String;
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=2, args_size=2
         0: aload_1
         1: areturn
      LineNumberTable:
        line 9: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       2     0  this   Lcom/example/bridge/demo/SonClass;
            0       2     1   str   Ljava/lang/String;
    MethodParameters:
      Name                           Flags
      str

  public java.lang.Object Apply(java.lang.Object);
    descriptor: (Ljava/lang/Object;)Ljava/lang/Object;
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: checkcast     #2                  // class java/lang/String
         5: invokevirtual #3                  // Method Apply:(Ljava/lang/String;)Ljava/lang/String;
         8: areturn
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/example/bridge/demo/SonClass;
    MethodParameters:
      Name                           Flags
      str                            synthetic
}
Signature: #21                          // Ljava/lang/Object;Lcom/example/bridge/demo/SuperClass<Ljava/lang/String;>;
SourceFile: "SonClass.java"

```

### 分析
我们都知道JAVA泛型并不是真正的泛型，在编译完成后泛型会被擦除，变成Object。如果没有桥接方法，编译完成后其实子类并没有实现父类方法。所以为了语义，编译器会自动生成桥接方法，来保证兼容性。
