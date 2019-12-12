# Step-by-Step ButterKnife II——实现注解器Syringe

下文将一步步介绍如何实现一个类ButterKnife的注解器。这里将其命名为Syringe（注射器）。以下实现的功能是ButterKnife中最常用的BindView。

## 自定义注解

新建一个Module，选择类型为Java Library，存放注解。

在该Module下，新建一个自定义注解接口
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.CLASS)
public @interface BindView {
    int value();
}
```

这是自定义注解的写法。其中

* 
@interface表示这是一个自定义注解
* 
@Target表示这个注解修饰的目标类型，这里使用的ElementType.FIELD表示该自定义注解所注解的目标是一个变量
* 
  @Retention表示这个注解的类型。

  注解分为三种，分别是源码注解、编译期注解、运行时注解，这三种注解的存在时间不同，如源码级注解只会存在于源代码中，编译期和运行期都不会有注解的标识，而此处以RetentionPolicy.CLASS表示的注解，标示的是该注解在编译期会存在，而过了编译期就被丢弃的注解。

* 
int value()：这是自定义注解的一种写法，表示该注解需要接收一个int的值，这个值也就是我们所需要的Id。

## 注解处理器

接下来定义注解处理器，新建一个Module，类型同样需要选择为Java Library。

### 依赖

在该Module的build.gradle中添加
```groovy
implementation 'com.squareup:javapoet:1.10.0'
// 这是我上一步的Module的名字，是一个自定义的命名
implementation project(":syringeannotation")
```

### 新建处理器类

新建一个类，笔者此处命名为SyringeProcessor，作为注解处理器，处理注解。

```java
public class SyringeProcessor extends AbstractProcessor
```

APT提供的注解处理器，需要继承自AbstractProcessor，我们的自定义注解处理器也当如此。

当编译使用该注解处理器的代码时，编译器会调用该注解处理器的方法process，在这个方法中我们就可以进行操作，这个方法也正是我们需要重写的方法。同时我们也需要重写一些其他的辅助方法。
```java
public class SyringeProcessor extends AbstractProcessor {

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton(BindView.class.getCanonicalName());
    }
    
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        ...
    }

}
```
首先是`getSupportedAnnotationTypes`方法。从这个方法返回的注解，将是该注解处理器会拦截下来的元素。也就是说，只有在这返回的注解，才会是这个注解处理器可以处理的。

接下来就是重写`process`方法，处理注解。

### 处理注解

在注解处理器中，将被注解的元素分为几种，分别是

| PackageElement       | 表示包                              |
| -------------------- | ----------------------------------- |
| TypeElement          | 表示类或接口                        |
| VariableElement      | 表示变量、enum 常量、参数、局部变量 |
| ExecutableElement    | 表示方法                            |
| TypeParameterElement | 表示泛型参数                        |

这些元素都继承于Element类，有一些通用的方法。常用的如：

* getKind：返回一个枚举值，表示该Element的类型，这里的类型不同于Java中的类型，这里指的是包、方法、变量等等。
* getSimpleName：返回该元素的简单名字。
* getEnclosingElement：返回封装本元素的最里层元素。如对TypeElement调用该方法，返回的是最里层的包，而对一个成员变量调用该方法，返回的则是一个TypeElement。
* getAnnotation：返回该元素的注解。

贴一下我的处理代码
```java
public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
       Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(BindView.class);
       // key-value对应的是Activity和每个Activity中注解的元素
       Map<TypeElement, List<Element>> activityWithField = new HashMap<>();

       for (Element element : elements) {
           // 改API返回的是封装ele的最内层的元素，bindview注解的是变量，所以拿到的是activity
           TypeElement activity = (TypeElement) element.getEnclosingElement();

           List<Element> bindingView = activityWithField.get(activity);
           if (bindingView == null) {
               bindingView = new ArrayList<>();
               activityWithField.put(activity, bindingView);
           }
           bindingView.add(element);
       }

       for (Map.Entry<TypeElement, List<Element>> activityWithBindingView : activityWithField.entrySet()) {
           TypeElement activity = activityWithBindingView.getKey();

           String activityNameStr = activity.getSimpleName().toString();

           List<Element> bindingViews = activityWithBindingView.getValue();

           // 构造方法
           MethodSpec.Builder constructorBuilder = MethodSpec.constructorBuilder()
                   .addModifiers(Modifier.PUBLIC)
                   .addParameter(ClassName.bestGuess(activityNameStr), "targetActivity")
                   .addStatement("this.targetActivity = targetActivity");

           // 类
           TypeSpec.Builder
                   classBuilder =
                   TypeSpec.classBuilder(activityNameStr + "_Syringe")
                           .addField(ClassName.bestGuess(activityNameStr), "targetActivity")
                           .addModifiers(Modifier.PUBLIC);

           for (Element bindingView : bindingViews) {
               String viewName = bindingView.getSimpleName().toString();

               int bindingValue = bindingView.getAnnotation(BindView.class).value();

               constructorBuilder.addStatement("targetActivity." +
                       viewName +
                       " = targetActivity.findViewById(" +
                       bindingValue +
                       ");");
           }
           classBuilder.addMethod(constructorBuilder.build());
           JavaFile
                   javaFile =
                   JavaFile.builder(((PackageElement) activity.getEnclosingElement()).getQualifiedName()
                                   .toString(),
                           classBuilder.build()).build();
           try {
               javaFile.writeTo(filer);
           } catch (IOException e) {
               e.printStackTrace();
           }

       }
       return true;
   }
```

* 首先从roundEnvironment中获取所有打了BindView注解的元素。这里的元素都是VariableElement。

* 扫一遍，将这些Element按不同的Activity分开来，每个Activity对应一组Element，这里以HashMap存储。依据是VariableElement的getEnclosingElement方法，获取的是TypeElement，也就是对应的Activity的Element。

* 对HashMap做遍历，处理每个Activity及其对应的Element。目标是生成一个这样子的类。

  ```java
  package com.fourfire.fourfirelib;
  
  public class MainActivity_Syringe {
    MainActivity targetActivity;
    public MainActivity_Syringe(MainActivity targetActivity) {
        this.targetActivity = targetActivity;
        targetActivity.mButton = targetActivity.findViewById(2131230851);
    }
  }
  ```

写法如上文的代码。这里使用的是JavaPoet的API，这是一个支持在代码中生成源代码的库，避免了拼接辅助源代码时大量的字符串操作。

* 首先生成构造方法。

  addModifiers设置的是方法的访问控制符。

  addParameter添加的是参数，其中该方法接收的第一个参数是要生成的参数的类型，第二个参数是形参名字。这里可以使用ClassName.bestGuess生成第一个参数。

  addStatement添加的是方法的内容。

* 
  生成类。

  classBuilder方法传入的是类名。

  addField添加该类的成员变量。

  addModifiers设置该类的访问控制符

  接下来遍历该Activity对应的一组Element，即被注解的变量，获取其注解的值，调用构造方法Builder的addStatement生成为这些变量赋值的代码。

  最后将构造方法添加入类的Builder中。

* 
  使用封装好的IO方法写入。

  需要注意的是，此处需要传入包名，包名可以通过activity获取到包，然后再获取该包的全名，则可以得到。

* 
return true表示已经处理的注解不需要其他处理器再次处理。

### 运行注解处理器

注解处理器写好了，但编译器如何知道要使用该注解处理器呢？这就需要配置。配置文件可以通过编辑

META-INF
文件。而google提供了一个库，可以自动生成。

在processor Module的build.gradle依赖中添加

```groovy
implementation 'com.google.auto.service:auto-service:1.0-rc2'
```

然后对写好的处理器类打注解

```java
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@AutoService(Processor.class)
public class SyringeProcessor extends AbstractProcessor {
	...
}
```

* SupportedSourceVersion代表的是支持的Java版本
* AutoService就是配置注解处理器，有了这个注解，会自动生成配置文件，而编译器在编译时就会使用该注解处理器。

当然处理processor Module的配置，对要使用该库的Module也要配置依赖。在app Module的build.gradle依赖中添加
```groovy
// 这是自定义注解所在的Module
implementation project(":syringeannotation")
// processor Module
annotationProcessor project(":syringeprocessor")
```

老版本中使用的不是annotationProcessor，而是apt，但现在的gradle已经淘汰了apt，也不需要做过多的配置，只需要这一行则解决问题。

build一下项目，在app Module的/build/generated/source/apt/debug下即可看到生成的代码

至此注解处理器则写好了。但笔者在实现这一段的时候有个坑，就是发生了依赖冲突。autoservice库依赖于guava，而androidx库也依赖与guava的一个module，发生了冲突。如果读者也出现了冲突，在app的build.gradle中，对androidx的guava的listenablefuture移除即可。
```java
implementation('androidx.appcompat:appcompat:1.1.0-alpha04') {
    exclude group: 'com.google.guava', module: 'listenablefuture'
}
```

## 绑定接口

下面将实现绑定接口，也就是ButterKnife.bind的行为。

由于我将自己实现的整个库命名为Syringe，bind方法我也就命名为syringe，注入的意思。

同样新建一个Module，选择的是Android Library。新建一个类
```java
public class Syringe {
    public static void sringe(Activity activity) {
        String activityName = activity.getClass().getName();
        String generateName = activityName + "_Syringe";
        try {
            Class.forName(generateName).getConstructor(activity.getClass()).newInstance(activity);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}
```

其实其实现非常简单，通过反射，调用到之前生成的辅助类的构造方法，则一切完成。

## 使用

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    @BindView(R.id.mButton)
    Button mButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Syringe.sringe(this);

        mButton.setOnClickListener(this);
    }
    
    @Override public void onClick(View v) {
        Toast.makeText(this, "toast", Toast.LENGTH_SHORT).show();
    }
}
```
使用和ButterKnife完全相同。而这里笔者没有实现click注解，所以还是手动setOnClickListener，事实上在ButterKnife中，这个也是通过注解处理可以实现，原理基本一致。