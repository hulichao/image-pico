# 算法-回溯问题解决框架

# 1.回溯问题简介

回溯问题,就是决策树的遍历过程,回溯问题需要有下面几个问题考虑

-   路径: 已经做出的选择,即从根节点到当前节点的路径
-   选择列表: 当前情况下还可以做哪些选择,即继续往下遍历节点,可以走哪些路
-   结束条件: 到达决策树的叶子节点,或者不满足条件停止

# 2.回溯问题框架

明白回溯问题的几个问题后,我们来看,回溯的基本框架

```java
List<String> result = new ArrayList<String>();
public void backtrack(已经选择的路径,可以做选择的列表) {
  if (满足结束条件)
    result.add(已经选择的路径);
    return;
    
   for 选择 in 可以做选择的列表 {
     选择列表.remove(选择);
     已经选择的路径.add(选择);
     backtrack(已经选择路径,可以做选择的路径);
     
     //撤销
     已经选择的路径.remove(选择);
     可以做选择的列表.add(选择);
   }
} 
```

# 3.案例

光学不练假把式,在看到思路前,先自己写一遍实现,再看有没有可以优化的地方.

## 3.1 字符串全排列

给定一个 **没有重复** 数字的序列，返回其所有可能的全排列。来源: [https://leetcode-cn.com/problems/permutations/](https://leetcode-cn.com/problems/permutations/ "https://leetcode-cn.com/problems/permutations/")

**示例:**

```java
//输入: [1,2,3]
//输出:
[
  [1,2,3],
  [1,3,2],
  [2,1,3],
  [2,3,1],
  [3,1,2],
  [3,2,1]
]
```

```java
class Solution {
    List<List<Integer>> result = new ArrayList<List<Integer>>();
    public List<List<Integer>> permute(int[] nums) { 
        int[] visited = new int[nums.length];      
        backtrack(new ArrayList<Integer>(), nums, visited);
        return result;
    }

    public void backtrack(List<Integer> path, int[] nums, int[] visited) {
    
        //边界，满足条件
        if (path.size() == nums.length) {
            result.add(new ArrayList(path));
            return;
        }
        
        for (int n = 0; n < nums.length; n++) {
            if (visited[n] == 1) continue;
            visited[n] = 1;
            path.add(nums[n]);
            backtrack(path, nums, visited);

            path.remove(path.size() - 1);
            visited[n] = 0;

        }
    }
}
```

**注**: 其中`res`是全局返回,tmp是当前路径上的节点,`visited `和 `nums `来标识当前还有哪些节点可以访问

## 3.2  合法ip地址

给定一个只包含数字的字符串，复原它并返回所有可能的 IP 地址格式。来源:[https://leetcode-cn.com/problems/restore-ip-addresses/](https://leetcode-cn.com/problems/restore-ip-addresses/ "https://leetcode-cn.com/problems/restore-ip-addresses/")

有效的 IP 地址 正好由四个整数（每个整数位于 0 到 255 之间组成，且不能含有前导 0），整数之间用 '.' 分隔。

例如："0.1.2.201" 和 "192.168.1.1" 是 有效的 IP 地址，但是 "0.011.255.245"、"192.168.1.312" 和 "192.168\@1.1" 是 无效的 IP 地址。

示例 1：

```java
输入：s = "25525511135"
输出：["255.255.11.135","255.255.111.35"]
示例 2：
```

```java
输入：s = "0000"
输出：["0.0.0.0"]
示例 3：
```

```java
输入：s = "1111"
输出：["1.1.1.1"]
示例 4：
```

```java
输入：s = "010010"
输出：["0.10.0.10","0.100.1.0"]
示例 5：
```

```java
输入：s = "101023"
输出：["1.0.10.23","1.0.102.3","10.1.0.23","10.10.2.3","101.0.2.3"]
```

**解决代码**

```java
class Solution {
    public List<String> restoreIpAddresses(String s) {
        List<String> res = new ArrayList();
        List<String> temp = new ArrayList();
        helper(res,temp,s);
        return res;
    }

    void helper(List<String> res,List<String> temp,String next) {
        if(temp.size() > 4) {
            return;
        }
        if(temp.size() == 4 && next.length() == 0) {
            String ip = temp.get(0) + "." + temp.get(1) + "." + temp.get(2) + "." + temp.get(3);
            res.add(ip);
            return;
        }
        for(int i = 0; i < next.length(); i++) {
            String s = next.substring(0,i+1);
            if(s.length() > 1 && s.charAt(0) == '0') {
                continue;
            } 
            if(s.length() > 3) {
                continue;
            }
            if(s.length() == 3 && "255".compareTo(s) < 0) {
                continue;
            }
            temp.add(s);
            helper(res,temp,next.substring(i+1));
            temp.remove(temp.size() - 1);
        }
    }
}
```

**注**: 整体框架还是采用上面的结构,不过在判断当前节点是否应该加入temp时候,要做一点稍微复杂的判断
