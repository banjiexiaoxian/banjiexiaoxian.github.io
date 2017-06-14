---
layout: post
title: "Cantor展开、全排列问题、魔板问题(JAVA实现)"
date: 2017-06-14 21:41:03 +0800
comments: true
categories: Basic Algrothims
---
# Cantor展开、全排列问题、魔板问题（JAVA实现）

本文由全排列问题的递归和非递归写法入手，引出Cantor展开的公式及其应用，最后讨论Cantor数的经典应用之魔板问题

- **全排列问题**
- **Cantor展开**
- **魔板问题**


-------------------
##目录

[TOC]


## 问题：给定字符串S[0…N-1]，设计算法，枚举S的全排列

以一个简单的示例来表示解题过程
示例 枚举0123的全排列
0123 0132 0213 0231 0312 0321
1023 1032 1203 1230 1302 1320
2013 2031 2103 2130 2301 2310
3012 3021 3102 3120 3201 3210
手动写出这些序列的时候，实际上是脑补了一个树形结构，如下：
![这里写图片描述](http://img.blog.csdn.net/20170220113519218?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGl1eWFubGluZ19jcw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
画出这个树的过程实际上是一个深度搜索的过程，每次到达叶子节点时就产生一个输出，之后再回溯，搜索下一个叶子节点。
深度搜索的过程实际上就是一个入栈出栈的过程，也就是一个递归过程。
###全排列之递归解法
使用递归时，代码结构是很清晰，也容易理解，便不再赘述，直接上代码。



```
//无重复的全排列递归写法
public class Permutation {
	public static final int N = 4;
	public static void main(String[] args){
		//初始化
		int[] sequence = new int[N];
		for(int i = 0;i < N;i++){
			sequence[i] = i+1;
		}
		permutation(sequence,N,0);
	}
	
	private static void print(int[] sequence){
		for(int i : sequence){
			System.out.print(i);
		}
		System.out.println();
	}
	
	private static void swap(int[] sequence,int i, int j){
		int tmp = sequence[i];
		sequence[i] = sequence[j];
		sequence[j] = tmp;
	}
	//固定前n位的全排列
	public static void permutation(int[] sequence, int size, int n){
		if(n == size - 1) print(sequence);
		
		for(int i = n; i < size;i++){
			//用其他位来交换第n位
			swap(sequence,i,n);
			permutation(sequence,size,n+1);
			swap(sequence,i,n);
		}
	}
}
```
当出现重复元素时，在用其他位交换第n位时，会有相同的两个位均与n位发生了交换。所以我们需要在交换时判断该位是否在之前已经被放置到n位过。
为了判断某个元素是否已经被访问过，另增加一个duplication数组。
与无重复元素的序列全排列相比，仅在permutation函数中增加了一个参数及两行代码。
代码如下
```
//有重复的递归写法
public class PermutationWithDuplicate {
	public static final int N = 4;
	
	public static void main(String[] args){
		//初始化
		int[] sequence = {1,2,3,4};
		boolean[] duplication = new boolean[N];
		permutation(duplication,sequence,N,0);
	}
	
	private static void print(int[] sequence){
		for(int i : sequence){
			System.out.print(i);
		}
		System.out.println();
	}
	
	private static void swap(int[] sequence,int i, int j){
		int tmp = sequence[i];
		sequence[i] = sequence[j];
		sequence[j] = tmp;
	}
	
	public static void permutation(boolean[] duplication,int[] sequence,int size,int n){
		
		//遍历每一根隐式树时，duplication的状态应该是要刷新的
		//所以duplication不能作为全局变量，这样会导致所有递归状态共用一个duplication
		
		//如果作为局部变量，那么每次调用permutation都需要重新创建这个数组，并且这个数组不会被全部访问到，耗费了时间和空间
		//所以作为参数在递归状态间传递，注意在回溯时需要恢复第n位，以及duplication数组。
		//因为非基本数据类型的参数都是传址引用，如果不恢复duplication的值，那么会导致所有递归状态共用一个duplication
//		boolean[] duplication = new boolean[N];
		if(n == size-1){
			print(sequence);
		}
		for(int i = n;i < size;i++){
			if(!duplication[(sequence[i]%N)]){
				duplication[(sequence[i]%N)] = true;
				swap(sequence,i,n);
				permutation(duplication,sequence,size,n+1);
				swap(sequence,i,n);
				duplication[(sequence[i]%N)] = false;
			}
		}
	}
}
```

###全排列之非递归解法

如果是对1234字符串，枚举其所有排列。相当于就是把由1、2、3、4构成的数字全部列举出来。既要列举，自然有个顺序，不妨从小到大列举。
1234 1243 1324 1342 1423 1432
2134 2143 2314 2341 2413 2431
3124 3142 3214 3241 3412 3421
4123 4132 4213 4231 4312 4321 

每一次迭代，我们都希望获取比当前的序列稍大的序列。而比当前的序列稍大的序列仅仅只与当前的序列相关，也就是通过当前的permutation，我们能立刻得到next permutation.了解了这个原理后，程序的框架便出来了。

```
while(hasNextPermutation){
	getNextPermutation();
	print;
}
```
可以看到，当某个序列从首位到末位是依次递减时，是不可能再增大的。
反之，当某个序列的低位比高位大时，可以将低位和高位交换，交换后，将此高位后的所有位数重新按次序排列得到新的序列。举例来讲，比如1342，从末位2往前遍历，4比3大，将4与3交换变为1432，再将4后面的32按从小到大的次序变为23，最后得到next permutation为1423。
代码如下：

```
public class NextPermutation {
	public static final int N = 4;
	public static void main(String[] args){
		//初始化
		int[] sequence = new int[N];
		for(int i = 0;i < N;i++){
			sequence[i] = i+1;
		}
		permutation(sequence);
	}
	public static void permutation(int[] sequence){
		print(sequence);
		int[] tmp = getNextPermutation(sequence);
		while(null != tmp){
			print(tmp);
			tmp = getNextPermutation(tmp);
		}
	}
	private static void print(int[] sequence){
		for(int i : sequence){
			System.out.print(i);
		}
		System.out.println();
	}
	
	private static void swap(int[] sequence,int i, int j){
		int tmp = sequence[i];
		sequence[i] = sequence[j];
		sequence[j] = tmp;
	}
	
	public static int[] getNextPermutation(int[] sequence){
		int scriptOfNeedSwap = 0;
		int i;
		//从后往前找
		for(i = N - 1;i >= 1;i--){
			if(sequence[i] > sequence[i-1]){
				scriptOfNeedSwap = i - 1;
				break;
			}
		}
		
		if(i == 0) return null;
		//找到第一个递减的位置，交换
		for(i = N - 1;i > scriptOfNeedSwap;i-- ){
			if(sequence[i] > sequence[scriptOfNeedSwap]){
				swap(sequence,i,scriptOfNeedSwap);
				break;
			}
		}
		//可以看到一个规律，交换之后后面的几位均是降序的，此时只需要对其进行翻转即可
		//翻转
		for(i = N-1;i > scriptOfNeedSwap;i--){
			swap(sequence,i,++scriptOfNeedSwap);
			if(i <= scriptOfNeedSwap) break;
		}
		
		return sequence;
	}
}
```
本解法对于有重复元素的序列是完全适用的，按从小到大的顺序将序列打印出来，天然就能达到去重的目的。

## 问题：如何计算N个无重复元素的某个排列是第几个排列
如何求解这个问题。很自然的想到，使用从小到大的顺序枚举出一个数列的所有排列，直到匹配目标排列。
然而Cantor展开使用公式解决了这个问题。
###Cantor展开
关于Cantor展开的原理这里不作详细介绍。Cantor展开的公式为：

> X=an*(n-1)!+an-1*(n-2)!+...+ai*(i-1)!+...+a2*1!+a1*0!

使用示例来帮助理解利用Cantor展开如何求解本问题。
假如序列s=[“A”,”B”,”C”,”D”]
需要求序列s'=[“D”,”A”,”B”,”C”]是序列s的全排列中的第几个排列，记作X(s');

```
X(s’) = 	3（在序列DABC中有3个比D小）*3!+
			0（在剩下的序列ABC中有0个比A小）*2！+
			0（在剩下的序列BC中有0个比B小）*1！+
			0（在剩下的序列C中有0个比C小）*0！
	=18
```


那么序列s’是s的全排列中第18个排列（从0开始计数）；
同理可以根据s算第18个排列序列。
> 总结：康托展开是一个全排列到一个自然数的一一映射



##魔板问题

在下面对求解魔板问题的过程中应用了隐式树的思想以及Cantor展开。
问题描述：
在魔方风靡全球之后不久，Rubik先生发明了它的简化版――魔板。魔板 由8个同样大小的方块组成，每个方块颜色均不相同，可用数字1-8分别表示。任一时刻魔板的状态可用方块的颜色序列表示：从魔板的左上角开始，按顺时针方 向依次写下各方块的颜色代号，所得到的数字序列即可表示此时魔板的状态。例如，序列(1,2,3,4,5,6,7,8)表示魔板状态为： 
　　1 2 3 4 
　　8 7 6 5 
　　对于魔板，可施加三种不同的操作，具体操作方法如下： 
　　A: 上下两行互换,如上图可变换为状态87654321
　　B: 每行同时循环右移一格,如上图可变换为41236785 
　　C: 中间4个方块顺时针旋转一格,如上图可变换为17245368 
　　给你魔板的初始状态与目标状态，请给出由初态到目态变换数最少的变换步骤，若有多种变换方案则取字典序最小的那种。

在杭电的OJ上有这样一个题，读者可以作为练习。[题目链接](http://acm.split.hdu.edu.cn/showproblem.php?pid=1430)
###解题思路
 魔板的每一种状态，实际上是序列12345678的一种全排列，最多可以有8!=40320种状态。
 **而魔板问题则是求解状态x最快可以经过多少个状态到达状态y。**
 我们也可以脑补出一棵三叉树，编号为1的状态经过A,B,C三种行为到达状态2,3,4。2,3,4再分别经过A,B,C三种行为扩展出更多的状态。
与最开始的全排列问题不同的是，在遍历这棵树的时候，我们采用的是广度优先搜索的方法，将这棵树做层次遍历，从根节点出发，到达某个节点所需要的最少的变换次数则是这个节点所在的层数。所以，当做广度优先搜索遍历，找到第一个满足条件的状态时，程序就可以返回了。
确定了广度优先搜索的算法，便可以确定数据结构采用先入先出的队列了。每扩展出新的节点，便将这些新的节点加入到队列中，等待被取出后再度扩展。
在遍历过程中，我们可以考虑做一些优化。

 - 判重。判断某个节点的状态是否已经遍历过，若已经遍历过，则无需再扩展此节点
 - 剪枝。 AA，BBBB，CCCC这样的变换路径都是没有必要的，会变回原本的状态
 
由Cantor展开，可以将魔板的每一钟状态映射为一个自然数，这就可以作为判重的依据。而计算Cantor数需要计算阶乘，对阶乘进行优化也是必须的，可以用一个hash数组将需要计算的阶乘事先保存下来。

具体代码如下：

```
public class MoBanProblem {
	static int[] hash = {1,1,2,6,24,120,720,5040};
	public static void main(String args[]){
		String number = "12345678";
		String end = "34512678";
		String rs = null;
		number = preProcess(number);
		end = preProcess(end);
		rs = bfs(number,end);
		if(rs != null){
			System.out.println(rs);
		}else{
			System.out.println("无法变换");
		}
		
		
	}
	//由于输入的字符串与魔板上字符的顺序不一致，所以需要对数据进行预处理，将后四位翻转
	private static String preProcess(String number){
		return number.substring(0,4)+number.substring(7,8)+number.substring(6,7)+number.substring(5,6)+number.substring(4,5);
	}
	//上下两行互换
	public static String fun_A(String number){
		String a = number.substring(4,8) + number.substring(0,4);
		return a;
	}
	//每行同时循环右移一格
	public static String fun_B(String number){
		String a = number.substring(3,4)+number.substring(0,3)+number.substring(7,8)+number.substring(4,7);
		return a;
	}
	
	//中间四个方块顺时针旋转一格
	public static String fun_C(String number){
		String a = number.substring(0,1)+number.substring(5,6)+number.substring(1,2)+number.substring(3,4)+
				number.substring(4,5)+number.substring(6,7)+number.substring(2,3)+number.substring(7,8);
		return a;
	}
	
	private static int getA(String status,int i){
		int rs = 0;
		for(int k = i+1;k < status.length();k++){
			if(status.charAt(k) < status.charAt(i)){
				rs++;
			}
		}
		return rs;
	}
	
	
	//计算当前状态的cantor序号
	public static int cantor(String status){
		int[] a = new int[8];
		int rs = 0;
		for(int i = 0, j = 7;i < status.length();i++,j--){
			a[j] = getA(status,i);
		}
		for(int i = 7;i >= 0;i--){
			rs += a[i]*hash[i];
		}
		return rs;
	
	}
	
	
	public static boolean matches(String a,String b){
		if(a.length() != b.length()) return false;
		if(a.equals(b)) return true;
		else  return false;
	}
	
	
	public static String bfs(String start,String end){
		//某个序列状态是否已被搜索过
		boolean[] visited = new boolean[40320];
		String[] ans = new String[40320];
		Queue<String> status = new Queue<String>();
		status.enqueue(start);
		while(!status.isEmpty()){
			String tmp = status.dequeue();
			if(matches(tmp,end)){
				return ans[cantor(tmp)];
			}
			if(!visited[cantor(tmp)]){
				if(ans[cantor(tmp)] == null) ans[cantor(tmp)] = "";
			    if(ans[cantor(tmp)] == "" || ans[cantor(tmp)].substring(ans[cantor(tmp)].length()-1,ans[cantor(tmp)].length()) != "A"){
					String fun_A_tmp = fun_A(tmp);
					status.enqueue(fun_A_tmp);
					ans[cantor(fun_A_tmp)] = ans[cantor(tmp)] + "A";
				}
				if(ans[cantor(tmp)].length() < 3 ||ans[cantor(tmp)].substring(ans[cantor(tmp)].length()-3,ans[cantor(tmp)].length()) != "BBB"){
					String fun_B_tmp = fun_B(tmp);
					status.enqueue(fun_B_tmp);
					ans[cantor(fun_B_tmp)] = ans[cantor(tmp)] + "B";
				}
				if(ans[cantor(tmp)].length() < 3 ||ans[cantor(tmp)].substring(ans[cantor(tmp)].length()-3,ans[cantor(tmp)].length()) != "CCC"){
					String fun_C_tmp = fun_C(tmp);
					status.enqueue(fun_C_tmp);
					ans[cantor(fun_C_tmp)] = ans[cantor(tmp)] + "C";
				}
			}
			visited[cantor(tmp)] = true;
			
			
		}
		return null;
		 
	}
	
}
```

 
 >总结：将全排列问题抽象为状态的枚举，将魔板问题抽象为状态的迁移是很关键的。Cantor展开将某个状态映射为某个自然数，在查找以及计数问题中可以起到很大的作用。
 


----------


**头一次认真写博文**
**谢谢阅读，欢迎指正~**
**若有同类问题希望能帮忙补充。**






