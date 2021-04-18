# JVM 类加载器和双亲委派机制

## 类加载器

Java 虚拟机的类加载有三大步骤：加载，链接，初始化。其中加载是指查找字节流（也就是由 Java 编译器生成的 class 文件）并据此创建类的过程，这中间我们需要借助类加载器来查找字节流。Java 虚拟机提供了 3 种类加载器，分别是：启动（Bootstrap）类加载器、扩展（Extension）类加载器、应用（Application）类加载器。除了启动类加载器外，其他的类加载器都是`java.lang.ClassLoader`的子类。

- **1）启动类/引导类加载器：Bootstrap ClassLoader**

启动类加载器，也叫根类加载器。这个类加载器使用 C/C++ 语言实现的，嵌套在 JVM 内部，java 程序无法直接操作这个类。它用来加载 Java 核心类库，如：`JAVA_HOME/jre/lib/rt.jar`、`resources.jar`、`sun.boot.class.path`路径下的包，用于提供 jvm 运行所需的包。

启动类加载器比较特殊，它不是`java.lang.ClassLoader`的子类，它没有父类加载器。它加载扩展类加载器和应用程序类加载器，并成为他们的父类加载器。

出于安全考虑，启动类只加载包名为：`java`、`javax`、`sun`开头的类。

- **2）扩展类加载器：Extension ClassLoader**

Java 语言编写，由`sun.misc.Launcher$ExtClassLoader`实现，派生继承自`java.lang.ClassLoader`，父类加载器为启动类加载器。

负责加载扩展目录 (`%JAVA_HOME%/jre/lib/ext`) 下的`jar`包，用户可以把自己开发的类打包成`jar`包放在这个目录下即可扩展核心类以外的新功能。

- **3）应用程序类加载器：Application Classloader**

Java 语言编写，由`sun.misc.Launcher$AppClassLoader`实现，派生继承自`java.lang.ClassLoader`，父类加载器为启动类加载器。负责加载`CLASSPATH`环境变量所指定的`jar`包与类路径。一般来说，用户自定义的类就是由`APP ClassLoader`加载的。开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

应用程序都是由以上三种类加载器互相配合进行加载的。如果有需要，我们也可以自定义类加载器。因为 JVM 自带的 ClassLoader 只能从本地文件系统加载标准的 class 文件，如果编写了自己的 ClassLoader，便可以做到如下几点：

1）在执行代码之前，自动验证数字签名。

2）动态地创建符合用户特定需要的定制化构建类。

3）从特定的场所取得 java class，例如数据库中和网络中。

- **JVM 类加载机制**

1）全盘负责。当一个类加载器负责加载某个`Class`时，该`Class`所依赖的和引用的其他`Class`也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入。

2）父类委托。先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类。

3）缓存机制。缓存机制将会保证所有加载过的`Class`都会被缓存，当程序中需要使用某个`Class`时，类加载器先从缓存区寻找该`Class`，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成`Class`对象，存入缓存区。这就是为什么修改了`Class`后，必须重启`JVM`，程序的修改才会生效。

## 类加载方式

类加载有三种方式：

1）命令行启动应用时候由 JVM 初始化加载；

2）通过`Class.forName()`方法动态加载；

3）通过`ClassLoader.loadClass()`方法动态加载。

例子：

```java
package com.neo.classloader;
public class loaderTest { 
        public static void main(String[] args) throws ClassNotFoundException { 
                ClassLoader loader = HelloWorld.class.getClassLoader(); 
                System.out.println(loader); 
            
                //使用ClassLoader.loadClass()来加载类，不会执行初始化块 
                loader.loadClass("Test"); 
            
                //使用Class.forName()来加载类，默认会执行初始化块 
                Class.forName("Test"); 
            
                //使用Class.forName()来加载类，并指定ClassLoader，初始化时不执行静态块 
                Class.forName("Test", false, loader); 
        } 
}
```

Test 类代码：

```java
public class Test { 
    static { 
        System.out.println("静态初始化块执行了！"); 
    } 
}
```

分别切换加载方式，会有不同的输出结果。

`Class.forName()`和`ClassLoader.loadClass()`区别：

1）`Class.forName()`：将类的`.class`文件加载到`JVM`中之外，还会对类进行解释，执行类中的`static`块；

2）`ClassLoader.loadClass()`：只干一件事情，就是将.class文件加载到 JVM 中，不会执行`static`中的内容,只有在`newInstance`才会去执行`static`块。

3）`Class.forName(name, initialize, loader)`带参函数也可控制是否加载`static`块。并且只有调用了`newInstance()`方法采用调用构造函数，创建类的对象 。

## 双亲委派机制

JVM 对`class`文件采用的是按需加载的方式，当需要使用该类时，JVM 才会将它的`class`文件加载到内存中产生`class`对象。

在加载类的时候，是采用的双亲委派机制，即把请求交给父类处理的一种任务委派模式。

![0.png](https://i.loli.net/2021/03/10/3dFXlmWo8qkSK5B.png)

- **工作原理**

1）如果一个类加载器接收到了类加载请求，它自己不会先去加载，而是把这个请求委托给父类加载器去执行。

2）如果父类还存在父类加载器，则继续向上委托，一直委托到启动类加载器：`Bootstrap ClassLoader`。

3）如果父类加载器可以完成加载任务，就返回成功结果；如果父类加载失败，则由子类自己去尝试加载，如果子类加载失败就会抛出`ClassNotFoundException`异常，这就是双亲委派模式。

我们来看`ClassLoader`中`loadClass`方法：

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先判断类是否已加载过，加载过就直接返回
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //有父类加载器，调用父加载器的loadClass
                        c = parent.loadClass(name, false);
                    } else {
                        //调用Bootstrap Classloader
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    long t1 = System.nanoTime();
                    //到自己指定类加载路径下查找是否有class字节码
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

这种加载方式可以**避免类的重复加载，当父亲已经加载了该类时，就没有必要子类加载器再加载一次**。其次也考虑到安全因素，比如我们自己写的一个`java.lang.String`的类，通过双亲委派机制传递到启动类加载器，而启动类加载器在核心`Java API`发现这个名字的类，发现该类已被加载，并不会重新加载我们新写的`java.lang.String`，而直接返回已加载过的`String.class`，这样保证生成的对象是同一种类型。

## 自定义类加载器

通常情况下，我们都是直接使用 JVM 提供的类加载器。但某些情况下，我们需要自定义类加载器。比如应用是通过网络来传输 Java 类的字节码，为保证安全性，这些字节码经过了加密处理，这时系统类加载器就无法对其进行加载，这样则需要自定义类加载器来实现。

自定义一个`Person`类，代码如下：

```java
public class Person {

  private int age;
  private String name;

  //省略getter/setter方法
}
```

测试代码：

```java
public class Test {

  public static void main(String[] args) throws Exception {
    Person person = new Person();
    System.out.println("person是由" + person.getClass().getClassLoader() + "加载的");
  }
}
```

运行结果如下：
![1.png](https://i.loli.net/2021/03/10/6qNjtc7yzGYh9BI.png)

然后把`Person.class`放置在其他目录下

![2.png](https://i.loli.net/2021/03/10/H3dQKvGlLbXC7yr.png)

再运行测试代码，发现会抛出`ClassNotFoundException`异常，因为在指定路径下查找不到`class`文件。

现在，我们自定义一个类加载器，负责加载`Person`类。我们只需要继承`ClassLoader`并重写`findClass`方法即可，这里面写查找字节码的逻辑。

```java
public class PersonClassLoader extends ClassLoader {

  private String classPath;

  public PersonClassLoader(String classPath) {
    this.classPath = classPath;
  }

  private byte[] loadByte(String name) throws Exception {
    name = name.replaceAll("\\.", "/");
    FileInputStream fis = new FileInputStream(classPath + "/" + name
        + ".class");
    int len = fis.available();
    byte[] data = new byte[len];
    fis.read(data);
    fis.close();
    return data;
  }

  protected Class<?> findClass(String name) throws ClassNotFoundException {
    try {
      byte[] data = loadByte(name);
      return defineClass(name, data, 0, data.length);
    } catch (Exception e) {
      e.printStackTrace();
      throw new ClassNotFoundException();
    }
  }
}
```

测试类：

```java
public class Test {

  public static void main(String[] args) throws Exception {
    PersonClassLoader classLoader = new PersonClassLoader("/home/shenxinjian");
    Class<?> pClass = classLoader.loadClass("com.test.classloading.demo.Person");
    System.out.println("person是由" + pClass.getClassLoader() + "类加载器加载的");
  }
}
```

自定义类加载器的核心在于对字节码文件的获取，如果是加密的字节码则需要在该类中对文件进行解密。这里的测试代码并未对`class`文件进行加密，因此没有解密的过程。这里需要注意：

1）这里传递的文件名需要是类的全限定性名称，即`com.test.classloading.demo.Person `格式的，因为`defineClass`方法是按这种格式进行处理的。

2）最好不要重写`loadClass`方法，因为这样容易破坏双亲委托模式。

3）`Person`类本身可以被 `AppClassLoader`类加载，因此我们不能把`com/test/classloading/demo/Person.class`放在类路径下。否则，由于双亲委托机制的存在，会直接导致该类由`AppClassLoader`加载，而不会通过我们自定义类加载器来加载。

- **自定义类加载器作用**

1）当`class`文件不在`classPath`路径下，如上面那种情况，默认系统类加载器无法找到该`class`文件，在这种情况下我们需要实现一个自定义的`classLoader`来加载特定路径下的`class`文件来生成`class`对象。

2）当一个`class`文件是通过网络传输并且可能会进行相应的加密操作时，需要先对`class`文件进行相应的解密后再加载到 JVM 内存中，这种情况下也需要编写自定义的`ClassLoader`并实现相应的逻辑。









