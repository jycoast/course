### 使用栈解决反转链表的问题

``` java
package org.springblade.modules.user;

import java.util.ArrayList;
import java.util.List;

class TreeNode {
	int val;
	TreeNode left;
	TreeNode right;

	TreeNode(int val) {
		this.val = val;
		this.left = null;
		this.right = null;
	}
}
class ListNode {
	int val;
	ListNode next;
	ListNode(int x) { val = x; }
}

public class Solution {
	public int maxDepth(TreeNode root) {
		// 基本情况：如果节点为空，返回深度 0
		if (root == null) {
			return 0;
		}
		// 递归计算左子树和右子树的深度
		int leftDepth = maxDepth(root.left);
		int rightDepth = maxDepth(root.right);
		// 返回左右子树深度的最大值加上当前节点的高度（1）
		return Math.max(leftDepth, rightDepth) + 1;
	}

	public void preOrder(TreeNode root) {
		List<Integer> list = new ArrayList<>();
		if (root == null)
			return;
		// 访问优先级：根节点 -> 左子树 -> 右子树
		list.add(root.val);
		System.out.println("前序遍历排序后的list为："+list);
		preOrder(root.left);
		preOrder(root.right);
	}
	public void inOrder(TreeNode root) {
		List<Integer> list = new ArrayList<>();
		if (root == null)
			return;
		// 访问优先级：左子树 -> 根节点 -> 右子树
		inOrder(root.left);
		list.add(root.val);
		System.out.println("中序遍历排序后的list为："+list);
		inOrder(root.right);
	}
	public void postOrder(TreeNode root) {
		List<Integer> list = new ArrayList<>();
		if (root == null)
			return;
		// 访问优先级：左子树 -> 右子树 -> 根节点
		postOrder(root.left);
		postOrder(root.right);
		list.add(root.val);
		System.out.println("后序遍历排序后的list为："+list);
	}
	public ListNode reverseList(ListNode head) {
		// 递归终止条件
		if (head == null || head.next == null) {
			return head;
		}
		ListNode newHead = reverseList(head.next);

		// 关键操作：将当前节点的下一个节点的 next 指向自己
		head.next.next = head; // 建立反向连接
		head.next = null;      // 断开原正向连接
		System.out.println("反转后的节点为"+head.val);
		return newHead;        // 返回反转后的新头节点
	}

	public static void main(String[] args) {
		// 构建一个简单的二叉树
		//       1
		//      / \
		//     2   3
		//    / \
		//   4   5
		TreeNode root = new TreeNode(1);
		root.left = new TreeNode(2);
		root.right = new TreeNode(3);
		root.left.left = new TreeNode(4);
		root.left.right = new TreeNode(5);
//		root.left.left.left = new TreeNode(6);

		Solution solution = new Solution();
//		solution.preOrder(root);   // 前序遍历
//		solution.inOrder(root);    // 中序遍历
//		solution.postOrder(root);  // 后序遍历

		ListNode listNode = new ListNode(1);
		listNode.next = new ListNode(2);
		listNode.next.next = new ListNode(3);
		listNode.next.next.next = new ListNode(4);
		ListNode node = solution.reverseList(listNode); //翻转链表

//		int result = solution.maxDepth(root);  // 计算出深度
//		System.out.println("Maximum depth of the binary tree is: " + result);
	}
}


```