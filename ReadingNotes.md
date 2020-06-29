# 《Java核心技术》
> 无后缀的浮点数值默认为 double 类型


> 我们强烈建议不要在程序中使用 char 类型，除非确实需要处理的是 UTF-16 代码单元。最好将字符串作为抽象数据类型处理

Java 采用的是 16 位的 Unicode 字符集，无法表示所有字符
```
public static void main(String[] args) {
	String s = "𠮷";
	System.out.println(s.length()); // 2
	System.out.println(s.codePointCount(0, s.length())); // 1
	System.out.println(s.charAt(0)); // ?
	System.out.println(s.substring(0, 1));// ?
	System.out.println(s.substring(0, 2));// 𠮷
	int[] codePoints = s.codePoints().toArray();
	System.out.println(new String(codePoints, 0, codePoints.length)); // 𠮷
}
  ```


> 在 Math 类中 ， 为了达到最快的性能 ， 所有的方法都使用计算机浮点单元中的例程。如果得到一个完全可预测的结果比运行速度更重要的话，那么就应该使用StrictMath类


> 解决默认方法冲突:
1)超类优先。如果超类提供了一个具体方法，超接口中同名而且有相同参数类型的默认方法会被忽略。
2)提示冲突。如果一个超接口提供了一个默认方法，另一个超接口提供了一个同名而且参数类型相同的方法（不论是否是默认方法），必须在子类中重写这个方法来解决冲突。



# 《Spring实战》
> Spring 自带多个容器实现，可以归为两类：bean工厂（BeanFactory），最简单的容器，提供基本的DI支持；应用上下文（ApplicationContext），基于 BeanFactory 构建，提供应用框架级别的服务

> ![bean生命周期](https://raw.githubusercontent.com/RobertHu0817/Markdown/master/pics/bean生命周期.png)
1. Spring 对 bean 进行实例化；
2. Spring 将值和 bean 的引用注入到 bean 对应的属性中；
3. 如果 bean 实现了 BeanNameAware 接口，Spring 将 bean 的 ID 传递给 setBeanName() 方法；
4. 如果 bean 实现了 BeanFactoryAware 接口，Spring将调用 setBeanFactory() 方法，将 BeanFactory 容器实例传入；
5. 如果 bean 实现了 ApplicationContextAware 接口，Spring 将调用 setApplicationContext() 方法，将bean所在的应用上下文的引用传入进来；
6. 如果 bean 实现了 BeanPostProcessor 接口，Spring 将调用它们的 postProcessBeforeInitialization() 方法；
7. 如果 bean 实现了 InitializingBean 接口，Spring 将调用它们的 afterPropertiesSet() 方法。类似地，如果 bean 使用 init-method 声明了初始化方法，该方法也会被调用；
8. 如果 bean 实现了 BeanPostProcessor 接口，Spring 将调用它们的 postProcessAfterInitialization() 方法；
9. 此时，bean 已经准备就绪，可以被应用程序使用，它们将一直驻留在应用上下文中，直到该应用上下文被销毁；
10. 如果 bean 实现了 DisposableBean 接口，Spring 将调用它的 destory() 接口方法。同样，如果 bean 使用 destory-method 声明了销毁方法，该方法也会被调用。


