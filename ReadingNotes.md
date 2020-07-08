# 《Java核心技术》

> 无后缀的浮点数值默认为 double 类型

> 我们强烈建议不要在程序中使用 char 类型，除非确实需要处理的是 UTF-16 代码单元。最好将字符串作为抽象数据类型处理

Java 采用的是 16 位的 Unicode 字符集，无法表示所有字符

```
public static void main(String[] args) throws NoSuchFieldException, SecurityException, IllegalArgumentException, IllegalAccessException {
	String s = "𠮷";
	System.out.println(s.length()); // 2
	System.out.println(s.codePointCount(0, s.length())); // 1
	System.out.println(s.charAt(0)); // ?
	System.out.println(s.substring(0, 1));// ?
	System.out.println(s.substring(0, 2));// 𠮷
	int[] codePoints = s.codePoints().toArray();
	System.out.println(new String(codePoints, 0, codePoints.length)); // 𠮷
	
	// String 底层使用的是 char[]
	Field valueField = String.class.getDeclaredField("value");
	valueField.setAccessible(true);
	char[] v = (char[]) valueField.get(s);
	System.out.println(v.length); // 2
	for (char c : v) {
		System.out.println(c); // 共两个?
	}
	System.out.println(v); // 𠮷
}
```

> 在 Math 类中 ， 为了达到最快的性能 ， 所有的方法都使用计算机浮点单元中的例程。如果得到一个完全可预测的结果比运行速度更重要的话，那么就应该使用StrictMath类

> 解决默认方法冲突:
> 1)超类优先。如果超类提供了一个具体方法，超接口中同名而且有相同参数类型的默认方法会被忽略。
> 2)提示冲突。如果一个超接口提供了一个默认方法，另一个超接口提供了一个同名而且参数类型相同的方法（不论是否是默认方法），必须在子类中重写这个方法来解决冲突。



# 《Spring实战》-V4

> Spring 自带多个容器实现，可以归为两类：bean工厂（BeanFactory），最简单的容器，提供基本的DI支持；应用上下文（ApplicationContext），基于 BeanFactory 构建，提供应用框架级别的服务

> ![bean生命周期](https://raw.githubusercontent.com/RobertHu0817/Markdown/master/pics/bean生命周期.png)
> 1. Spring 对 bean 进行实例化；
> 2. Spring 将值和 bean 的引用注入到 bean 对应的属性中；
> 3. 如果 bean 实现了 BeanNameAware 接口，Spring 将 bean 的 ID 传递给 setBeanName() 方法；
> 4. 如果 bean 实现了 BeanFactoryAware 接口，Spring将调用 setBeanFactory() 方法，将 BeanFactory 容器实例传入；
> 5. 如果 bean 实现了 ApplicationContextAware 接口，Spring 将调用 setApplicationContext() 方法，将bean所在的应用上下文的引用传入进来；
> 6. 如果 bean 实现了 BeanPostProcessor 接口，Spring 将调用它们的 postProcessBeforeInitialization() 方法；如果 bean 实现了 InitializingBean 接口，Spring 将调用它们的 afterPropertiesSet() 方法。类似地，如果 bean 使用 init-method 声明了初始化方法，该方法也会被调用；
> 7. 如果 bean 实现了 BeanPostProcessor 接口，Spring 将调用它们的 postProcessAfterInitialization() 方法；
> 8. 此时，bean 已经准备就绪，可以被应用程序使用，它们将一直驻留在应用上下文中，直到该应用上下文被销毁；如果 bean 实现了 DisposableBean 接口，Spring 将调用它的 destory() 接口方法。同样，如果 bean 使用 destory-method 声明了销毁方法，该方法也会被调用。

> （@Autowired）不管是构造器、Setter 方法还是其他的方法，Spring 都会尝试满足方法参数上所声明的依赖。假如有且只有一个 bean 匹配依赖需求的话，那么这个 bean 将会被装配进来。

> SpEL 表达式要放到“#{...}”中，这与属性占位符有些类似，属性占位符需要放到“${...}”之中。

> 1. 表示字面值——整数、浮点数、字符串以及布尔值；
> 2. 引用 bean、属性和方法——#{student.getName()?.toUpperCase()};
> 3. 在表达式中使用类型——#{T(java.lang.Math).PI}
> 4. ...(挺复杂的，没必要)

> Spring 切面可以应用5种类型的通知：
> - 前置通知
> - 后置通知
> - 返回通知
> - 异常通知
> - 环绕通知

> 织入（Weaving）是把切面应用到目标对象并创建新的代理对象的过程。切面在指定的连接点被织入到目标对象中。在目标对象的生命周期里有多个点可以进行织入：
> - 编译期：切面在目标类编译时被织入。这种方式需要特殊的编译器。AspectJ 的织入编译器就是以这种方式织入切面的。
> - 类加载期：切面在目标类被加载到 JVM 时被织入。这种方式需要特殊的类加载器，它可以在目标类被引入应用之前增强该目标类的字节码。AspectJ5 的加载时织入（load-time weaving，LTW）就支持以这种方式织入切面
> - 运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP 容器会为目标对象动态地创建一个代理对象。Spring AOP 就是以这种方式织入的。

> Spring 提供了4种类型的 AOP 支持：
> - 基于代理的经典 Spring AOP——有些过时，相较基于注解的 AOP 显得非常笨重和过于复杂
> - 纯 POJO 切面——xml 配置，借助于 Spring 的 aop 命名空间
> - @AspectJ 注解驱动的切面——借鉴于 AspectJ 注解切面，本质上依然是 Spring 基于代理的 AOP
> - 注入式 AspectJ 切面
> 前三种都是 Spring AOP 实现的变体，构建在动态代理基础之上，局限于方法拦截；如果你的AOP 需求超过了简单的方法调用（如构造器或属性拦截），那么你需要考虑使用第四种类型，使用 AspectJ 来实现切面

> Spring AOP 几点关键知识：
> - Spring 通知是 Java 编写的——虽然 AspectJ 现在支持基于注解的切面，但 AspectJ 最初是以 Java 语言扩展的方式实现的。这种方式优点在于通过特有的 AOP 语言，我们可以获得更强大和细粒度的控制，以及丰富的 AOP 工具集，但缺点在于我们需要额外学习新的工具和语法
> - Spring 在运行时通知对象——Spring 在运行时才创建代理对象，通过在代理类中包裹切面，把切面织入到 Spring 管理的 bean 中
> - Spring 只支持方法级别的连接点
Spring AOP：动态代理，可以基于 JDK 动态代理（Proxy 类，必须实现接口，底层通过反射实现）或者 CGLIB（底层通过继承实现） 实现，每次运行生成代理类，性能稍差，且只能作用于方法级别
AspectJ：静态代理，需借助特定编译器，性能更好，能获得更细粒度的控制

> Spring 仅支持 AspectJ 切点指示器(pointcut designator)的一个子集，使用其来定义 Spring 切面：
<table>
    <tr>
        <td>AspectJ 指示器</td>
        <td>描述</td>
    </tr>
    <tr>
        <td>arg()</td>
    	<td>限制连接点匹配参数为指定类型的执行方法</td>
    </tr>
    <tr>
        <td>@args()</td>
    	<td>限制连接点匹配参数由指定注解标注的执行方法</td>
    </tr>
    <tr>
        <td>execution()</td>
    	<td>用于匹配是连接点的执行方法</td>
    </tr>
    <tr>
        <td>this()</td>
    	<td>限制连接点匹配 AOP 代理的 bean 引用为指定类型的类</td>
    </tr>
    <tr>
        <td>target</td>
    	<td>限制连接点匹配目标对象为指定类型的类</td>
    </tr>
    <tr>
        <td>@target()</td>
    	<td>限制连接点匹配特定的执行对象，这些对象对应的类要具有指定类型的注解</td>
    </tr>
    <tr>
        <td>within()</td>
    	<td>限制连接点匹配指定的类型</td>
    </tr>
    <tr>
        <td>@within()</td>
    	<td>限制连接点匹配指定注解所标注的类型（当使用 Spring AOP 时，方法定义在由指定注解所标注的类里）</td>
    </tr>
     <tr>
        <td>@annotation</td>
    	<td>限定匹配带有指定注解的连接点</td>
    </tr>
</table>
> 只有 execution 指示器是实际执行匹配的，而其他的指示器都是用来限制匹配的。这说明 execution 指示器是我们在编写切点定义时最主要使用的指示器。在此基础上，我们使用其他指示器来限制所匹配的切点。

> Spring 使用 AspectJ 注解来声明通知方法：
> * @After
> * @AfterReturning
> * @AfterThrowing
> * @Around
> * @Before

 
