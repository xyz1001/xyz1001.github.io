---
title: 算法---质数筛选法
author: 张帆
tags:
  - 算法
abbrlink: 10856
date: 2016-04-08 18:20:00
---

通过循环判断数n是否不能被所有不大于其平方根的数整除来判断一个数是否为质数或者输出一个小范围内（10000以内）的质数个数的算法相信大家都接触过，这种方法十分直接简单，但一旦在解决输出一个很大范围（大于10万）内所有质数或质数个数这类问题时就会因效率低下而导致时间复杂度很高。在此介绍一种通过筛除法查找质数的方法。

质数筛选法，顾名思义就是通过筛选的方式，将数字中的合数筛除，剩下的即质数。算法流程很简单，假设求`2-N`中的所有质数，将范围分为两部分，一部分为已筛除部分，由于2为质数，设定该部分初始范围为`1-1`；另一部分为待筛除部分，初始范围为`2-N`，计数量为`cnt`。算法流程如下：

<!--more-->

## 基本实现

1. 寻找当前待筛除部分第一个质数`i`，由于该数不能被已筛除部分所有质数整除，因而其一定为质数，计数量`cnt+1`；
2. 若`i==N`，说明整个范围均已筛除，不存在质数，程序结束；
3. 将待筛除部分是i倍数的数全部筛除；
4. 已筛除部分扩展至`2-i`，待筛除部分扩展至`(i+1)-N`，跳转至1；

``` c++
/**
 * 利用质数筛选法输出较大范围内质数数量
 */
#include <iostream>

using namespace std;
const int MAX = 100000000;   //质数查找范围
int main()
{
    int *num=new int[MAX], cnt = 0; //num数组下标i代表数i
    //全部置1，代表均为质数
    for(int i=0;i<MAX;++i)
    {
        num[i]=1;
    }

    int i = 2;  //第一个质数
    //根据num[i]是否为1判断i是否为质数
    while(i<MAX)
    {
        if(num[i] == 1) //如果当前数为剩下数字中第一个“质数”，则其为质数
        {
            //cout<<i<<' ';
            ++cnt;  //质数数量+1
            int j = 2, temp;
            //筛除剩下数字中该质数的倍数
            while(1)
            {
                temp = i * j;
                if(temp<MAX)
                {
                    num[temp] = 0;
                    ++j;
                }
                else
                    break;
            }
        }
        ++i;
    }
    cout<<endl<<cnt<<endl;
    return 0;
}
```

当范围为100000000（1亿）时，程序运行时间为6.107 s。

## 进一步优化

由于筛除过程中i的`2-(i-1)`倍数在之前已经被筛除（例如当筛除3的2倍=6时，在之前筛除2的3倍时已经完成)，因此只需筛除到N的平方根即可，优化如下：

``` c++
/*
 *利用质数筛选法输出较大范围内质数数量
 */
#include <iostream>
#include <cmath>
using namespace std;

const int MAX = 100000000;   //质数查找范围
int main()
{
    int *num=new int[MAX], cnt = 0; //num数组下标i代表数i
    //全部置1，代表均为质数
    int max_sqrt=int(sqrt(MAX))+1;
    for(int i=0;i<MAX;++i)
    {
        num[i]=1;
    }

    int i = 2;  //第一个质数
    //根据num[i]是否为1判断i是否为质数
    while(i<max_sqrt)
    {
        if(num[i] == 1) //如果当前数为剩下数字中第一个“质数”，则其为质数
        {
            //cout<<i<<' ';
            int j = i, temp;
            //筛除剩下数字中该质数的倍数
            while(1)
            {
                temp = i * j;
                if(temp<MAX)
                {
                    num[temp] = 0;
                    ++j;
                }
                else
                    break;
            }
        }
        ++i;
    }

    //遍历质数
    for(int i=2;i<MAX;++i)
    {
        cnt+=num[i];
        //if(num[i]==1)
            //cout<<i<<' ';
    }
    cout<<endl<<cnt<<endl;
    return 0;
}
```

当范围为100000000（1亿）时，程序运行时间为4.427 s

<script src="https://utteranc.es/client.js"
        repo="xyz1001/xyz1001.github.io"
        issue-term="title"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
