# 包含min函数的栈

## 题目描述

定义栈的数据结构，请在该类型中实现一个能够得到栈中所含最小元素的min函数（时间复杂度应为O（1））。

注意：保证测试中不会当栈为空的时候，对栈调用pop()或者min()或者top()方法。

## 思路

定义两个栈dataStack，minStack

dataStack无论何时，只要push就入栈，minStakc只有当minStack空或者node<minStack.top()才给入栈



```c++
class Solution {
public:
    stack<int> minStack;
    stack<int> dataStack;
    void push(int value) {
        dataStack.push(value);
        if(minStack.empty()||value<minStack.top())
        {
            minStack.push(value);
        }
        else
        {
            minStack.push(minStack.top());
        }
    }
    void pop() {
        minStack.pop();
    }
    int top() {
        return minStack.top();
    }
    int min() {
        return minStack.top();
    }
};
```

