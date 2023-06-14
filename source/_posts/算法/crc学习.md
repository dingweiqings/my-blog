---
title: 算法/crc学习
date: 2022-11-21 11:26:27
tags:
---
## 奇偶校验
## 磁盘冗余阵列
## 数学原理
### 多项式代数和多项式除法
高等代数中的多项式代数知识
```
M/N=Q...R 

```

这里容易得到，R会比N幂次要低，就和整数除法m/n=q..r ，0<=r<n 一样

1. 除法规则-竖式除法
每次消去余数最高项，直到余数最高次项小于除数
[!除法](../../assets/divide.png)

大牛在1961年发布的论文 https://en.wikipedia.org/wiki/W._Wesley_Peterson


## 算法
1. 朴素的竖式多项式除法的翻译
这里主要思路是用商消去最高位，然后在输入的数组上减去商*生成多项式，因此余数也是保存在输入的被除多项式位置
```
int divide( double num[], int nlen,
            double den[], int dlen,
            double quotient[], int *qlen )
{
    int n, d, q;
    // The lengths are one more than the last index; decrement them
    // here so the call is less confusing
    nlen--;
    dlen--;
    q = 0;
    // when n > dlen, the result is no longer a polynomial
    // (e.g. trying to divide x by x^2
    for ( n = nlen; n >= dlen; n-- )
    {
      // First, divide the nth element of numerator with the last element
      // of the denominator
      quotient[ n - dlen ] = num[ n ] / den[ dlen ];
      q++;
      // Now, multiply each element of the denominator by each
      // corresponding element of the numerator and subtract the
      // result
      for ( d = dlen; d >= 0; d-- )
      {
        num[ n - ( dlen - d ) ] -= den[ d ] * quotient[ n - dlen ];
        //采用模2除法，需要修改成这行代码
        num[ n - ( denlen - d ) ] = fabs( num[ n - ( denlen - d ) ] );
      }
    }
    *qlen = q;

    return ( nlen - *qlen + 1 );
}
```


1. 使用异或来计算
这里比较难理解，让我想了很久，还是基于竖式除法的思路，从竖式除法我们可以发现，32位的生成多项式最高位可以不存储，因为每次都是消去最高项；可以用一个32位int来保存余数，长度小于32位的二进制序列余数就是它自己，因此需要在右端补上32位；从竖式除法中我们可以发现，做减法的次数等于被除数长度-除数长度+1；
```
unsigned long int compute_crc( unsigned long input,
                               int len,
                               unsigned long divisor )
{
  //要做被除数长度-除数长度+1=24+32-33+1=24  
  while ( len-- )
  {
    //如果最高位是1，那么这位是要被消去，余数等于剩下的和除数异或；如果最高位是0，我们发现最高位死0，实际上是
    //商为0，和全0异或等于它自己，等效于直接左移
    input = ( input & 0x80000000 ) ? divisor ^ ( input << 1 ) : ( input << 1 );
  }
  //余数存储在32位，这里实际上等价于竖式多项式的低32位；因为运算过程中，余数始终保存在input位置
  return input;
}

...
  unsigned long int crc32_divisor = 0x04C11DB7;
  //下面的input按ABC实际上的值字节做了反转，多项式是低字节在低位，为了位对齐
  //   0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 1.0, // C
  //   0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, // B
   //  0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 0.0, 1.0  // A
  unsigned long int input = 0x8242C200; // ABC; backwards & left aligned

  printf( "%lx\n", compute_crc( input, 24, crc32_divisor ); // 5A5B433A
```


## 常见组件关于crc的实现
1. mariadb中用的是查表，或者使用cpu的sse来计算
2. zlib 


# 引用
1. 这是大牛的ppt讲解，https://pdfs.semanticscholar.org/44c1/4780d58015f8411fb85efa58a4aa3747a6ad.pdf
2. 论文原版https://apt.cs.manchester.ac.uk/ftp/pub/apt/papers/Peterson-Brown_61.pdf
1. https://www.shuxuele.com/algebra/polynomials-division-long.html
2. 模2除法https://blog.csdn.net/weixin_39450145/article/details/83987836
3. gcc中crc32的实现
4. mysql中crc32的实现
5. crc32 校验和 https://commandlinefanatic.com/cgi-bin/showarticle.cgi?article=art008
6. 常用的除数多项式
7. github doc https://github.com/komrad36/CRC
