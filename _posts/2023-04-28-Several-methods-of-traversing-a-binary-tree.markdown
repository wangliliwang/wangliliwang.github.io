---
layout: post
title:  "Several methods of traversing a binary tree"
date:   2023-04-28 12:33:49 +0800
categories: Computer-Science
tags: ["Algorithm", "Golang", "Binary Tree"]
toc: true
language: english
sidebar: true
author: Wang Li
---

Binary tree is a special kind of graph, so there are two kinds of traversal methods, depth-first and breadth-first. The detailed methods are as shown below:

![](/assets/image/20230428-traversal/methods.png)

for the following binary tree, the effects is as follows:

![](/assets/image/20230428-traversal/tree-example.png)

1. preorder: 1 -> 2 -> 4 -> 6 -> 7 -> 3 -> 5
2. inorder: 6 -> 4 -> 7 -> 2 -> 1 -> 3 -> 5
3. postorder: 6 -> 7 -> 4 -> 2 -> 5 -> 3 -> 1
4. level order: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7
5. zigzag level order: 1 -> 3 -> 2 -> 4 -> 5 -> 7 -> 6

The following is a detailed description of the algorithms.

Special instruments: the code part of this article is assisted by `Github Copilot`, and the translation part from Chinese to English is also assisted by `Github Copilot`.

## Definition

Binary tree is defined as follows:

```go
type Node struct {
	Left, Right *Node
	Val         int
} 
```

Binary tree is represented by root node `root`, the number of nodes is `N`, and the maximum height of the tree is `h`.

Other data structures are defined as follows:

```go
type Queue[T any] interface {
	Enqueue(value T)
	Dequeue() (T, bool)
	Peek() (T, bool)
	Len() int
}

type Stack[T any] interface {
	Push(value T)
	Pop() (T, bool)
	Peek() (T, bool)
	Len() int
}
```

## DFS

### recursive

Tree is defined recursively, so recursive method is the most natural and simplest.

1. Time complexity, because each node is visited once, so: `O(N)`.
2. Space complexity, because the recursive call stack is used, so: `O(h)`.

The implementation of preorder traversal is as follows:

```go
func PreorderRecursive(root *Node) []int {
    if root == nil {
        return nil
    }
    result := []int{root.Val}
    result = append(result, PreorderRecursive(root.Left)...)
    result = append(result, PreorderRecursive(root.Right)...)
    return result
}
```

### stack 

1. Time complexity, because each node is visited once, so: `O(N)`.
2. Space complexity, because the stack is used, so: `O(h)`.

The implementation of preorder traversal is as follows:

```go
func PreorderTraversal(root *Node) []int {
	if root == nil {
		return nil
	}
	result := make([]int, 0)
	stack := NewStack[*Node]()
	stack.Push(root)
	for stack.Len() > 0 {
		node, _ := stack.Pop()
		result = append(result, node.Val)
		if node.Right != nil {
			stack.Push(node.Right)
		}
		if node.Left != nil {
			stack.Push(node.Left)
		}
	}
	return result
}
```

### Morris

Morris traversal is a method that does not use stack or recursive call stack, so it is a constant space complexity method.

1. Time complexity, because each node is visited once, so: `O(N)`.
2. Space complexity: `O(1)`.

Special instruments: the implementation of Morris's postorder traversal is more complex, and interested readers can implement it by themselves.

The implementation of preorder traversal is as follows:

```go
func PreorderTraversal(root *Node) []int {
	if root == nil {
		return nil
	}
    result := make([]int, 0)
	cur := root
	for cur != nil {
		if cur.Left == nil {
            result = append(result, cur.Val)
			cur = cur.Right
		} else {
			pre := cur.Left
			for pre.Right != nil && pre.Right != cur {
				pre = pre.Right
			}
			if pre.Right == nil {
                result = append(result, cur.Val)
				pre.Right = cur
				cur = cur.Left
			} else {
				pre.Right = nil
				cur = cur.Right
			}
		}
	}
	return result
}
```

## BFS

### queue

1. Time complexity, because each node is visited once, so: `O(N)`.
2. Space complexity, because the queue is used, so: `O(N)`.

The implementation of level order traversal is as follows:

```go
func LevelOrderTraversal(root *Node) []int {
	var result []int
	if root == nil {
		return result
	}
	q := NewQueue[*Node]()
	q.Enqueue(root)
	for q.Len() > 0 {
		node, _ := q.Dequeue()
        result = append(result, node.Val)
		if node.Left != nil {
			q.Enqueue(node.Left)
		}
		if node.Right != nil {
			q.Enqueue(node.Right)
		}
	}
	return result
}
```
