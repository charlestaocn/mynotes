### 双亲(parent)委派

> Java虚拟机对class文件采用的是 `按需加载`的方式，也就是当需要使用该类时才会将他的class文件加载到内存中生成class对象，而且加载某个类的class文件时，Java虚拟机采用的而是双亲委派模式。即把请求交由父类加载器处理，它是一种任务委派模式。

![[tomcat/_resources/【第三篇】Tomcat进阶篇(1)/d843fe2719af676aa2d1c61848e3ed60_MD5.png]]

双亲委派的工作原理：

1. 如果一个类加载器收到了类加载的请求，它并不会自己先去加载，而且把这个请求委托给父类的加载器去执行。
2. 如果父类加载器还存在其父类加载器，则进一步向上委托，依次递归请求，最终将请求流转到顶层的启动类加载器
3. 如果父类加载器可以完成类加载任务，就成功返回，如果父类无法完成任务，子加载器才会尝试自己去加载。

举个简单的例子，我们自定义一个java.lang.String类，然后添加对应的main方式执行。

```java
public class String {
    public static void main(String[] args) {
        System.out.println("自定义String类");
    }
}
```

报错信息为：

![[tomcat/_resources/【第三篇】Tomcat进阶篇(1)/026eb82344e152e51893a9ea3ee86f6f_MD5.png]]

原因就是双亲委派机制通过BootstrapClassLoader加载的是java包下的String，而不会加载我们自定义的。

双亲委派机制的有点：

1. 避免类的重复加载
2. 保护程序的安全，防止核心API被随意的篡改
### .class 加密解密
1. 将 正常编译后的文件进行加密， 让正常的 appclassload 无法加载
2. 自定义classloader ，添加 解密方法，解密方法中校验 密码(解密串文件) 正确的话正常加载
### 类隔离技术/内容更改 

```java
new ClassLoader() {  
    public Class<?> loadC(String s, boolean needEnhance) throws ClassNotFoundException {  
        if (needEnhance) {  
            return loadClass(s);  
        } else {  
            return super.loadClass(s);  
        }  
    }  
  
    @Override  
    public Class<?> loadClass(String name) throws ClassNotFoundException {  
        //getClassByte  
        //doSomethingBad...        //defineClass..and return;        return null;  
    }  
}.loadC("clzName", false);
```