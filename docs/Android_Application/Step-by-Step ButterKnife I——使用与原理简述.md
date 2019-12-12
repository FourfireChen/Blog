# Step-by-Step ButterKnife I——使用与原理简述

ButterKnife是很好用的基于注解的注入库，可以使用注解生成很多代码，帮助开发者节省很多诸如`findViewById`等等代码。

## 使用

以Activity中一个Button点击为例。
```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    private Button button;
    @Overrideprotected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        button = findViewById(R.id.mButton);
        button.setOnClickListener(this);
    }
    @Overridepublic void onClick(View v) {
        Toast.makeText(this, "click", Toast.LENGTH_SHORT).show();
    }
}
```
这是我们常见的写法。一旦控件多了，就需要写很多findViewById，很多click方法（或其中做很多判断）。使用ButterKnife后：

```java
public class MainActivity extends AppCompatActivity {
    @BindView(R.id.mButton)Button mButton;
    @Overrideprotected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
    }
    @OnClick(R.id.mButton)public void onViewClicked() {
        Toast.makeText(this, "click", Toast.LENGTH_SHORT).show();
    }
}
```
在Button变量上打注解，在click方法上打注解。以此可以实现和上一个相同的效果。

当然ButterKnife不止可以绑定View和onClick方法，还可以绑定各种资源、监听器等等。

## 实现原理

凡事初接触总会有猜测。笔者最初自己想象的ButterKnife的原理，是简单的反射，也是最简单的实现方式。
调用ButterKnife.bind的时候，对传进来的Activity的所有变量遍历一遍，如果打了注解，就为其赋值。

这个方法可以实现，但仔细思考可知，效率低下。即便是如今的Java，对反射的处理已经十分成熟，但可以想象如果Activity的变量有成千上万个，那也是一个非常大的开销。再举一个极端的例子，如果有许多个变量，但需要处理注解的只有一个，此时用这种方法也无法避免要对所有变量遍历一次，效率非常差。

那么ButterKnife本身是如何实现的呢？

ButterKnife是基于APT实现的。APT，全称Annotation Processing Tool，是一种处理注解的工具，基于APT，可以实现编译时注解，即在编译器处理源文件中的注解，并且APT提供生成额外源文件的功能，也就是说，在编译器，可以根据源代码上的注解，生成额外的源文件来处理。

ButterKnife正是采用此方法。

在编译期间，通过APT，获得所有打过注解的变量、方法，依据这些注解中提供的信息，对目标类（如Activity）的某些变量（如注解了BindView的View）生成辅助类，写好对目标的操作代码，这是以.java，即源代码的形式存在的。

在调用ButterKnife.bind的时候，调用了生成的辅助类，从而将运行期大量检查注解的时间消耗，提前到了编译期，大大提高了效率。

在gradle插件2.2之前，apt框架一般使用第三方的android-apt库，而在gradle插件2.2之后，google官方内置了框架annotationProcessor，这也是我们在下一篇文章使用的框架。使用这个框架时，不需要配置，只需要将注解处理器在build.gradle中添加依赖即可。
