---
title: LeetCode解题(Java)
date: 2021-10-23 21:06:42.42
updated: 2021-11-02 14:00:09.144
url: /archives/leetcodejava
categories: 
tags: Java | 算法
---

### 41 缺失的第一个正数
---
>给你一个未排序的整数数组 nums ，请你找出其中没有出现的最小的正整数。
请你实现时间复杂度为 O(n) 并且只使用常数级别额外空间的解决方案。
	示例 1：
			输入：nums = [1,2,0]
			输出：3
	示例 2：
			输入：nums = [3,4,-1,1]
			输出：2
	示例 3：
			输入：nums = [7,8,9,11,12]
			输出：1
 ---
```java
    public int firstMissingPositive(int[] nums) {
	// 排序  为了以后更好的操作数组
        Arrays.sort(nums);
        int len = nums.length;
	// 由1开始查找元素	
        int result =  1 ;
	// 最大的数 只能为1 -- 数组的长度 遇到一个从1开始的递增数就把结果+1
	//因为元素已经排序完成 就是从小到大的顺序  中断则输出
        for(int i = 0;i<len;++i){
            if(nums[i]==result) result ++;
        }
              return result;
    }
```
### 237. 删除链表中的节点

>请编写一个函数，用于 删除单链表中某个特定节点 。在设计函数时需要注意，你无法访问链表的头节点 head ，只能直接访问 要被删除的节点 。
题目数据保证需要删除的节点 不是末尾节点 。 
![image.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic//image_1635832221378.png)
示例 1：
输入：head = [4,5,1,9], node = 5
输出：[4,1,9]
解释：指定链表中值为 5 的第二个节点，那么在调用了你的函数之后，该链表应变为 4 -> 1 -> 9
示例 2：
![image.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic//image_1635832234163.png)
输入：head = [4,5,1,9], node = 1
输出：[4,5,9]
解释：指定链表中值为 1 的第三个节点，那么在调用了你的函数之后，该链表应变为 4 -> 5 -> 9
示例 3：
输入：head = [1,2,3,4], node = 3
输出：[1,2,4]
示例 4：
输入：head = [0,1], node = 0
输出：[1]
示例 5：
输入：head = [-3,5,-99], node = -3
输出：[5,-99]
 

提示：

>链表中节点的数目范围是 [2, 1000]
-1000 <= Node.val <= 1000
链表中每个节点的值都是唯一的
需要删除的节点 node 是 链表中的一个有效节点 ，且 不是末尾节点
---
```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public void deleteNode(ListNode node) {
        ListNode nextNode = node.next;
        node.val = nextNode.val;
        node.next = nextNode.next;
    }
}
```
提交评测结果:
![image.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic//image_1635832335808.png)
解题思路:
1. 不给head头结点无法遍历链表,由此不可以循环找出node节点的前一个节点进行删除操作
2. 有其给的提示**不是末尾节点** 可以根据末尾节点的特性考虑问题.链表中各个链表的性质都是一样的,除了头节点没有前驱,末尾节点**没有后继**.
3. 由没有后继想到要在后继节点上做文章: 删除后继,将后继的value赋值给当前节点的value属性,看似要删除的是node,实则删除的是node.next 原因是因为我们无法找出node.pre节点,因此要使node充当pre节点的角色
4. 本题基础是链表删除特定节点,删除模板如下
```java
	// 核心代码
	preNode.next =  node.next;
	// node节点彻底脱离链表
	node.next = null;
```