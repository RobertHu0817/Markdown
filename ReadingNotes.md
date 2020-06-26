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
