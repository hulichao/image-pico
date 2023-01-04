# 算法-Java多线程协同 && 红包随机派发算法

# 1.使用Java多线程协同打印1到n

题目要求如下：

-   对给定整型n, 开启n个线程（编号分别为1到n）。
-   线程的工作逻辑为：编号为x的线程只能打印整数x，
-   实现代码逻辑，使得n个线程协同工作按顺序打印自然数列：1, 2, 3, ..., n。

**思路**：多个线程自旋等待是否任务轮到自己了。

```java
public class ThreadDemo {
    //当前正在执行任务，全局可见
    public static volatile char now;

    public static void main(String[] args) {
        //输入任务
        char[] input = {'1','2','3','4','5'};
        //每个线程要执行任务数
        final int n = 1;
        for (int j = 0; j < input.length; j++) {
            final int t = j;
            //开启线程任务
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < n; i++) {
                        //循环等待，自旋
                        while (now != input[t]) {}
                        System.out.print(input[t]);//处理任务
                        //修改当前执行任务的全局状态
                        if (t + 1 < input.length)
                            now = input[t + 1];
                        else
                            now = input[0];
                    }
                }
            });
            thread.start();
        }
        //边界
        now = input[0];
    }
}
```

# 2.编写一个红包随机算法

题目要求如下：

-   请编写一个红包随机算法。需求为：给定一定的金额，一定的人数，保证每人都能随机获得一定的金额。
-   比如100元的红包，10个人抢，每人分得一些金额。约束条件为，最佳手气金额不能超过最大金额的90%。请给出java代码实现。

思路：从最大最小的金额中间取一个随机数，生成对应的数字后，如果多了再向平均线靠近，如果少了增加一点点

```java
import java.util.Random;

public class RedBagDemo {

    //随机数种子
    static Random random = new Random();

    public static void main(String[] args) {

        long[] result = generate(100, 10, 90, 1);
        for (long l : result) {
            System.out.println(l);
        }

    }

    //放大取随机再缩小
    static long xRandom(long min, long max) {
        return sqrt(nextLong(sqr(max - min)));
    }

    //红包总额度，人数，红包最大金额，红包最小金额

    public static long[] generate(long total, int count, long max, long min) {

        long[] result = new long[count];

        //取平均值，从平均值上加减
        long average = total / count;

        for (int i = 0; i < result.length; i++) {
            //红包大了，往平均线上减
            if (nextLong(min, max) > average) {
                long temp = min + xRandom(min, average);
                result[i] = temp;
                total -= temp;
            } else {
                //红包小了，往平均线上减
                long temp = max - xRandom(average, max);
                result[i] = temp;
                total -= temp;
            }
        }

        // 余钱，给不超过最大额的人每个都加一块
        while (total > 0) {
            for (int i = 0; i < result.length; i++) {
                if (total > 0 && result[i] < max) {
                    result[i]++;
                    total--;
                }
            }
        }

        // 如果总额小于0，按人头减1
        while (total < 0) {
            for (int i = 0; i < result.length; i++) {
                if (total < 0 && result[i] > min) {
                    result[i]--;
                    total++;
                }
            }
        }
        return result;
    }

    //缩小
    static long sqrt(long n) {
        return (long) Math.sqrt(n);
    }

    //放大
    static long sqr(long n) {
        return n * n;
    }

    //下一个不超过n的随机数
    static long nextLong(long n) {
        return random.nextInt((int) n);
    }
    //下一个在min 和 max间的随机数

    static long nextLong(long min, long max) {
        return random.nextInt((int) (max - min + 1)) + min;
    }

}
```

