# 算法 - 如何从股票买卖中，获得最大收益

作为一名从副业中已赚取几个月工资的韭菜，显然对这类题目很有搞头，但是实际中往往不知道的是股票的未来价格，所以需要预测，而你的实盘实际上也会反过来影响股票，所以没人能完整预测股票的走势，那些从回溯中取的最大值的算法，就是下面的几种，有必要掌握一下，假若某一天你穿越回去，你任选一种算法，那么你就可以从1万到1个亿，可能一个月就够了，哦，对了，如果有人能穿越过去，记得带我一下。。。。。

## [121. 买卖股票的最佳时机](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock/ "121. 买卖股票的最佳时机")

```java
// 股票只允许买卖一次 可利用贪心 找到最小的min 价格 再去找最大的max 价格 那么两者之间的差值就是 结果

public int maxProfit(int[] prices) {
    if (prices.length == 0) return 0;
    int min = prices[0];
    int res = 0;
    for (int i = 1; i < prices.length; i++) {
        res =Math.max(res,prices[i]-min);
        min = Math.min(min,prices[i]);
    }
    return res;
}
```

## [122. 买卖股票的最佳时机 II](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-ii/ "122. 买卖股票的最佳时机 II")

```java
public int maxProfit(int[] prices) {
        if(prices.length == 0)return 0;
        int sell = 0;
        int buy = Integer.MIN_VALUE;
        for (int i = 0; i < prices.length; i++) {
            int t = sell;
            sell = Math.max(sell,prices[i] + buy);
            buy = Math.max(buy,t-prices[i]);
        }
        return sell;
}

```

## [123. 买卖股票的最佳时机 III](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-iii/ "123. 买卖股票的最佳时机 III")

```java
public int maxProfit(int[] prices) {
        if (prices.length == 0)return 0;
        int k = 2;
        int[][][] dp = new int[prices.length][k+1][2];
        for (int i = 0; i < prices.length; i++) {
            for (int j = 1; j <= k ; j++) {
                if (i == 0){
                    dp[i][j][1] = -prices[i];
                }else {
                    dp[i][j][0] = Math.max(dp[i-1][j][0],prices[i]+dp[i-1][j][1]);
                    dp[i][j][1] = Math.max(dp[i-1][j][1],dp[i-1][j-1][0]-prices[i]);
                }
            }
        }
        return dp[prices.length -1][k][0];
}
```

## [188. 买卖股票的最佳时机 IV](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-iv/ "188. 买卖股票的最佳时机 IV")

```java
//这个是 买入k 次 但是 k 大于 数组长度的一半时候 实际和无限买入情况是一样的 可以加快速度
    public int maxProfit(int k, int[] prices) {
        if (prices.length == 0) return 0;
        if ( k > prices.length/2){
            return fastMaxProfit(prices);
        }
        int[][][] dp = new int[prices.length][k+1][2];
        for (int i = 0; i < prices.length; i++) {
            for (int j = 1; j <= k; j++) {
                if (i == 0){
                    dp[i][j][1] = -prices[i];
                }else {
                    dp[i][j][0] = Math.max(dp[i-1][j][0],dp[i-1][j][1] + prices[i]);
                    dp[i][j][1] = Math.max(dp[i-1][j][1],dp[i-1][j-1][0] - prices[i]);
                }
            }
        }
        return dp[prices.length - 1][k][0];
    }

    int fastMaxProfit(int[] price) {
        if (price.length == 0) return 0;
        int sell = 0;
        int buy = -price[0];
        for (int i = 0; i < price.length; i++) {
            int t = sell;
            sell = Math.max(sell,price[i] + buy);
            buy = Math.max(buy,t - price[i]);
        }
        return sell;
    }
```

## [309. 最佳买卖股票时机含冷冻期](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/ "309. 最佳买卖股票时机含冷冻期")

```java
//有冷冻期 就是 sell 保存多一天
public int maxProfit(int[] prices) {
    if (prices.length == 0) return 0;
    int sell = 0;
    int prev = 0;
    int buy = -prices[0];
    for (int i = 0; i < prices.length; i++) {
        int t = sell;
        sell = Math.max(sell,prices[i] + buy);
        buy = Math.max(buy,prev - prices[i]);
        prev = t;
    }
    return sell;

}
```

## [714. 买卖股票的最佳时机含手续费](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-with-transaction-fee/ "714. 买卖股票的最佳时机含手续费")

```java
// 思路 每次 交易的时候再减去手续费即可
public int maxProfit(int[] prices, int fee) {
    if (prices.length == 0) return 0;
    int sell = 0;
    int buy = -prices[0] - fee;
    for (int i = 0; i < prices.length; i++) {
        int t = sell;
        sell = Math.max(sell,prices[i] + buy);
        buy = Math.max(buy,t - prices[i] - fee);
    }
    return sell;

}

```
