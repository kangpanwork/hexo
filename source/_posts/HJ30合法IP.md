---
title: HJ30合法IP
date: 2022-05-11 15:31:51
tags: 牛客网华为题目
---

## 描述

IPV4 地址可以用一个 32 位无符号整数来表示，一般用点分方式来显示，点将 IP 地址分成 4 个部分，每个部分为8 位，表示成一个无符号整数（因此正号不需要出现），如 10.137.17.1，是我们非常熟悉的IP地址，一个IP地址串中没有空格出现（因为要表示成一个 32 数字）。

现在需要你用程序来判断IP是否合法。

数据范围：数据组数：1≤*t*≤18 

进阶：时间复杂度：O(n) ，空间复杂度：O(n)



### 输入描述：

输入一个ip地址，保证不包含空格

### 输出描述：

返回判断的结果YES or NO

## 题解

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

// 测试数据 
// a.1.1.1 
// .1.1.1
// 1.1.1.
// 01.1.1.1
// +1.1.1.1
public class HJ90 {
    public static void main(String[] args) throws IOException {
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(System.in));
        String str;
        while ((str = bufferedReader.readLine()) != null) {
            String result = checkIp(str);
            System.out.println(result);
        }
    }

    public static String checkIp(String str) {
        String result = "YES";
        String[] arrStr = str.split("\\.");
        if (arrStr.length != 4) {
            result = "NO";
        }
        for (String ele : arrStr) {
            if (ele.isEmpty() || ele.length() > 3) {
                result = "NO";
                break;
            }
            char[] chars = ele.toCharArray();
            boolean flag = false;
            for (int i = 0; i < chars.length; i++) {
                if (!Character.isDigit(chars[i])) {
                    result = "NO";
                    flag = true;
                    break;
                }
            }
            if (flag) {
                break;
            }
            if (chars.length > 1 && chars[0] == '0') {
                result = "NO";
                break;
            }
            int num = Integer.parseInt(ele);
            if (num > 255 || num < 0) {
                result = "NO";
                break;
            }
        }
        return result;
    }
}
```

