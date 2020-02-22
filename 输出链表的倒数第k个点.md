## 题目描述

输入一个链表，输出该链表中倒数第k个结点。

## 思路

**用vector做中间存储**

头指针空，k小于0直接return null；

链表长vec.size()小于k，return null

没问题的话return vec[vec.size()-k];

```c++
/*
struct ListNode {
	int val;
	struct ListNode *next;
	ListNode(int x) :
			val(x), next(NULL) {
	}
};*/
class Solution {
public:
    ListNode* FindKthToTail(ListNode* pListHead, unsigned int k) {
        //反转链表，然后正序输出
        vector<ListNode*> vec;
        if(pListHead==nullptr||k<=0) return NULL;
        ListNode *p=pListHead;
        while(p!=nullptr)
        {
            vec.push_back(p);
            p=p->next;
        }
        if(vec.size()<k) return NULL;
        
        reverse(vec.begin(),vec.end());
        return vec[k-1];
      //也能直接输出
      //return vec[vec.size()-k];
    }
};
```

