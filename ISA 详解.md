[TOC]
### 一. 认识ISA指针
**isa**：是一个Class 类型的指针. 每个实例对象有个isa的指针,他指向对象的类，而Class里也有个isa的指针, 指向meteClass(元类)。

OC中的所有的对象可分类三种类型：
1. **实例对象（instance）**
2. **类对象（class）**
3. **元类对象（mate-class）**


#### 1. isa指针：
- 实例对象的isa指向类对象；
- 类对象的isa指向元类对象；
- 元类对象的isa指向基类的元类对象；

#### 2. superclass指针：
- 类对象的superclass指向其父类类对象；  
如果没有父类，superclass指针为nil；
- 元类对象的superclass指向其父类类对象；  
基类的元类对象的superclass指向基类的类对象；

#### 3. 方法查找：
// 这里暂时忽略cache查找
- 对象方法的查找轨迹；  
通过实例对象的isa指针找到对应的类对象，在类对象的method_list中查找，方法不存在，就通过superclass在父类中查找；
- 类方法的查找轨迹；  
通过类对象的isa指针找到对应的元类对象，在元类对象的method_list中查找，方法不存在，就通过superclass在父类中查找；

### 二. ISA本质
#### 1. ISA历史
在arm64架构之前，isa就是一个普通的指针，存储着Class、Meta-Class对象的内存地址；  
从arm64架构开始，对isa进行了优化，变成了一个共用体（union）结构，还使用位域来存储更多的信息；

```Objective-C
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
      uintptr_t nonpointer        : 1;                                         
      uintptr_t has_assoc         : 1;                                         
      uintptr_t has_cxx_dtor      : 1;                                         
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ 
      uintptr_t magic             : 6;                                         
      uintptr_t weakly_referenced : 1;                                         
      uintptr_t deallocating      : 1;                                         
      uintptr_t has_sidetable_rc  : 1;                                         
      uintptr_t extra_rc          : 8
    };
#endif
};
```


#### 2. ISA 位域解释
- nonpointer  
0，代表普通的指针，存储着Class、Meta-Class对象的内存地址；
1，代表优化过，使用位域存储更多的信息；

- has_assoc    
是否有设置过关联对象，如果没有，释放时会更快；

- has_cxx_dtor   
是否有C++的析构函数（.cxx_destruct），如果没有，释放时会更快；

- shiftcls    
存储着Class、Meta-Class对象的内存地址信息；

- magic  
用于在调试时分辨对象是否未完成初始化；

- weakly_referenced  
是否有被弱引用指向过，如果没有，释放时会更快；

- deallocating  
对象是否正在释放；

- extra_rc  
里面存储的值是引用计数器减1；

- has_sidetable_rc  
引用计数器是否过大无法存储在isa中；
如果为1，那么引用计数会存储在一个叫SideTable的类的属性中；