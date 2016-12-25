---
layout: post
title:  Java class和interface初始化的几种条件
date:   2016-02-15
---

<p class="intro"><span class="dropcap">J</span>ava中的类和接口在某些情况下看似被引用了但不会进行初始化动作，比如：</p>

{% highlight java %}

class Super {
	static int taxi = 1729;
}
class Sub extends Super {
	static { System.out.print("Sub "); }
}
class Test {
	public static void main(String[] args) {
	    System.out.println(Sub.taxi);
	}
}
{% endhighlight %}
上述代码中的子类Sub虽然被引用，但实际上是用的Sub.taxi这个引用变量，而这个引用变量是父类中的一个静态变量，所以没有触发Sub的初始化动作，所以输出：1729，而不是 Sub 1729。下面举一个接口的例子：
{% highlight java %}

interface I {
	int i = 1, ii = Test.out("ii", 2);
}
interface J extends I {
	int j = Test.out("j", 3), jj = Test.out("jj", 4);
}
interface K extends J {
	int k = Test.out("k", 5);
}
class Test {
	public static void main(String[] args) {
	    System.out.println(J.i);
	    System.out.println(K.j);
	}
	static int out(String s, int i) {
	    System.out.println(s + "=" + i);
	    return i;
	}
}

{% endhighlight %}

输出：
1
j=3
jj=4
3
  上述例子中J.i是一个常量，不会引起I的初始化，而K.j不是一个常量，需要J的初始化，但是不会引起父类接口I的初始化，也不会引起K自身的初始化。
  可以发现，尽管有些类和接口被引用了，却不会导致初始化动作，还有一个常见的例子就是 Object o = null;对象o也未被初始化。下面总结一下所有引起类和接口T初始化的条件，
<ul>
<li> 类T的一个实例被创建（new）	</li>
<li> 类T的静态方法被调用</li>
<li> T的静态变量被引用或赋值</li>
<li> T的子类被初始化</li>
<li> 类T包含main方法</li>
</ul>
详细参考oracle java官方文档：[http://docs.oracle.com/javase/specs/jls/se7/html/jls-12.html#jls-12.5][1]

[1]:	http://docs.oracle.com/javase/specs/jls/se7/html/jls-12.html#jls-12.5
