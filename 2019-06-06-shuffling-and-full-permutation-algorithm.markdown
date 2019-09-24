---
layout:     post
title:      "洗牌算法与全排列"
subtitle:   "洗牌算法可以看做是每次等概论地从所有扑克牌的全排列中取出一个排列"
date:       2019-06-06
author:     "ChenWenKe"
tags:
	- 算法
mathjax: true
---

​        今天在**机器之心**公众号上看到[一道有趣的题目: 洗牌算法](https://mp.weixin.qq.com/s/LQP-hRicjtZL068NM365eA)，细想之后发现和全排列算法有很多关联，在此记录一下。

### 洗牌算法

​       洗牌算法顾名思义就是用来洗牌的算法，也即生成一个所有扑克牌的一个排列。 要求这个排列生成的概率和其他排列生成的概率是一样的。 换句话说，洗牌算法可以是每次等概论地从所有扑克牌的全排列中取出一个排列。

![Overhand3](/blog/img/Overhand3.jpg)

#### 算法思路(一)

​        思考这个问题可以先把54张扑克牌依次编号为 $$\left[1,  54\right]$$ ，然后假设有54个槽位，每个槽位放置一张扑克牌。

- 从54张扑克牌中随机选一张，放置到第一个槽位，每张扑克牌放置在第一个槽位的概率是  $$\frac{1}{54}$$ ；
- 从余下的53张牌中随机选一张，放置到第二个槽位，每张扑克牌放置在第二个槽位的概率是 `未放进第一个槽位的概率 * 放进了第二个槽位的概率` = （1  -  $$\frac{1}{54}$$）$$\times$$ $$\frac{1}{53}$$ = $$\frac{1}{54}$$ ；
- 同理，后续每个槽位，每张扑克牌放置进去的概率都是 $$\frac{1}{54}$$ ，算法正确性得证。

#### 算法思路(二)

​        从[1,54]这54个数字的全排列中随机取出一个。这个方法理论上可行，只是存放全排列占用内存太大，因为有个词叫**阶乘爆炸**。

#### 算法(一)实现

​        为了便于编程实现，我们把数组的最后一个位置认为是第一个槽位，倒数第二个位置为第二个槽位 … 。

```cpp
#include <iostream>
#include <cstdlib>
#include <vector>
using namespace std;

class Solution {
    public:
        vector<int> CardsShuffling() {
            // initialize random seed
            srand(time(NULL));

            const int cardsNum = 54;
            vector<int> res(54, 0);
            for(int i = 0; i < cardsNum; i++) {
                res[i] = i+1;
            }
            for(int slot = cardsNum-1; slot >= 0; slot--) {
                // generate secret number [0, slot]
                int pos = rand() % (slot + 1);  // 从余下的扑克牌中随机取一张

                swap(res[pos], res[slot]);  // 放置到第(54 - slot)个槽位中
            }

            return res;
        }
};

int main() {
    // test
    Solution solve;
    vector<int> res = solve.CardsShuffling();
    for(auto iter = res.begin(); iter != res.end(); iter++) {
        cout << *iter << "  ";
    }
    cout << endl;
}
```

### 全排列算法

​      全排列算法，即生成全部排列可能的算法。假设给定一个魔方，则这个魔方所处的所有状态组成一个全排列。假设给定一个字符串: "bcd"，则这个字符串的全排列为: `bcd`, `cbd`, `dcb`, `bdc`, `dbc`, `cdb`，共 $${3!} = 3 \times 2 \times 1= 6$$ 种情况。 为了编程描述方便，我们下面以字符串的全排列为例：

![cube](/blog/img/cube.png)

#### 算法思路

假设我们要生成字符串"abcd"的全排列，并且由上面已知字符串"bcd"的全排列。要得到"abcd"的全排列， 我们只需要依次把 a 与`bcd的全排列`中的字母逐个交换位置即可。

- 假定: "bcd"的排列是`bcd`, a 与`bcd`不替换得到排列 `abcd`, `a`与`b`替换得到排列`bacd`, … 最终得到排列为：`abcd`, `bacd`, `cbad`, `dbca`。
- 如果 "bcd"的排列是`cbd`, 同理最终得到排列为: `acbd`, `cabd`, `bcad`, `dcba`。
- 同理，依次得到"bcd"的排列为  `dcb`, `bdc`, `dbc`, `cdb`时的"abcd"的排列。 由上可知，替换位置之后，由每个"bcd"的排列，得到了4个"abcd"的排列，而且由于是a 与 "bcd"排列中的字母逐个替换位置，因此每个字母出现在每个位置的概率也是均等的。"abcd"的排列数为: $$ “bcd”的排列数 \times 4  = 3! \times 4 = 4!$$ 。算法正确性得证。

#### 算法实现

按照上述算法思路，直观实现如下：

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <set>
using namespace std;

class Solution {
public:
    vector<string> GeneratePermutations(string str){
        vector<string> res;
        // 当字符串为空时，返回空列表
        if (str.size() == 0)
            return res;
        // 当字符串长度为1时，全排列只有一个(它自身)。
        if (str.size() == 1) {
            res.push_back(str);
            return res;
        }

        // 得到除第一个字符外，其他字符的全排列。
        string subStr(str.begin()+1, str.end());
        vector<string>subPermutations = GeneratePermutations(subStr);

        // 用第一个字符，和后面字符的全排列中的字符，逐个替换位置。
        for(int i = 0; i < subPermutations.size(); i++) {
            string tmp = str[0] + subPermutations[i];
            res.push_back(tmp);
            for(int j = 1; j < tmp.size(); j++) {
                swap(tmp[0], tmp[j]);
                res.push_back(tmp);
                // 替换回位置，以便下个循环继续用第一个字符和其他字符替换位置。
                swap(tmp[0], tmp[j]);
            }
        }
        return res;
    }
};

int main() {
    // test
    Solution solve;
    string str = "abcd";
    vector<string> res = solve.GeneratePermutations(str);
    cout << "permutation number: " << res.size() << endl;
    set<string> uniquePermutations;
    for(int i = 0; i < res.size(); i++) {
        uniquePermutations.insert(res[i]);
    }
    cout << "unique permutations number: " << uniquePermutations.size() << endl;

    return 0;
}
```

​        上述程序虽然比较直观，但是其中有很多临时变量，在输入的字符串长度比较大时，容易栈溢出。 稍微优化下程序，节约内存。

​        **程序思路为**: 例如给定字符串：abcd, 分别生成以 a为开头，bcd的全排列；以b为开头，acd的全排列; 以c为开头，abd的全排列；以d为开头，abc的全排列。 依次递归生成 bcd, acd…的全排列。上述过程的递归树，是一颗以""为根，树深为4，路径上的字符各不相同(分别是"a", "b", "c", "d")的二叉树。生成全排列的过程，就是遍历这颗递归树的过程，每条路径即是一个排列。

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <set>
using namespace std;

class Solution {
public:
    vector<string> GeneratePermutations(string str){
        vector<string> res;
        // 当字符串为空时，返回空列表
        if (str.size() == 0)
            return res;
        // 当字符串长度为1时，全排列只有一个(它自身)。
        if (str.size() == 1) {
            res.push_back(str);
            return res;
        }

        // 生成 str 从 0 开始, 到 str.end()这些字符的全排列，
        // 并存放到 res 中返回。
        genPermatations(str, 0, res);

        return res;
    }

private:
    void genPermatations(string str, int pos, vector<string> &res) {
        // 到达最后一个字符(边界)，搜索树已经达到叶子结点，路径放入res返回。
        if (pos + 1 == str.size()) {
            res.push_back(str);
            return;
        }
        for(int i = pos; i < str.size(); i++) {
            swap(str[pos], str[i]);
            genPermatations(str, pos + 1, res);
            swap(str[pos], str[i]);
        }

        return;
    }
};

int main() {
    // test
    Solution solve;
    string str = "abcd";
    vector<string> res = solve.GeneratePermutations(str);
    cout << "permutation number: " << res.size() << endl;
    set<string> uniquePermutations;
    for(int i = 0; i < res.size(); i++) {
        uniquePermutations.insert(res[i]);
    }
    cout << "unique permutations number: " << uniquePermutations.size() << endl;

    return 0;
}
```

#### 有序全排列

​        现在对上面生成的全排列添加一个限制条件: 要求生成的全排列是按照字母序生成的。即: 生成的"abc"的全排列需要依次是：`abc`, `acb`, `bac`, `bca`, `cab`, `cba`。

##### 算法思路

​       仍按照遍历递归树的方式进行思考 ， 如果给定一个字符"a", 递归树只有一条路径`""->"a"`。如果给定两个字符"ab", 遍历它的递归树的顺序应该是依次以: "a", "b"结点开头，路径分别是: `""->"a"->"b"`, `""->"b"->"a"`。 如果给定三个字符："abc",  遍历它的递归树的顺序应该是依次以: "a", "b", "c"结点开头，然后有序的遍历后续结点。也即: 遍历"a"结点后，有序遍历由"bc"生成的递归树; 遍历"b"结点后，有序遍历由"ac"生成的递归树；遍历"c"结点后，有序遍历由"ab"生成的递归树。

​       当然，首先生成全排列，然后对全排列进行排序，也是一种方法，只是不够优雅。

##### 算法实现

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <set>
#include <algorithm>
using namespace std;

class Solution {
public:
    vector<string> GeneratePermutations(string str){
        vector<string> res;
        // 当字符串为空时，返回空列表
        if (str.size() == 0)
            return res;
        // 当字符串长度为1时，全排列只有一个(它自身)。
        if (str.size() == 1) {
            res.push_back(str);
            return res;
        }

        // 生成 str 从 0 开始, 到 str.end()这些字符的全排列，
        // 并存放到 res 中返回。
        genPermatations(str, 0, res);

        return res;
    }

private:
    void genPermatations(string str, int pos, vector<string> &res) {
        // 到达最后一个字符(边界)，搜索树已经达到叶子结点，路径放入res返回。
        if (pos + 1 == str.size()) {
            res.push_back(str);
            return;
        }

        // 对后面的字符进行排序，从而较小字母序的字符结点，先被访问。
        // 这样，遍历路径的顺序即为字符序从小到大。
        sort(str.begin() + pos, str.end());

        for(int i = pos; i < str.size(); i++) {
            swap(str[pos], str[i]);
            genPermatations(str, pos + 1, res);
            swap(str[pos], str[i]);
        }

        return;
    }
};

int main() {
    // test
    Solution solve;
    string str = "abcd";
    vector<string> res = solve.GeneratePermutations(str);
    cout << "permutation number: " << res.size() << endl;
    set<string> uniquePermutations;
    for(int i = 0; i < res.size(); i++) {
        uniquePermutations.insert(res[i]);
    }
    cout << "unique permutations number: " << uniquePermutations.size() << endl;
    for(int i = 0; i < res.size(); i++) {
        cout << res[i] << endl;
    }

    return 0;
}
```

#### 去重全排列

​        同样的，对生成的全排列添加一个限制条件，生成的全排列里，如果有重复排列只保留一份。例如给定字符串："aab", 其去重全排列为： `aab`, `aba`, `baa`； 如果给定字符串:"aabb", 其去重全排列为: `aabb`, `abab`, `abba`, `baab`, `baba`, `bbaa`，共 $$ \frac {^{4} P_{4}} {\binom{2}{2} \times \binom{2}{2}}  = \frac {4!} {2! \times 2!} = 6$$ 种情况。

##### 算法思路

​        仍旧按照遍历递归树的思路进行思考，例如给定"aabb"， 递归树第二层结点分别是: "a", "b"。也即遍历"a"结点后，遍历由"abb"生成的递归树；遍历"b"结点后，遍历由"aab"生成的递归树。 "abb"生成的递归树有三条不同的路径:`"a"->"b"->"b"`,`"b"->"a"->"b"`, `"b"->"b"->"a"`，"aab"生成的递归树叶有三条不同的路径：  `"a"->"a"->"b"`,`"a"->"b"->"a"`, `"b"->"a"->"a"`。 因此遍历"aabb"递归树得到的路径(排列)分别是: `"a"->"a"->"b"->"b"`,`"a"->b"->"a"->"b"`, `"a"->b"->"b"->"a"`,  `"b"->a"->"a"->"b"`,`"b"->a"->"b"->"a"`, `"b"->b"->"a"->"a"`。**去除重复的关键是: 对于兄弟节点，不能是相同的字符。**

##### 算法实现

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <set>
#include <unordered_set>
using namespace std;

class Solution {
public:
    vector<string> GeneratePermutations(string str){
        vector<string> res;
        // 当字符串为空时，返回空列表
        if (str.size() == 0)
            return res;
        // 当字符串长度为1时，全排列只有一个(它自身)。
        if (str.size() == 1) {
            res.push_back(str);
            return res;
        }

        // 生成 str 从 0 开始, 到 str.end()这些字符的全排列，
        // 并存放到 res 中返回。
        genPermatations(str, 0, res);

        return res;
    }

private:
    void genPermatations(string str, int pos, vector<string> &res) {
        // 到达最后一个字符(边界)，搜索树已经达到叶子结点，路径放入res返回。
        if (pos + 1 == str.size()) {
            res.push_back(str);
            return;
        }
        unordered_set<char> ch;
        for(int i = pos; i < str.size(); i++) {
            // 如果还没有相同的兄弟结点出现过，往下遍历。
            if (ch.find(str[i]) == ch.end()) {
                ch.insert(str[i]);  // 记录该结点已被遍历
                swap(str[pos], str[i]);
                genPermatations(str, pos + 1, res);
                swap(str[pos], str[i]);
            }
        }
        return;
    }
};

int main() {
    // test
    Solution solve;
    string str = "aadd";
    vector<string> res = solve.GeneratePermutations(str);
    cout << "permutation number: " << res.size() << endl;
    set<string> uniquePermutations;
    for(int i = 0; i < res.size(); i++) {
        uniquePermutations.insert(res[i]);
    }
    cout << "unique permutations number: " << uniquePermutations.size() << endl;

    return 0;
}
```

#### 有序去重全排列

​        同时加上上面两个限制条件，生成的全排列里，不能有重复，且按照字母序有序生成。

##### 算法思路

​        组合解决上诉两个问题的思路。

##### 算法实现

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <set>
#include <unordered_set>
#include <algorithm>
using namespace std;

class Solution {
public:
    vector<string> GeneratePermutations(string str){
        vector<string> res;
        // 当字符串为空时，返回空列表
        if (str.size() == 0)
            return res;
        // 当字符串长度为1时，全排列只有一个(它自身)。
        if (str.size() == 1) {
            res.push_back(str);
            return res;
        }

        // 生成 str 从 0 开始, 到 str.end()这些字符的全排列，
        // 并存放到 res 中返回。
        genPermatations(str, 0, res);

        return res;
    }

private:
    void genPermatations(string str, int pos, vector<string> &res) {
        // 到达最后一个字符(边界)，搜索树已经达到叶子结点，路径放入res返回。
        if (pos + 1 == str.size()) {
            res.push_back(str);
            return;
        }

        // 对后面的字符进行排序，从而较小字母序的字符结点，先被访问。
        // 这样，遍历路径的顺序即为字符序从小到大。
        sort(str.begin() + pos, str.end());

        unordered_set<char> ch;
        for(int i = pos; i < str.size(); i++) {
            // 如果还没有相同的兄弟结点出现过，往下遍历。
            if (ch.find(str[i]) == ch.end()) {
                ch.insert(str[i]);  // 记录该结点已被遍历
                swap(str[pos], str[i]);
                genPermatations(str, pos + 1, res);
                swap(str[pos], str[i]);
            }
        }
        return;
    }
};

int main() {
    // test
    Solution solve;
    string str = "aadd";
    vector<string> res = solve.GeneratePermutations(str);
    cout << "permutation number: " << res.size() << endl;
    set<string> uniquePermutations;
    for(int i = 0; i < res.size(); i++) {
        uniquePermutations.insert(res[i]);
    }
    cout << "unique permutations number: " << uniquePermutations.size() << endl;

    for(int i = 0; i < res.size(); i++) {
        cout << res[i] << endl;
    }

    return 0;
}
```
