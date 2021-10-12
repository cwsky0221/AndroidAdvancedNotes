Lambda

## 一、什么是Lambda
> Lambda 表达式，也可称为闭包，它是推动 Java 8 发布的最重要新特性。

Lambda 允许把函数作为一个方法的参数（函数作为参数传递进方法中）。使用 Lambda 表达式可以使代码变的更加简洁紧凑。
上面是java对于lambda的释义，那么什么是lambda的本质呢。java的函数调用邮件四个指令（invokevirtual、invokespecial、invokestatic、invokeinterface)，而在java8之后专门给lambda新增了一个指令，就是invokedynamic。
我们可以简单的把lambda理解问一个动态的链接，他将一个lambda表达式指向的其实是一个静态的方法调用，而这个方法调用会返回他所需要的表述类型等等信息。

看看代码：
```
public class OtherTestViewHolder extends ViewHolder {
    private int i = 100;

    public OtherTestViewHolder(@NonNull View itemView) {
        super(itemView);
        itemView.setOnClickListener((v) -> {
            Log.i("", "");
            ToastHelper.toast(v, v, "1234");
        });
    }
}

```

```
//删除无效的不想阅读的代码
// access flags 0x1
 public <init>(Landroid/view/View;)V
  L2
   LINENUMBER 19 L2
   ALOAD 1

   // 通过INVOKEDYNAMIC 将当前的的setOnClickListener 链接到lambda$new$0静态方法上去。
   INVOKEDYNAMIC onClick()Landroid/view/View$OnClickListener; [
     // handle kind 0x6 : INVOKESTATIC
     java/lang/invoke/LambdaMetafactory.metafactory(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
     // arguments:
     (Landroid/view/View;)V, 
     // handle kind 0x6 : INVOKESTATIC
     com/wallstreetcn/sample/adapter/OtherTestViewHolder.lambda$new$0(Landroid/view/View;)V, 
     (Landroid/view/View;)V
   ]
   INVOKEVIRTUAL android/view/View.setOnClickListener (Landroid/view/View$OnClickListener;)V
  L3
   LINENUMBER 22 L3
   RETURN
  L4
   LOCALVARIABLE this Lcom/wallstreetcn/sample/adapter/OtherTestViewHolder; L0 L4 0
   LOCALVARIABLE itemView Landroid/view/View; L0 L4 1
   MAXSTACK = 2
   MAXLOCALS = 2


   
// access flags 0x100A
private static synthetic lambda$new$0(Landroid/view/View;)V
 L0
  LINENUMBER 20 L0
  LDC ""
  LDC ""
  INVOKESTATIC android/util/Log.i (Ljava/lang/String;Ljava/lang/String;)I
  POP
 L1
  LINENUMBER 21 L1
  ALOAD 0
  ALOAD 0
  LDC "1234"
  INVOKESTATIC com/wallstreetcn/sample/ToastHelper.toast (Ljava/lang/Object;Landroid/view/View;Ljava/lang/Object;)V
  RETURN
 L2
  LOCALVARIABLE v Landroid/view/View; L0 L2 0
  MAXSTACK = 2
  MAXLOCALS = 1

```
上面是一个非常简单的java中的Lambad，以及其对应的字节码翻译。我们可以看到，其中Lambda的部分被翻译出来的就是INVOKEDYNAMIC，然后将这部分指向了一个静态方法而已，而静态方法中就是我们原始的java代码的那部分。

## 二、那么安卓中的lambda最后真的是java中的lambda吗?
虽然安卓在后续的版本上支持了java8的语法，但是由于线上分布了大量低版本的设备，所以安卓在实际生成产物的时候，并不是一个java8的INVOKEDYNAMIC语法，而是被Desugar脱糖成了一个匿名内部类了。
这样就能同时兼容到线上的所以旧版的安卓os设备，因为并没有新的字节码指令被引入，所以就不需要考虑兼容性问题了。

> 所以相对来说安卓的Lambda比java8的Lambda更像是一个语法糖，因为是由Desugar脱糖器处理成匿名内部类。

*总结：java会用invokeDynmic来处理lambda，而在Android上仅仅是个存语法糖了，transformClassesWithDesugarForDebug会将lambda翻译成匿名内部类*

感谢：[https://juejin.cn/post/6918733287015841800/](https://juejin.cn/post/6918733287015841800/)

