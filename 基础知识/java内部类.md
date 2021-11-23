#### Java 内部类

#### (一) 概述
把类定义在另一个类的内部，该类就被称为内部类。

#### (二) 访问规则
A:可以直接访问外部类的成员，包括私有

B:外部类要想访问内部类成员，必须创建对象

#### (三) 内部类的分类
A：成员内部类

（1）格式：
```
class Outer {
      private int age = 20;
      //成员位置
      class Inner {
          public void show() {
              System.out.println(age);
          }
      }
  }

```
(2) 创建内部类对象：
```
  //成员内部类不是静态的：
  外部类名.内部类名 对象名 = new 外部类名.new 内部类名();
  ​
  //成员内部类是静态的：
  外部类名.内部类名 对象名 = new 外部类名.内部类名(); 
```

B：局部内部类
局部内部类——就是定义在一个方法或者一个作用域里面的类
特点：主要是作用域发生了变化，只能在自身所在方法和属性中被使用

(1)格式：
```
class Outer {
      public void method(){
          class Inner {
          }
      }
  }
```

C：静态内部类
我们所知道static是不能用来修饰类的,但是成员内部类可以看做外部类中的一个成员,所以可以用static修饰,这种用static修饰的内部类我们称作静态内部类,也称作嵌套内部类.
特点：不能使用外部类的非static成员变量和成员方法
(1)格式：
```
class Outter {
      int age = 10;
      static age2 = 20;
      public Outter() {        
      }
       
      static class Inner {
          public method() {
              System.out.println(age);//错误
              System.out.println(age2);//正确
          }
      }
  }
  ​
  public class Test {
      public static void main(String[] args)  {
          Outter.Inner inner = new Outter.Inner();
          inner.method();
      }
  }

```

​D：匿名内部类
(1)格式：
```
new 类名或者接口名() {
      重写方法();
  }
```
我们在开发的时候，会看到抽象类，或者接口作为参数。

而这个时候，实际需要的是一个子类对象。

如果该方法仅仅调用一次，我们就可以使用匿名内部类的格式简化。



#### 使用内部类的原因
(1)封装性
作为一个类的编写者，我们很显然需要对这个类的使用访问者的访问权限做出一定的限制，我们需要将一些我们不愿意让别人看到的操作隐藏起来，如果我们的内部类不想轻易被任何人访问，可以选择使用private修饰内部类，这样我们就无法通过创建对象的方法来访问，想要访问只需要在外部类中定义一个public修饰的方法，间接调用

(2)实现多继承 

```
作者：二境志
链接：https://www.zhihu.com/question/26954130/answer/708467570
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

public class Demo1 {
      public String name() {
          return "BWH_Steven";
      }
  }
  
  public class Demo2 {
      public String email() {
          return "xxx.@163.com";
      }
  }
  
  public class MyDemo {
  ​
      private class test1 extends Demo1 {
          public String name() {
              return super.name();
          }
      }
  ​
      private class test2 extends Demo2  {
          public String email() {
              return super.email();
          }
      }
  ​
      public String name() {
          return new test1().name();
      }
  ​
      public String email() {
          return new test2().email();
      }
  ​
      public static void main(String args[]) {
          MyDemo md = new MyDemo();
          System.out.println("我的姓名:" + md.name());
          System.out.println("我的邮箱:" + md.email());
      }
  }
```
我们编写了两个待继承的类Demo1和Demo2，在MyDemo类中书写了两个内部类，test1和test2 两者分别继承了Demo1和Demo2类，这样MyDemo中就间接的实现了多继承

(3)用匿名内部类实现回调功能
我们用通俗讲解就是说在Java中，通常就是编写一个接口，然后你来实现这个接口，然后把这个接口的一个对象作以参数的形式传到另一个程序方法中， 然后通过接口调用你的方法，匿名内部类就可以很好的展现了这一种回调功能
