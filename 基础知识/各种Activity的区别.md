#### Activity、FragmentActivity和AppCompatActivity的区别

##### Activity
Activity是最基础的一个，是其它类的直接或间接父类。
Activity中只能使用系统自带的host Fragment（API Level 11中加入），对应getFragmentManager方法来控制Activity和Fragment之间的交互。

##### FragmentActivity
在v4包中引入FragmentActivity，FragmentActivity间接继承自Activity，并提供了对v4包中support Fragment的支持。
在FragmentActivity中必须使用getSupportFragmentManager方法来处理support Fragment的交互。也可以处理support Fragment的嵌套使用。

##### AppCompatActivity
AppCompatActivity继承自FragmentActivity，同时取代了ActionBarActivity。
AppCompatActivity支持ActionBar功能，同时更推荐使用ToolBar。AppCompatActivity为支持Material Design风格控件提供了便利。

##### 总结
优先级由1到3
1.如果方便使用toolbar，或者需要支持Material Design风格控件，就用AppCompatActivity
2.如果单纯需要使用fragment，就用FragmentActivity
3.其他情况，可以使用Activity

##### Material Design风格控件
AppBarLayout，ToolBar，DrawerLayout、NavigationView，CardView等等
