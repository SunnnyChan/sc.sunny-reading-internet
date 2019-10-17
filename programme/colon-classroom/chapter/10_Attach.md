# 附

## 表1 
* Java/C#的抽象类与接口在语法上的区别
    
    |         | 抽象类|  接口  |
    | --------|:-----| :----|
    |提供实现类代码|能|否|
    |多重继承|否|能|
    |拥有非public成员|能|否|
    |拥有域成员|能|否（Java中的static final域成员除外）|
    |拥有static成员|能|否（Java中的static final域成员除外）|
    |拥有非abstract方法成员|能|否|
    |方法成员的默认修饰符|否|public abstract（Java：可选；C#：不能含有任何修饰符）|
    |域成员的默认修饰符|否|Java：public static final；C#：不允许域成员|

## 表2 
* Java/C#的抽象类与接口在语义上的区别

    |   |关系|共性|特征|联系|重用|实现|重点|演变|
    | --------|:-----|:----|:----|:----|:----|:----|:----|:----|
    |接口|can-do|相同功能|边缘特征|横向联系|规范重用|多重实现|可置换性|新增类型|
    |抽象类|is-a|相同种类|核心特征|纵向联系|代码重用|多级实现|可扩展性|新增成员|
