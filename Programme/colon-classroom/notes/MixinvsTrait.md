```md
Mixin和Trait

  本人认为这两个的涵义根据语言不同，而解释有所不同。但是它们的目的都是作为单继承不足的一种补充，或者是变相地实现多继承。实际上Java的接口也是变相的实现多继承，但是java的接口只是定义signature，没有实现体。在某种意义上Mixin和Trait这两者有点类似于抽象类，或者是有部分或全部实现体的Interface，但是在具体语言中，有表现出不一样的用法。总体上，笔者认为没有特别固定的或者是严格的区别。Mixin和Trait这两者都不能生成实例，否则就跟class没什么区别了。

Scala trait 
class Person ; //实验用的空类  
  
trait TTeacher extends Person {    
    def teach //虚方法，没有实现     
}    
trait TPianoPlayer extends Person {    
    def playPiano = {println("I’m playing piano. ")} //实方法，已实现    
}    
class PianoplayingTeacher extends Person with TTeacher with TPianoPlayer {    
    def teach = {println("I’m teaching students. ")} //定义虚方法的实现    
} 

PHP traits 
  // the template  
trait TSingleton {  
  private static $_instance = null;  
   
  public static function getInstance() {  
    if (null === self::$_instance)  
    {  
      self::$_instance = new self();  
    }   
    return self::$_instance;  
  }  
}  
class FrontController {  
  use TSingleton;  
}  
// can also be used in already extended classes  
class WebSite extends SomeClass {  
  use TSingleton;  
}  

Ruby mixin 
module Foo  
  def bar  
    puts "foo";  
  end  
end  

然后我们把这个模块混入到对象中去： 
class Demo  
  include Foo  
end

如上编码后，模块中的实例方法就会被混入到对象中： 
d=Demo.new  
d.bar

从上面看来，从使用上来说，可能看不出什么区别，但是如果说非要说区别的话，笔者通过搜索摘录和归纳了一些区别： 
1）Mixin可能更多的是指动态语言，它是在执行到某个点的时候，将代码插入到其中来达到代码复用的效果。Trait更多的是编译过程中，通过一些静态手段赋值代码到类中使得其拥有Trait中的一些功能以达到代码复用的目的； 
2）“Mixins may contain state, (traditional) traits don't.”这个区别比较弱，事实上Scala中Trait已经可以保存状态了（成员变量）； 
3）“Mixins use "implicit conflict resolution", traits use "explicit conflict resolution"”。这个区别可能是个明显的区别；但是如果某个语言它可以让Trait implicit resolve，那也没什么大不了。 
4）“Mixins depends on linearization, traits are flattened.”这个区别可能有。至少Scala里面貌似Trait是Flattened处理的，跟Java嵌套类差不多。 

类还是Trait？ 
当我们考虑是否一个“概念”应该成为一个Trait 或者一个类的时候，记住作为混入的Trait 对于“附属”行为来说最有意义。
如果你发现某一个Trait 经常作为其它类的父类来用，导致子类会有像父Trait 那样的行为，那么考虑把它定义为一个类吧，让这段逻辑关系更加清晰。
------------------------------------------------------------------------------------------------------------
Yii框架的组件行为管理机制和Mix-in

在Yii框架的官网，我们可以看到关于Behaviors & events的介绍: Behaviors are simply a way of adding methods to an object.

class SomeClass extends CBehavior{
    public function add($x, $y) { return $x + $y; }
}

class TestComponent extends CComponent {
}

$test_comp = new TestComponent();
$test_comp->attachbehavior('blah', new SomeClass);
$test_comp->add(2, 5);

在TestComponent类的对象创建的后，我们可以通过调用attachbehavior给对象添加新的方法。

通过其源码（在base/CComponent.php）可以知道它是通过在组件类内部以私有变量的方式存储这些添加的方法所在的对象， 
通过魔术方法__call，当调用一个未定义的方法时需要调用__call方法的特性，
遍历所有通过attachbehavior方法添加进来的对象， 并判断此对象是否禁用并且此对象是否存在需要调用的方法，如果存在则调用。

此种实现方式存在如下一些问题：
如果多个对象存在相同的方法，则程序调用时永远会调用第一次添加进去的方法
如果我们只是需要某个对象中的某个方法，但是在存储上需要将整个对象添加到列表中

也许你会觉得这些都是一些如果，都是一些假设，可能不会出现，这有些像众所周知的goto语句问题，
如果用得好，这是一个利器，如果用得不好，可能会给你带来痛苦。 Yii框架中的这种机制实现运行时的方法绑定，虽然类的属性和实例参数仍然归属于其它类和对象。

在PHP中，接口是可以多继承的，但是接口只是形态上的多继承，是一种对于类实现的约束，是一种规格。 如果要实现这种类的重用，Ruby受Lisp的影响引入了Mix-in，在PHP5.4引入了trait关键字。

在Ruby中Mix-in的关键字是module，而在即将推出的PHP5.4，其对应的关键字是trait； 如果要复用这个定义的类，在Ruby中使用include，而在PHP5.4中使用use。
Ruby：
module SomeClass
    def add(x, y)
        return x + y
    end
end

class TestComponet
    include SomeClass
end

test = TestComponet.new
puts test.add(10, 20)

PHP：
<?PHP
trait SomeClass {
    public function add($x, $y) {
        return $x + $y;
    }
}

class TestComponent {
    use SomeClass;
}

$obj = new TestComponent();
echo $obj->add(10, 20);


------------------------------------------------------------------------------------------------------------
现在排名靠前的面向对象的编程语言中，Java、C#等都是以单继承+接口来实现面向对象，
但是这在一定程序了稀释了继承的力量， 因为在业内推荐以组合的方式使用类。
这在一些常见的设计模式中有明显的体现，想想在GOF的23个设计模式中有多少个是使用了继承的呢？ 大多数是以接口+组合的方式实现。
其实作为一个类来说，它也比较难做，即要能代码复用，又得被实例化，偏向谁呢？ 
这个时候Mixin可能就有一些用武之地了。

Mixin最早起源于一个Lisp，Mixin鼓励代码重用，Mixin可以实现运行时的方法绑定，虽然类的属性和实例参数仍然是在编译时定义。 在面向对象编程语言，Mixin是一个提供了一些被用于继承或在子类中重用的功能的类，它类似于一种多继承， 但是实际上它是一种中小粒度的代码复用单元，而不直接用于实例化。 虽然这不是一种专业的方式进行功能复用，这在实现多继承的同时，在一定程序上避免了多继承的明显问题。

PHP和Java类似，也是单继承+接口。 我们知道，一个类可以实现任意数量的接口，这对一个类需要实现多个抽象的时候非常有用。 然而，对于要实现了多个接口的类，每个类都需要实现这些接口，而大多数情况下，这些接口都是可以共用的。 PHP并没有提供内置机制来定义和使用这些可重用代码，虽然我们可以对一地些接口使用一个抽象类来共用代码，但是如果这些类必须继承另一个抽象类呢？ 就算是可以通过抽象类的多次继承实现代码的共用，但是整个继承体系将会变得非常复杂，如果不能实现重用，那么可能我们只得CTRL + C 和 CTRL + V了。 大多数的情况下我们其实只是需要重用一些代码而已。

虽然PHP在之前没有提供完善的解决方案，但在新发布PHP5.4中，出现了一个关键字trait。 通过这个关键字我们可以定义抽象为一个Trait，如下示例：
trait HelloWorld {
    public function sayHello() {
        echo 'Hello World!';
    }
}

class TheWorldIsNotEnough {
    use HelloWorld;
    public function sayHello() {
        echo 'Hello Universe!';
    }
}

$o = new TheWorldIsNotEnough();
$o->sayHello(); // echos Hello Universe!

trait的实现

因为trait是一个语言结构，所以我们从zend_language_scanner.l文件中找到trait对应的关键字标识为：T_TRAIT 在zend_lang_parse.y文件中根据标识找到：

class_entry_type:
        T_CLASS         { $$.u.op.opline_num = CG(zend_lineno); $$.EA = 0; }
    |   T_ABSTRACT T_CLASS { $$.u.op.opline_num = CG(zend_lineno); $$.EA = ZEND_ACC_EXPLICIT_ABSTRACT_CLASS; }
    |   T_TRAIT { $$.u.op.opline_num = CG(zend_lineno); $$.EA = ZEND_ACC_TRAIT; }
    |   T_FINAL T_CLASS { $$.u.op.opline_num = CG(zend_lineno); $$.EA = ZEND_ACC_FINAL_CLASS; }
;

T_TRAIT作为一个关键字和class并级，它作为一个另一种类型的类存在。它将与接口、类处于同一字段标识。 其定义在zend_complie.h文件，如下：

#define ZEND_ACC_IMPLICIT_ABSTRACT_CLASS    0x10
#define ZEND_ACC_EXPLICIT_ABSTRACT_CLASS    0x20
#define ZEND_ACC_FINAL_CLASS                0x40
#define ZEND_ACC_INTERFACE                  0x80
#define ZEND_ACC_TRAIT                      0x120

以上的标识只是对应类的ce_flags结构，除此之外，在为的结构方面也有调整，如下：

struct _zend_class_entry {
    ...//   省略，木有修改
    zend_class_entry **interfaces;  //  接口列表
    zend_uint num_interfaces;   //  接口数

    zend_class_entry **traits;  //  traits列表
    zend_uint num_traits;   //      traits数
    zend_trait_alias **trait_aliases;   //  别名
    zend_trait_precedence **trait_precedences;

}

从上面的结构可以看出，PHP为traits增加了对应的字段存储。
从PHP的介绍中我们可知，trait可以动态绑定，则其执行应该是中间代码执行期间。 因此，如果使用了trait关键字，将会有对应的中间代码：ZEND_BIND_TRAITS 生成。 ZEND_BIND_TRAITS关键字最终调用zend_complie.c文件中的zend_do_bind_traits函数完成traits类的绑定，如下代码：

ZEND_API void zend_do_bind_traits(zend_class_entry *ce TSRMLS_DC) /* {{{ */
{

    if (ce->num_traits <= 0) {
        return;
    }

    /* complete initialization of trait strutures in ce */
    zend_traits_init_trait_structures(ce TSRMLS_CC);

    /* first care about all methods to be flattened into the class */
    zend_do_traits_method_binding(ce TSRMLS_CC);

    /* then flatten the properties into it, to, mostly to notfiy developer about problems */
    zend_do_traits_property_binding(ce TSRMLS_CC);

    /* verify that all abstract methods from traits have been implemented */
    zend_verify_abstract_class(ce TSRMLS_CC);

    /* now everything should be fine and an added ZEND_ACC_IMPLICIT_ABSTRACT_CLASS should be removed */
    if (ce->ce_flags & ZEND_ACC_IMPLICIT_ABSTRACT_CLASS) {
        ce->ce_flags -= ZEND_ACC_IMPLICIT_ABSTRACT_CLASS;
    }
}
/* }}} */

以上的绑定过程并不是和接口或类一样的的简单的合并操作，在合并操作之前需要处理引用和别名等情况， 此时类结构中的trait_aliases和trait_precedences就发挥作用了。 
trait定义的结构最终也是一个类。
```