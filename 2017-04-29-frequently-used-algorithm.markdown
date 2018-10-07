---
layout:     post
title:      "最常用的一些算法"
subtitle:   "整理一下最常用的算法：二叉树的遍历(递归，非递归)， 快速排序，归并排序，二分查找， 快速幂"
date:       2017-04-29
author:     "ChenWenKe"
header-img: "img/post-bg-cattle.jpg"
tags:
    - 算法
    - C++
---

主要整理一些最最常用的，确实需要手撸的，即使是在迷糊状态下，仍需条件反射下，精准，快速的写出的一些算法。 以供自己和小伙伴不断熟练，提高撸速。 

#### 二叉树遍历

- 二叉树遍历的递归写法：

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <stack>
using namespace std;

struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(int x):val(x), left(NULL), right(NULL) {}
};

// 先序遍历
void preOrder(TreeNode* root)
{
    if(root == NULL)
        return;
    cout << root->val << " ";
    preOrder(root->left);
    preOrder(root->right);
}

// 中序遍历
void inOrder(TreeNode* root)
{
    if(root == NULL)
        return;
    inOrder(root->left);
    cout << root->val << " ";
    inOrder(root->right);
}

// 后序遍历
void lastOrder(TreeNode* root)
{
    if(root == NULL)
        return;
    lastOrder(root->left);
    lastOrder(root->right);
    cout << root->val << " ";
}

```
<br/>

- 非递归写法：
- 先序遍历
[题目链接](https://leetcode.com/problems/binary-tree-preorder-traversal/#/description)

```cpp
vector<int> preorderTraversal(TreeNode* root) {
        vector<int> res;
        stack<TreeNode*> temp;
        if(root == NULL)
            return res;
        temp.push(root);
        while(!temp.empty())
        {
            TreeNode* pos = temp.top();
            temp.pop();
            res.push_back(pos->val);
            if(pos->right != NULL)
                temp.push(pos->right);
            if(pos->left != NULL)
                temp.push(pos->left);
        }
        return res;
}

```
<br/>
- 中序遍历

[题目链接](https://leetcode.com/problems/binary-tree-inorder-traversal/#/description)

```cpp
// 只有当左子树已经访问完后，才能访问根节点
/*
    对于任一节点 P
    1)  若其左孩子不为空，则将 P 入栈并将 P 的左孩子置为当前 P ,
        然后对当前 P 再进行相同的处理；
    2)  若其左孩子为空，则取栈顶元素并进行出栈操作，访问该栈顶节点，
        然后将当前 P 置为栈顶节点的右孩子；
    3)  直到 P 为 NULL 并且栈为空则遍历结束；
*/
vector<int> inOrderTraversal(TreeNode* root){
    vector<int> res;
    stack<TreeNode*> temp;
    if(root == NULL)
        return res;
    TreeNode* pos = root;
    while(!temp.empty() || pos != NULL)
    {
        if(pos != NULL){
            // push 左子树入栈
            temp.push(pos);
            pos = pos->left;
        }
        else
        {
            pos = temp.top();
            temp.pop();
            res.push_back(pos->val);
            pos = pos->right;
        }
    }
    return res;
}

```
<br/>
- 后序遍历

[题目链接](https://leetcode.com/problems/binary-tree-postorder-traversal/#/description)

```cpp
// 先入栈根，然后是右子树，最后左子树
// 要求最后访问根节点， 即访问该根节点时必须访问完左子树和右子树，
// 我们只需要保证访问某一节点时，该节点的右子树已经被访问， 否则需要
// 将该节点重新压入栈。
/*
对于任一结点 P, 将其入栈， 然后沿其左子树一直往下搜索， 直到搜索到没有左孩子的结点，
此时该结点出现在栈顶，但是此时不能将其出栈并访问，因为其右孩子还没有被访问。
所以接下来按照相同的规则对其右子树进行相同的处理，当访问完其右孩子时，该结点又出现在
栈顶，此时可以将其出栈并访问。 这样就保证了正确的访问顺序。 可以看出，在这个过程中，
每个结点都两次出现在栈顶，并且只有第二次出现在栈顶时，才能访问它。
*/
 vector<int> postorderTraversal(TreeNode* root) {
        vector<int> res;
        stack<TreeNode*> tmp;
        if(root == NULL)
            return res;
        TreeNode* pos = root;   // 记录正在访问的结点
        TreeNode* pre = NULL;   // 记录前一个访问的结点
        do
        {
            while(pos != NULL)      // 把左子树结点都压进栈
            {
                tmp.push(pos);
                pos = pos->left;
            }
            pre = NULL;             // 左结点压完后，初始化前一个访问结点为 NULL
            while(!tmp.empty())
            {
                pos = tmp.top();
                tmp.pop();
                if(pos->right == pre)       // 右孩子已经被访问
                {
                    res.push_back(pos->val);    // 右孩子已经被访问，可以访问该结点
                    pre = pos;      // 记录刚刚访问的结点
                }
                else
                {
                    tmp.push(pos);  // 第一次访问该结点，再次放入栈中
                    pos = pos->right;
                    break;
                }
            }
        } while(!tmp.empty());

        return res;
    }
```
<br/>
- 层次遍历

[题目链接](https://www.nowcoder.com/practice/7fe2212963db4790b57431d9ed259701?tpId=13&tqId=11175&tPage=2&rp=1&ru=%2Fta%2Fcoding-interviews&qru=%2Fta%2Fcoding-interviews%2Fquestion-ranking)

```cpp
/*
struct TreeNode {
	int val;
	struct TreeNode *left;
	struct TreeNode *right;
	TreeNode(int x) :
			val(x), left(NULL), right(NULL) {
	}
};*/
class Solution {
public:
    vector<int> PrintFromTopToBottom(TreeNode* root) {
		vector<int> res; 
        queue<TreeNode*> tmp; 
        if(root == NULL)
            return res; 
        TreeNode* pos = root; 
        tmp.push(pos); 
        while(!tmp.empty())
       	{
            pos = tmp.front(); 
            tmp.pop();
            res.push_back(pos->val); 
            if(pos->left != NULL)
                tmp.push(pos->left); 
            if(pos->right != NULL)
                tmp.push(pos->right); 
        }
        return res; 
    }
};
```
<br/>

#### 排序

- 插入排序

```cpp
void InsertSort(vector<int> arr, int n)
{
	for(int j = 1; j < n; j++)
	{
		int key = arr[j]; 
		// Insert arr[j] into the sorted sequece arr[1..j-1]
		i = j - 1; 
		while(i >= 0 && arr[i] > key)	// search the right pos 
		{
			arr[i+1] = arr[i]; 		// move elements 
			i = i - 1; 			
		}
		arr[i+1] = key; 
	}
}
```
<br/>

- 归并排序

```cpp
void merge_sort(int* A, int left, int right, int * T)   // right 为尾后元素
{
	if(right - left > 1)
    {
        int mid = left + (right - left) / 2;    // 划分
        int pos1 = left, pos2 = mid, pos = left;
        merge_sort(A, left, mid, T);     // 递归求解
        merge_sort(A, mid, right, T);     // 递归求解
        while(pos1 < mid || pos2 < right)
        {
            if(pos2 >= right || (pos1 < mid && A[pos1] <= A[pos2]))   // 从左半数组复制到临时空间
                T[pos++] = A[pos1++];
            else
                T[pos++] = A[pos2++];                    // 从右半数组复制到临时空间
        }
        for(pos = left; pos < right; pos++) A[pos] = T[pos];         // 从辅助空间复制回 A 数组。
    }
}

```
<br/>

- 快速排序

```cpp
// quick sort

// 划分函数
int Partition(int data[], int length, int start, int end);

void quick_sort(int data[], int length, int start, int end)
{
    // end 是最后一个元素的下标
    if(start == end)
        return;
    int index = Partition(data, length, start, end);
    if(index > start)
        quick_sort(data, length, start, index-1);
    if(index < end)
        quick_sort(data, length, index+1, end);
}

int Swap(int& a, int& b);    // 交换函数
int Partition(int data[], int length, int start, int end)   // 划分函数
{
    if(data == NULL || length <= 0 || start < 0 || end >= length)
        throw runtime_error("Invalid Parameters");
    // int index = RandomInRange(start, end);       // 随机选取划分因子
    // swap(data[index], data[end]);
    int small = start - 1;
    for(int index = start; index < end; index++)
    {
        if(data[index] < data[end])
        {
            small++;
            if(small != index)
                Swap(data[index], data[small]);
        }
    }
    small++;
    Swap(data[small], data[end]);
    return small;
}

int Swap(int& a, int& b)    // 交换函数
{
    int tmp = a;
    a = b;
    b = tmp;
}

```
<br/>

- 计数排序

```cpp
void counting_sort(int data[], int begin, int end, int length)
{
    if(data == NULL || length <= 0 || begin < 0 || end >= length)
        throw runtime_error("Invalid Parameters");
    vector<int> count(100, 0);      // 假设排序的数字是 0 ~ 99 之间的整数 
    for(int i = begin; i <= end; i++)
        count[data[i]]++;
    for(int i = 1; i < count.size(); i++)
        count[i] += count[i-1];
    vector<int> T(length, 0);
    for(int i = begin; i <= end; i++)
        T[i] = data[i];
    for(int i = end; i >= begin; i--)   // 倒着扫描，保持排序的稳定性
    {
        data[count[T[i]] - 1] = T[i];   // 数组是从 0 开始的
        count[T[i]]--;
    }
}

```
<br/>

#### 求逆序数
使用归并排序求逆序数

```cpp
int cnt = 0; 
void merge_sort(int data[], int start, int end, int T[])
{
	if(end - start > 1)
	{
		int mid = start + (end - start)/2;	// 划分 
		int pos1 = start, pos2 = mid, pos = start; 
		merge_sort(data[], begin, mid, T); 
		merge_sort(data[], mid, end, T); 
		while(pos1 < mid || pos2 < end)
		{	
			if(pos2 >= end || (pos1 < mid && data[pos1] < data[pos2]))
				T[pos++] = data[pos1++]; 
			else
			{
				T[pos++] = data[pos2++]; 
				cnt += mid - pos1;		// 统计逆序数 
			}
		}
		for(pos = start; pos < end; pos++)
			data[pos] = T[pos]; 
	}
}
```
<br/>
#### 线性时间选择

基于快速排序的ParTition函数实现线性选择，如查找中位数，第 k 大的数，第 k 小的数，算法时间复杂度的数学期望是O(n)。 

[题目链接](https://leetcode.com/problems/kth-largest-element-in-an-array/#/description)

```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        int left = 0, right = nums.size() - 1; 
        if(k > nums.size())
            throw runtime_error("Invalid Parameter"); 
        
        int target = nums.size() - k; 
        
        while(true)
        {
            int pos = partition(nums, left, right); 
            if(pos == target)
                return nums[pos]; 
            else if(pos > target)
                right = pos - 1; 
            else 
                left = pos + 1; 
        }
    }
    
private:
    int partition(vector<int>& nums, int left, int right)
    {
        int mid = nums[right]; 
        int smaller = left - 1; 
        for(int pos = left; pos < right; pos++)
        {
            if(nums[pos] < mid)
            {
                smaller++; 
                if(nums[pos] != nums[smaller])
                {
                    swap(nums[pos], nums[smaller]); 
                }
            }
        }
        smaller++; 
        swap(nums[smaller], nums[right]); 
        
        return smaller; 
    }
};
```

<br/>

#### 二分查找

- 二分查找的迭代实现

```cpp
int bsearch(int data[], int start, int end, int val)	// end 为未后元素
{
	int mid; 
	while(start < end)
	{
		mid = start + (end - start)/2; 
		if(data[mid] == val)
			return mid; 
		else if(data[mid] > val)
			end = mid; 
		else 
			start = mid + 1; 
	}
	return -1;	// 没找到
}
```
<br/>

- 二分下界查找

```cpp
int lower_bound(int data[], int start, int end, int val)
{
	int mid; 
	while(start < end)
	{
		mid = start + (end - start)/2; 
		if(data[mid] >= val)
			end = mid; 
		else 
			start = mid + 1; 
	}
	return start; 
}

```

- 二分上界查找

```cpp
int upper_bound(int data[], int start, int end, int val)
{
	int mid; 
	while(start < end)
	{
		mid = start + (end - start)/2; 
		if(data[mid <= val])
			start = mid+1; 
		else
			end = mid; 
	}
	return start; 
}
```
<br/>

#### 快速幂

- 非递归写法

```cpp
int fastPow(int base, int num)
{
	if(num < 0)
		throw runtime_error("Invalid Input\n"); 
	else if(num == 0)
		return 1; 
	int res = 1; 
	while(num)
	{
		if(num & 1)
			res *= base; 
		base *= base; 
		num >>= 1; 
	}
	return res; 
}
```
<br/>

- 递归写法

```cpp
int fastPow(int base, int num)
{
	if(num < 0)
		throw runtime_error("Invalid Input\n"); 
	else if(num == 0)
		return 1; 

	if(num & 1)
		return base * fastPow(base * base, num >> 1); 
	else
		return fastPow(base * base, num >> 1); 
}
```
<br/>

> Talk is cheap, show me the code.

<br/>


#### 稍详细些的笔记

- [二叉树遍历](https://github.com/CaoTanXiaoKe/CodePool/blob/master/NormalAlgorithm/BinaryTreeTraversal.md)
- [排序算法比较](https://github.com/CaoTanXiaoKe/CodePool/blob/master/NormalAlgorithm/SortAlgrithomNote.md)
- [归并排序](https://github.com/CaoTanXiaoKe/CodePool/blob/master/NormalAlgorithm/MergeSort.md)
- [二分查找](https://github.com/CaoTanXiaoKe/CodePool/blob/master/NormalAlgorithm/BinarySeach.md)
- [快速幂](https://github.com/CaoTanXiaoKe/CodePool/blob/master/NormalAlgorithm/FastPow.md)