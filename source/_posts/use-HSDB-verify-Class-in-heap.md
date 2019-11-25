---
title: 使用HSDB验证Class对象和类的静态对象保存在堆中
date: 2019-10-17 14:28:10
tags: ["java","JVM"]
categories : "技术"
---
### HSDB(Hotspot Debugger)
运行
```
图形界面    java -cp $JAVA_HOME/lib/sa-jdi.jar sun.jvm.hotspot.HSDB
命令行      java -cp $JAVA_HOME/lib/sa-jdi.jar sun.jvm.hotspot.CLHSDB
```
本文使用的时命令行CLHSDB。
由于HSDB会先attach进程，然后暂停进程，所以线上慎用。。。

### 验证过程
使用到的类
```
public class Main {
    private static String name = "yuck";
    private static int age = 24;

    public static void main(String[] args) throws IOException {
        new Main();
        System.in.read();
    }
}

```
VM参数
```
-XX:+UseSerialGC  // 默认的XX:+UseParallelGC， 我在scanoops 会报no such type的异常，不知道是不是bug...
-XX:-UseCompressedOops
```
找到进程pid ,attach
```
> jps
19852 Main

> java -cp $JAVA_HOME/lib/sa-jdi.jar sun.jvm.hotspot.HSDB
> attach 19852
```
查看堆中内存使用情况
```
hsdb> universe
Heap Parameters:
Gen 0:   eden [0x0000000012400000,0x0000000012961068,0x0000000013eb0000) space capacity = 27983872, 20.155523867461945 used
  from [0x0000000013eb0000,0x0000000013eb0000,0x0000000014200000) space capacity = 3473408, 0.0 used
  to   [0x0000000014200000,0x0000000014200000,0x0000000014550000) space capacity = 3473408, 0.0 usedInvocations: 0

Gen 1:   old  [0x0000000014550000,0x0000000014550000,0x0000000018800000) space capacity = 69926912, 0.0 usedInvocations: 0
```
可以看到堆中目前只使用了eden区域存放数据。
在该区域扫描我们建的类

```
hsdb> scanoops 0x0000000012400000 0x0000000012961068 com.example.demo.Main
0x000000001275f610 com/example/demo/Main
hsdb> whatis 0x000000001275f610
Address 0x000000001275f610: In thread-local allocation buffer for thread "main" (1)  [0x000000001273ced0,0x000000001275f820,0x00000000127c70c0,{0x00000000127c70d8})

 ```
 扫到我们要的类，使用`whatis`发现分配在TLAB中(我的环境时jdk8，默认TLAB优化时开启的)
 使用`inspect`进一步查看对象的信息
 ```
hsdb> inspect 0x000000001275f610
instance of Oop for com/example/demo/Main @ 0x000000001275f610 @ 0x000000001275f610 (size = 16)
_mark: 1
_metadata._klass: InstanceKlass for com/example/demo/Main
```
 Main只有静态变量，所以对象实例大小为16byte,在64位关闭指针压缩的情况下 MARK WORLD(8byte) 加上类型指针(8byte,开启压缩是4byte) 正好是16byte。
**InstanceKlass就是指的是Class对象，所以这里就可以证明Class对象是存在堆中，而不是方法区**
使用`mem`找到具体地址 查看具体信息，`mem addresss length`
```
hsdb> mem 0x000000001275f610 2
0x000000001275f610: 0x0000000000000001
0x000000001275f618: 0x0000000018c03028

hsdb> whatis 0x0000000018c03028
pointer to InstanceKlass
hsdb> inspect 0x0000000018c03028
Type is InstanceKlass (size of 440)
juint Klass::_super_check_offset: 48
Klass* Klass::_secondary_super_cache: Klass @ null
Array<Klass*>* Klass::_secondary_supers: Array<Klass*> @ 0x0000000018800f88
Klass* Klass::_primary_supers[0]: Klass @ 0x0000000018801c00
oop Klass::_java_mirror: Oop for java/lang/Class @ 0x000000001275d298 Oop for java/lang/Class @ 0x000000001275d298
jint Klass::_modifier_flags: 1
Klass* Klass::_super: Klass @ 0x0000000018801c00
Klass* Klass::_subklass: Klass @ null
jint Klass::_layout_helper: 16
Symbol* Klass::_name: Symbol @ 0x0000000019af8fb0
AccessFlags Klass::_access_flags: 538968097
markOop Klass::_prototype_header: 5
Klass* Klass::_next_sibling: Klass @ 0x0000000018b5c5f8
u8 Klass::_trace_id: 42074112
Klass* InstanceKlass::_array_klasses: Klass @ null
Array<Method*>* InstanceKlass::_methods: Array<Method*> @ 0x0000000018c02d60
Array<Method*>* InstanceKlass::_default_methods: Array<Method*> @ null
Array<Klass*>* InstanceKlass::_local_interfaces: Array<Klass*> @ 0x0000000018800f88
Array<Klass*>* InstanceKlass::_transitive_interfaces: Array<Klass*> @ 0x0000000018800f88
Array<u2>* InstanceKlass::_fields: Array<u2> @ 0x0000000018c02d40
u2 InstanceKlass::_java_fields_count: 2
ConstantPool* InstanceKlass::_constants: ConstantPool @ 0x0000000018c02b70
ClassLoaderData* InstanceKlass::_class_loader_data: ClassLoaderData @ 0x000000000300c090
u2 InstanceKlass::_source_file_name_index: 30
char* InstanceKlass::_source_debug_extension: char @ null
Array<jushort>* InstanceKlass::_inner_classes: Array<jushort> @ 0x0000000018800f58
int InstanceKlass::_nonstatic_field_size: 0
int InstanceKlass::_static_field_size: 2
u2 InstanceKlass::_static_oop_field_count: 1
int InstanceKlass::_nonstatic_oop_map_size: 0
bool InstanceKlass::_is_marked_dependent: 0
u2 InstanceKlass::_minor_version: 0
u2 InstanceKlass::_major_version: 52
u1 InstanceKlass::_init_state: 4
Thread* InstanceKlass::_init_thread: Thread @ 0x0000000002f14000
int InstanceKlass::_vtable_len: 5
int InstanceKlass::_itable_len: 2
u1 InstanceKlass::_reference_type: 0
OopMapCache* InstanceKlass::_oop_map_cache: OopMapCache @ null
JNIid* InstanceKlass::_jni_ids: JNIid @ null
nmethod* InstanceKlass::_osr_nmethods_head: nmethod @ null
BreakpointInfo* InstanceKlass::_breakpoints: BreakpointInfo @ null
u2 InstanceKlass::_generic_signature_index: 0
jmethodID* InstanceKlass::_methods_jmethod_ids: jmethodID @ 0x0000000019af8ad0
u2 InstanceKlass::_idnum_allocated_count: 3
Annotations* InstanceKlass::_annotations: Annotations @ null
nmethodBucket* InstanceKlass::_dependencies: nmethodBucket @ null
Array<int>* InstanceKlass::_method_ordering: Array<int> @ 0x0000000018800f40
Array<int>* InstanceKlass::_default_vtable_indices: Array<int> @ null

 ```
 可以看到Class信息中
 ```
 oop Klass::_java_mirror: Oop for java/lang/Class @ 0x000000001275d298 Oop for java/lang/Class @ 0x000000001275d298
 ```
  _java_mirror表示该Klass的Java层镜像类（在Java7中由镜像类持有类型的静态成员）

 ```
 hsdb> whatis 0x000000001275d298
Address 0x000000001275d298: In thread-local allocation buffer for thread "main" (1)  [0x000000001273ced0,0x000000001275f820,0x00000000127c70c0,{0x00000000127c70d8})

hsdb> inspect 0x000000001275d298
instance of Oop for java/lang/Class @ 0x000000001275d298 @ 0x000000001275d298 (size = 176)
name: "yuck" @ 0x000000001275f5b8 Oop for java/lang/String @ 0x000000001275f5b8
age: 24
```
可以看到 该镜像中持有我们类中，创建的两个静态变量。


### 参考资料
https://book.douban.com/subject/25847620/annotation
https://www.iteye.com/blog/rednaxelafx-730461#comments
https://www.tuicool.com/articles/ryQv2iB
