---
layout: post
title: "二叉树的构建与遍历"
description: "java 实现二叉树的构建以及前、中、后序和层序遍历（递归和循环）"
date: 2017-06-30
tags: [Java,算法]
comments: true
share: true
---

```java
import java.util.HashMap;
import java.util.LinkedList;
import java.util.Map;
import java.util.Queue;
import java.util.Stack;

/**
 * @author zhangyong
 * @create 2017-06-24
 */
public class BinaryTree {

    private Node root;

    /**
     * 构建二叉树
     *
     * @param node
     * @param data
     */
    public void build(Node node, int data) {
        if (root == null)
            root = new Node(data);
        else {
            if (data < node.data) {
                if (node.left == null) {
                    node.left = new Node(data);
                } else {
                    build(node.left, data);
                }
            } else {
                if (node.right == null) {
                    node.right = new Node(data);
                } else {
                    build(node.right, data);
                }
            }
        }
    }

    /**
     * 访问节点
     *
     * @param node
     */
    public void visit(Node node) {
        System.out.print(node.data + " ");
    }

    /**
     * 前序遍历
     *
     * @param node
     */
    public void preOrderRec(Node node) {
        if (node != null) {
            visit(node);
            preOrderRec(node.left);
            preOrderRec(node.right);
        }
    }

    /**
     * @param node 树的根节点
     * 利用栈模拟递归过程实现循环先序遍历二叉树
     * 这种方式具备扩展性，它模拟递归的过程，将左子树点不断的压入栈，直到null，然后处理栈顶节点的右子树
     */
    public void preOrderStack(Node node){
        if(node==null)return;
        Stack<Node> s= new Stack<>();
        while(node!=null||!s.isEmpty()){
            while(node!=null){
                visit(node);
                s.push(node);//先访问再入栈
                node=node.left;
            }
            node=s.pop();
            node=node.right;//如果是null，出栈并处理右子树
        }
    }

    /**
     * 中序遍历
     *
     * @param node
     */
    public void inOrderRec(Node node) {
        if (node != null) {
            inOrderRec(node.left);
            visit(node);
            inOrderRec(node.right);
        }
    }

    /**
     *
     * @param node 树根节点
     * 利用栈模拟递归过程实现循环中序遍历二叉树
     * 思想和上面的preOrderStack_2相同，只是访问的时间是在左子树都处理完直到null的时候出栈并访问。
     */
    public void inOrderStack(Node node){
        if(node==null)return;
        Stack<Node> s=new Stack<Node>();
        while(node!=null||!s.isEmpty()){
            while(node!=null){
                s.push(node);//先访问再入栈
                node=node.left;
            }
            node=s.pop();
            visit(node);
            node=node.right;//如果是null，出栈并处理右子树
        }
    }


    /**
     * 后序遍历
     *
     * @param node
     */
    public void postOrderRec(Node node) {
        if (node != null) {
            postOrderRec(node.left);
            postOrderRec(node.right);
            visit(node);
        }
    }

    /**
     *
     * @param root 树根节点
     * 后序遍历不同于先序和中序，它是要先处理完左右子树，然后再处理根(回溯)，所以需要一个记录哪些节点已经被访问的结构(可以在树结构里面加一个标记)，这里可以用map实现
     */
    public void postOrderStack(Node root){
        if(root==null)return;
        Stack<Node> s=new Stack<Node>();
        Map<Node,Boolean> map=new HashMap<Node,Boolean>();
        s.push(root);
        while(!s.isEmpty()){
            Node temp=s.peek();
            if(temp.left!=null&&!map.containsKey(temp.left)){
                temp=temp.left;
                while(temp!=null){
                    if(map.containsKey(temp))break;
                    else s.push(temp);
                    temp=temp.left;
                }
                continue;
            }
            if(temp.right!=null&&!map.containsKey(temp.right)){
                s.push(temp.right);
                continue;
            }
            Node t=s.pop();
            map.put(t,true);
            visit(t);
        }
    }

    /**
     * 层序遍历
     * 层序遍历二叉树，用队列实现，先将根节点如队列，只要队列不为空，然后出队列，并访问，接着将访问节点的左右子树依次如队列
     *
     * @param node
     */
    public void levelTravel(Node node) {
        if (node == null)
            return;
        Queue<Node> q = new LinkedList<>();
        q.add(node);
        while (!q.isEmpty()) {
            Node temp = q.poll();
            visit(temp);
            if (temp.left != null) q.add(temp.left);
            if (temp.right != null) q.add(temp.right);
        }
    }

    /**
     * 获取根节点
     *
     * @return
     */
    public Node getRoot() {
        return root;
    }

    /**
     * 二叉树节点类
     */
    private class Node {
        private int data;
        private Node left;
        private Node right;

        public Node(int data) {
            this.data = data;
        }
    }

}

```
