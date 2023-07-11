---
title: 算法面试/基础
date: 2021-10-09 20:42:52.761
updated: 2021-10-09 20:46:21.19
url: /archives/albasic
categories: 
tags: 基础 | Java | 算法
---

### NC1 大数加法
题目:

        以字符串的形式读入两个数字，编写一个函数计算它们的和，以字符串形式返回。
    （字符串长度不大于100000，保证字符串仅由'0'~'9'这10种字符组成）
```java
    while (i >= 0 || j >= 0 || carry != 0) {
        int x = i < 0 ? 0 : s.charAt(i--) - '0';
        int y = j < 0 ? 0 : t.charAt(j--) - '0';
        int sum = x + y + carry;
        sb.append(sum % 10);//添加到字符串尾部
        carry = sum / 10;
    }
```
---
### NC2 重排链表
给定一个单链表 `L` 的头节点 `head` ，单链表 `L` 表示为：

` L0 → L1 → … → Ln-1 → Ln `
请将其重新排列后变为：

```
L0 → Ln → L1 → Ln-1 → L2 → Ln-2 → …
```

不能只是单纯的改变节点内部的值，而是需要实际的进行节点交换。

```java
 List<ListNode> list = new ArrayList<>();
        while(head!= null){
            list.add(head);
            head=head.next;
        }
        int i =0;
        int j = list.size()-1;
        while(i<j){
            list.get(i).next = list.get(j);
            i++;
            if(i==j) break;
            list.get(j).next = list.get(i);
            j--;
        }
        list.get(i).next = null;
        
```
为提高查找效率,将链表转换为ArrayList集合进行处理

还有一种方式是找中位,反转中位后的链表,在进行链表的合并(见链表反转题目)

---
### NC9 二叉树中是否存在节点和为指定值的路径
起始就是深度优先遍历二叉树算法

深度优先遍历二叉树部分
```java
  void dfs(TreeNode root,int sum,int tar){
        if(root==null|| flag == true){
            return;
        }
        tar += root.val;
        if(root.left==null&&root.right==null){
            if(sum == tar){
                flag  = true;
            }
        }else{
            dfs(root.left,sum,tar);
            dfs(root.right,sum,tar);
        }
    }
```
还有一种不使用递归的深度优先遍历二叉树的方式:
```java

```
#### 深度优先遍历算法精析:

---
### 判断链表是否有环
采用快慢指针法:
        一个指针走两步 一个指针走一步，如果快指针直接到了null 说明没有环， 如果有环的话 总有一次结果会让快指针和慢指针相等。
```java
 public boolean hasCycle(ListNode head) {
        ListNode p = head;
        ListNode q = head;
        while(p!=null && p.next!=null){
            p = p.next.next;
            q = q.next;
            if(p==q){
                return true;
            }
 
        }
        return false;
    }
```
### 冒泡排序 

```math
O(n2^)
```
```java
  private static void bubbleSort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
      for (int j = 0; j < arr.length - i - 1; j++) {
        if (arr[j] > arr[j + 1]) {
          int temp = arr[j];
          arr[j] = arr[j + 1];
          arr[j + 1] = temp;
        }
      }
    }
  }
```
### 快速排序
```math
O(nlogn)
```
```java
private  void quickSort(int[] arr, int head, int end) {
    if (head >= end) {
      return;
    }
    int low = head;//头指针
    int high = end;//尾指针
    int pivot = arr[low]; //基准
    while (low < high) {//首尾不能交叉
    //尾指针找比基准小的元素
      while (low < high && arr[high] > pivot) {
        high--;
      }
      // 头指针找比基准大的元素
      while (low < high && arr[low] <= pivot) {
      
        low++;
      }
      //找到了相应元素 交换
      swap(arr, low, high);
    }
    //没有找到相应元素
    swap(arr, low, head);
    quickSort(arr, head, low - 1);
    quickSort(arr, low + 1, end);
  }
  //交换元素的方法
    private  void swap(int[] arr, int i, int j) {
    int temp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;
  }
```
### 二分查找
```math
O(nlogn)
```
```java
  private static int findByHalf(int[] demo, int searchNum) {
    int start = 0;
    int end = demo.length - 1;
    while (start <= end) {
      int mid = (start + end) / 2;
      if (demo[mid] == searchNum) {
        return mid;
      } else if (demo[mid] > searchNum) {
        end = mid - 1;
      } else {
        start = mid + 1;
      }
    }
    return -1;
  }
```
### 跳台阶
动态规划:

    公式:
    dept[i] = dept[i-1]+dept[i-2];
    跳到第i层的组合数= 跳到第i-1层的组合数+跳到i-2层的组合数
    因为每次只能跳1层或者两层
```java
public int climbStairs(int n) {
      if (n == 1) {
        return 1;
      }
      if (n == 2) {
        return 2;
      }
      int[] dept = new int[n + 1];
      dept[1] = 1;
      dept[2] = 2;
      for (int i = 3; i <= n; i++) {
        dept[i] = dept[i - 1] + dept[i - 2];
      }
      return dept[n];
    }
```
### 树的高度(递归)
```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    return Math.max(maxDepth(root.left), maxDepth(root.right)) + 1;//+1 表示加上根节点
}
```
### 平衡树(递归)
---
1. 何为平衡二叉树<br>
    平衡二叉树或者是一棵空树，或者是具有以下性质的二叉排序树：<br>
    (1)它的左子树和右子树的高度之差绝对值不超过1；<br>
    (2)它的左子树和右子树都是平衡二叉树。
2. 实现对平衡二叉树的判定
```java
private boolean result = true;

public boolean isBalanced(TreeNode root) {
    maxDepth(root);
    return result;
}

public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    int l = maxDepth(root.left);
    int r = maxDepth(root.right);
    if (Math.abs(l - r) > 1) result = false;
    return 1 + Math.max(l, r);
}
```
### 两节点的最长路径
```java
private int max = 0;
public int deamaterOfTree(TreeNode node){
    depth(root);//写成方法,实现递归
    return max;
}
private int depth(TreeNode root) {
    if(root==null) return 0; //空树 回0
    int leftDepth = depth(root.left);//获得左子树高度
    int rightDepth = depth(root.right); //获得右子树高度
    max = Math.max(max,leftDepth+rightDepth); //贪心求得子树字长路径
    return Math.max(leftDepth,rightDepth)+1;
}
```
### 翻转树
对二叉树进行翻转 (左->右/右->左)<br>
由此联想到temp充当中间变量,交换两数的值
```java
swap(int a ,int b){
    int temp = a;
    a = b ;
    b = temp;
}
//如果实现左右树交换,即交换父亲节点的左右指针
swapTree(TreeNode root){
    TreeNode temp = root.left;
    root.left = root,right;
    root.right = temp;
}
//综上所述:可得出代码:
public TreeNode invertTree(TreeNode root) {
    if (root == null) return null;
    TreeNode left = root.left; 
    root.left = invertTree(root.right);
    root.right = invertTree(left);
    return root;
}
```
### 归并两颗二叉树
```
Input:
       Tree 1                     Tree 2
          1                         2
         / \                       / \
        3   2                     1   3
       /                           \   \
      5                             4   7

Output:
         3
        / \
       4   5
      / \   \
     5   4   7
```
```java
public TreeNode mergeTrees(TreeNode t1, TreeNode t2) {
    if (t1 == null && t2 == null) return null;
    if (t1 == null) return t2;
    if (t2 == null) return t1;
    //归并根节点
    TreeNode root = new TreeNode(t1.val+t2.val);
    //遍历归左右
    root.left = mergeTrees(t1.left, t2.left);
    root.right = mergeTrees(t1.right, t2.right);
    return root;
}
```
### 判断路径和是否等于一个数
根/左/右 先序遍历 每次减去节点值
```java
public boolean hasPathSum(TreeNode root, int sum) {
    if (root == null) return false;
    if (root.left == null && root.right == null && root.val == sum) return true;
    return hasPathSum(root.left, sum - root.val) || hasPathSum(root.right, sum - root.val);
}
```






# 手写线程安全的单例模式
```java
public class Singlton{
    //
    private Singlton(){
        
    }
    //volatile 保证内存可见性 防止指令重排
    private static volatile Singlton INSTANCE;
    public static Singlton getInstance(){
    //懒加载并且保证线程安全 保证了效率
        if(INSTANCE == null){
            synchrozed(Singlton.class){
                if(INSTANCE == null){
                    INSTANCE = new Singlton();
                }
            }
        }
        return INSTANCE;
    }
}
```


### 阿里笔试 : 给定任务数 任务的开始时间 任务的持续时间 求最大并发量
```java
	public static void main(String[] args) {
		Scanner scanner = new Scanner(System.in);
		int n = scanner.nextInt();
		int[][] a = new int[n][2];
		for (int i = 0; i < n; ++i) {
			a[i][0] = scanner.nextInt();
			a[i][1] = scanner.nextInt();
		}
		int max = 0;
		int thread = 0;
		//从1开始扫描 有并发则并发数++
		for(int i = 1;i<= searchForMaxEndTime(a);++i){
			int corrent = 0;
			for(int j = 0;j<n;++j){

				if (i>= a[j][0]&&i<=a[j][0]+a[j][1]) {
					thread = corrent++;
				}
				
			}
			max = Math.max(thread, max);
		}
		if (max>=1) {
			max++;
		}
		System.out.println(max);
	}
	//查找最大结束时间
	private static int searchForMaxEndTime(int [][] temp){
		int max = temp[0][0]+temp[0][1];
		for(int i = 1;i<temp.length;++i){
			max = Math.max(temp[i-1][0]+temp[i-1][1], temp[i][0]+temp[i][1]);
		}
		return max;
	}
```
### 阿里笔试: 数组的乘积能不能被另一个数组的乘积整除
```java
public static void main(String[] args) {
  Scanner scanner = new Scanner(System.in);
  List<Integer> palyer = new ArrayList<>();
  List<Integer> boss = new ArrayList<>();
  Random random = new Random();

  for (int i = 0; i < 5; ++i) {
   palyer.add(scanner.nextInt());
  }

  for (int i = 0; i < 6; ++i) {
   boss.add(scanner.nextInt());

  }
  long playerSum = 1l;
  long bossSum = 1l;
  for (int i = 0; i < palyer.size(); ++i) {
   playerSum *= palyer.get(i);
  }
  for (int i = 0; i < boss.size(); ++i) {
   bossSum *= boss.get(i);
  }
  if (playerSum != 0 && bossSum % playerSum == 0) {
   System.out.println("yes");
  } else {
   System.out.println("no");
  }
 }
```
 ### 阿里笔试: 求和不小于0的最大子序列长度
 快排+条件判断
 ```java
 public static void main(String[] args) {
		int[] demo = new int[] { -2, -435, -12, -432, 43, 4 };
		int end = demo.length - 1;
		int sum = 0;
		System.out.println(Arrays.toString(demo));
		quickSort(demo, 0, end);
		System.out.println(Arrays.toString(demo));
		if (demo[end] < 0) {
			System.out.println("0");
			return;
		}
		int res = 0;
		for (int i = end; i >= 0; --i) {
			sum += demo[i];
			if (sum >= 0) {
				res++;
			}
			
		}
		System.out.println(res);
	}
	// 快排算法

	private static void quickSort(int[] demo, int head, int end) {
		// TODO Auto-generated method stub
		if (head >= end)
			return;
		int low = head;// 头指针
		int high = end;// 尾指针
		int pivot = demo[low]; // 基准
		while (low < high) {
			while (low < high && demo[high] > pivot) {
				high--;
			}
			while (low < high && demo[low] <= pivot) {
				low++;
			}
			// 找到交换
			swap(demo, low, high);
		}
		// 没找到交换
		swap(demo, low, head);
		quickSort(demo, head, low - 1);
		quickSort(demo, low + 1, end);
	}

	private static void swap(int[] demo, int low, int high) {
		int temp = demo[low];
		demo[low] = demo[high];
		demo[high] = temp;
	}
 ```