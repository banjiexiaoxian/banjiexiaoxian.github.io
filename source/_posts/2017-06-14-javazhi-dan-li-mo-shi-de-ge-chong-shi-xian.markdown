---
layout: post
title: "Java之单例模式的各种实现"
date: 2017-03-14 16:07:05 +0800
comments: true
categories: JDK
---
最近连续在各种群里、博客里看到单例模式的讨论。根据我的理解总结一下：
先直接说结论：最优雅最简洁最稳的方法是使用枚举实现单例模式。
<!—more—>
##饿汉式

```
//无懒加载
//在类加载时初始化唯一的实例对象，由jvm在多线程环境时保证线程安全
//增加了初始化的时间和内存开销
public class SingleDog {
	private static final SingleDog instance = new SingleDog();
	
	private SingleDog(){}
	
	public SingleDog getInstance(){
		return instance;
	}
}
```

##懒汉式

```
//懒加载
//因为实例化在getInstance里执行，所以每次访问instance，都需要获取同步锁，比较笨重；
public class SingleDog {
private static SingleDog instance = null;

private SingleDog(){}

synchronized public static SingleDog getInstance(){
	//判断是否需要对Instance进行初始化
	if(instance == null){
		instance = new SingleDog();
	}
	return instance;
	
}
```
##懒汉式多线程环境下优化 之 双重检查锁定

```
//懒加载(双重检查锁定)
//多线程访问时，只在可能需要对Instance进行初始化时获取锁，大多数时候直接读取Instance即可
public class SingleDog {
	//需用volatile关键字保证对这个变量所做的操作是所有线程可见的
	private volatile static SingleDog instance = null;
	
	private SingleDog(){}
	
	public static SingleDog getInstance(){
		//判断是否需要对Instance进行初始化
		if(instance == null){
			synchronized(SingleDog.class){
				//对于已经获取到锁的其他线程，再次判断Instance是否需要进行初始化
				if(instance == null){
					instance = new SingleDog();
				}
			}
		}
		return instance;
		
	}
//双重检查锁定也不能避免使用重度锁synchronized，在获取和释放锁的过程中会有性能损耗；同时需要两个if判断来确保只有一个实例，程序逻辑比较复杂
```

##懒汉式多线程环境下优化 之 静态内部类实现

```

//与第一个饿汉式相比，使用一个私有静态内部类来代替SingleDog类，持有instance实例，达到了懒加载的目的。
//摆脱了重度锁synchronized
//那么多线程环境下一个类是否会被初始化多次呢？
//jvm在类加载时会确保线程的安全，如果多个线程去初始化一个类，只会有第一个线程被执行，其他线程都会被阻塞而且不会再次进入到类的初始化中去。同一个类加载器下，一个类只会被初始化一次
public class SingleDog {
	private SingleDog() {}
	
	private static class SingleHolder{
		private static SingleDog instance = new SingleDog();
	}
	
	public static SingleDog getInstance(){
		return SingleHolder.instance;
	}
}
```
##有什么办法破坏上述单例模式
###反射
虽然构造函数已经被限定为private了，但是只要有构造函数存在，就可以通过Java反射机制获取到它，还能强制性的setAccessble，将其设定为可访问，从而利用构造函数再生成一个新的实例，破坏单例模式。部分代码如下：

```
		SingleDog s1 = SingleDog.getInstance();
		Class<SingleDog> cls = (Class<SingleDog>) s1.getClass();
		Constructor<SingleDog> cons = cls.getDeclaredConstructor(new Class[] {});
		cons .setAccessible(true);
		SingleDog s2 = cons.newInstance(new Object[] {});
		System.out.println(s1 == s2);//false
```
###反序列化
当将Java对象序列化后，再反序列时，readObject方法会自动创建一个新的实例。天然破坏了被序列化对象的单例模式。
不过这个问题很好解决，只需要在类中添加一个方法就行：

```
public class SingleDog {
	private SingleDog() {}
	
	private static class SingleHolder{
		private static SingleDog instance = new SingleDog();
	}
	
	public static SingleDog getInstance(){
		return SingleHolder.instance;
	}
	//划重点！
	public Object readResolve() {
        return instance;
    }
}

```
##枚举实现单例模式

```
//枚举实现单例模式
//防反射、防反序列化、线程安全（实例在类初始化期间就已经创建）
//枚举类实际上是继承于java.lang.Enum的一个抽象类，所以是无法用反射获取到其构造方法的
enum SingleDog {
    INSTANCE;
	public static SingleDog getInstance() {
		return INSTANCE;      
	} 
}
```


----------
相关文章：
[深度分析 Java 的枚举类型：枚举的线程安全性及序列化问题](http://blog.jobbole.com/94074/)
[ 浅谈使用单元素的枚举类型实现单例模式 ](http://blog.csdn.net/huangyuan_xuan/article/details/52193006)
[单例模式那些坑](http://www.th7.cn/Program/java/201511/682115.shtml)

