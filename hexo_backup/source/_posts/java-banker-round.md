---
title: java 实现四舍六入
date: 2020-12-17 17:41:11
tags:
---


## 在金融业务场景中，计算金额经常需要保留位数，由于保留位数有限，经常需要取舍的情况，往往在电商、银行系统中，金额是以整数形式保存，单位为货币最小单位，例如分。但是在结算时额外的参数如折扣、利率、税率等存在着大量的浮点数，计算结果则需要转换为整数。

### 简单处理一般是四舍五入，但是这样存在很明显的问题，就是 “入” 的概率大于 “舍”，明显的，遇到 1、2、3、4 舍，遇到 5、6、7、8、9 入，粗看这种就可以发现问题。如果想要两边平衡，则 “四舍六入” 才是合理的，但是，5 怎么办？
具体问题可以看下这个回答

[https://www.zhihu.com/question/28943072/answer/42673180](https://www.zhihu.com/question/28943072/answer/42673180)

### 在大量样本中，四舍五入后的计算结果的总和会明显大于直接计算总和的结果，对金融单位计算利息来说，这样很显然是一个亏本的行为，如果不亏本的算，依旧是简单处理，那么结果相反，客户就不开心了，在 sql 中经常也需要进行四舍六入操作，这里我们可以用java 实现，然后编译成 `udf`，就可以在 sql 中使用了。

### 银行家舍入（Banker's Round）
亦叫做 “四舍六入五成双” ，四舍六入，使得两头（即进和舍）概率相等，但是，在 4 和 6 之间的 5 就需要特别对待。具体规则如下：

舍去位的数值小于5时，直接舍去；
舍去位的数值大于等于6时，进位后舍去；
当舍去位的数值等于5时，分两种情况：5后面还有其他数字（非0），则进位后舍去；若5后面是0（即5是最后一位），则根据5前一位数的奇偶性来判断是否需要进位，奇数进位，偶数舍去。
舍去位，当小于 5，即 0 ~ 4.999999…… 则舍去，大于 6，即 6 ~ 10 则进位，则中间区间那个数字，5 ~ 5.999999…… ，只要使该区间内存在的数字平均分布，即可保证取舍概率相等。于是得到上述算法。
<!-- more -->
按上述规则，之前的 2.55 + 3.45 = 6 得出的结果如下：

2.6 + 3.4 = 6
在 java 中，BigDecimal 中 `ROUND_HALF_EVEN` 实现了 银行家舍入，具体参考 [ROUND_HALF_EVEN](https://docs.oracle.com/javase/7/docs/api/java/math/BigDecimal.html#ROUND_HALF_EVEN)
`
ROUND_HALF_EVEN
public static final int ROUND_HALF_EVEN
Rounding mode to round towards the "nearest neighbor" unless both neighbors are equidistant, in which case, round towards the even neighbor. Behaves as for ROUND_HALF_UP if the digit to the left of the discarded fraction is odd; behaves as for ROUND_HALF_DOWN if it's even. Note that this is the rounding mode that minimizes cumulative error when applied repeatedly over a sequence of calculations.
See Also:
Constant Field Values
`
大概意思就是说，ROUND_HALF_EVEN 模式会取最近的邻居除非两个邻居都是相等的，无论在哪种情况下，round 会取偶数的邻居，如果保留位数的左边数字是奇数的话会向上进位，如果左边位数是偶数的话会向下舍去，这是舍入模式,最大限度地减少累积误差时反复应用一系列的计算。
### 具体实现和测试代码如下:
```java

package com.company;

import java.math.BigDecimal;
import java.math.MathContext;
import java.math.RoundingMode;
import java.util.ArrayList;

public class Main {

    public static void main(String[] args) {
        // write your code here
        /****
         * 银行家舍入
         * 也叫做 四舍六入五成双，使得两头(即进和舍)概率相等，但是，在4和6之间的5需要特别对待，
         * 具体规则如下:
         * 1 舍去位的数值小于5时，直接舍去；
         * 2 舍去位的数值大于等于6时，进位后舍去；
         * 3 当舍去位的数值等于5时，分两种情况：5后面还有其他数字（非0），
         * 则进位后舍去；若5后面是0（即5是最后一位），则根据5前一位数的奇偶性来判断是否需要进位，
         * 奇数进位，偶数舍去。
         * 舍去位，当小于 5，即 0 ~ 4.999999…… 则舍去，大于 6，
         * 即 6 ~ 10 则进位，则中间区间那个数字，5 ~ 5.999999…… ，
         * 只要使该区间内存在的数字平均分布，即可保证取舍概率相等。于是得到上述算法。
         */
        ArrayList<BigDecimal> arrayList = new ArrayList<>();
        arrayList.add(new BigDecimal("11.554"));
        arrayList.add(new BigDecimal("11.556"));
        arrayList.add(new BigDecimal("11.5551"));
        arrayList.add(new BigDecimal("11.550"));
        arrayList.add(new BigDecimal("11.560"));
        for (int i = 0; i < arrayList.size(); i++) {
            BigDecimal bigDecimal = arrayList.get(i);
            bigDecimal = bigDecimal.round(new MathContext(4, RoundingMode.HALF_EVEN));
            System.out.println("before round: " + arrayList.get(i) + "  afer round: " + bigDecimal);
        }
    }
}

```
运行如下:
```
bash
before round: 11.554  afer round: 11.55
before round: 11.556  afer round: 11.56
before round: 11.5551  afer round: 11.56
before round: 11.550  afer round: 11.55
before round: 11.560  afer round: 11.56

Process finished with exit code 0
```
hive UDF 的注册和开发可以参考
[https://www.cnblogs.com/swordfall/p/11167486.html](https://www.cnblogs.com/swordfall/p/11167486.html)
[https://www.jianshu.com/p/c4bc53463379](https://www.jianshu.com/p/c4bc53463379)
[https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF)



