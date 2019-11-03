<font face="微软雅黑">

# KMP 算法笔记

---

实现 strStr() 函数。

给定一个 haystack 字符串和一个 needle 字符串，在 haystack 字符串中找出 needle 字符串出现的第一个位置 (从0开始)。如果不存在，则返回  -1。

<br>

**示例 1：**
```
输入: haystack = "hello", needle = "ll"
输出: 2
```

**示例 2：**
```
输入: haystack = "aaaaa", needle = "bba"
输出: -1
```

<br>

**说明：**

当 needle 是空字符串时，我们应当返回什么值呢？这是一个在面试中很好的问题。

对于本题而言，当 needle 是空字符串时我们应当返回 0 。这与C语言的 strstr() 以及 Java的 indexOf() 定义相符。

<br>

**思路：**
- 分别用变量 m 和 n 记录 haystack 字符串和 needle 字符串的长度。
- 若 n=0，返回 0，符合题目要求。
- 求出 needle 的 LPS，即最长的公共前缀和后缀数组。
- 分别定义两个指针 i 和 j，i 扫描 haystack，j 扫描 needle。
- 进入循环体，直到 i 扫描完整个 haystack，若扫描完还没有发现 needle 在里面，就跳出循环。
- 在循环体里面，当发现 i 指针指向的字符与 j 指针指向的字符相等的时候，两个指针一起向前走一步，i++，j++。
- 若 j 已经扫描完了 needle 字符串，说明在 haystack 中找到了 needle，立即返回它在 haystack 中的起始位置。
- 若 i 指针指向的字符和 j 指针指向的字符不相同，进行跳跃操作，j = LPS[j - 1]，此处必须要判断 j 是否大于 0。
- j=0，表明此时 needle 的第一个字符就已经和 haystack 的字符不同，则对比 haystack 的下一个字符，所以 i++。
- 若没有在 haystack 中找到 needle，返回 -1。

``` cpp {.line-numbers}
class Solution {
public:
    vector<int> getLPS(const string& s)
    {
        int n = s.size();
        vector<int> lps(n);
        int i = 1, len = 0; // lps 的第一个值一定是 0，即长度为 1 的字符串的最长公共前缀后缀的长度为 0，
                            // 直接从第二个位置遍历。
                            // 并且，初始化当前最长的 lps 长度为 0，用 len 变量记录下
        while (i < n)
        {
            if (s[i] == s[len]) lps[i++] = ++len;   // 若 i 指针能延续前缀和后缀，则更新 lps 值为 len+1
            
            else if (len > 0) len = lps[len - 1];   // 否则，判断 len 是否大于 0，
                                                    // 尝试第二长的前缀和后缀，是否能继续延续下去
            else ++i;                               // 所有的前缀和后缀都不符合，则当前的 lps 为 0，i++
        }
        return lps;
    }
    
    int strStr(string haystack, string needle)
    {
        int n = needle.size();
        int m = haystack.size();
        
        if (n == 0) return 0;
        
        vector<int> lps = getLPS(needle);
        
        int i = 0, j = 0;
        while (i < m)
        {
            if (haystack[i] == needle[j])
            {
                ++i, ++j;
                if (j == n) return i - n;
            }
            else if (j > 0) j = lps[j - 1];
            else ++i;
        }
        return -1;
    }
};
```
<br>

**复杂度分析：**

KMP 算法需要 O(n) 的时间计算 LPS 数组，还需要 O(m) 的时间扫描一遍 haystack 字符串，整体的时间复杂度为 O(m + n)。这比暴力法快了很多。