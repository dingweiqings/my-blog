---
title: yacc使用
date: 2023-09-25 11:10:26
tags: yacc
categories: 编译器
---
# 简单介绍下编译原理
词法分析->语法分析->生成中间代码->机器无关的优化->生成汇编(机器相关的优化)->交给汇编器生成机器码
编译原理我当时看完了龙书+斯坦福的编译器公开课之后,对其中细节还是一知半解,当时就暂时先放下了.最开始的递归下降,nfa,dfa直接给我劝退了。
AST啥的也理解不透彻.
# 词法分析和语法分析框架
lex和yacc是unix下的词法/语法分析框架,
flex/bison是词法/语法分析框架
bison和yacc实际上用途是同一种，不过在不同平台下,yacc已经成为语法分析框架的代名词。下文统一使用lex/yacc

# Lex和YACC内部是如何工作的？
main ->yyparse() -> yylex()
yylex 会读取在yyin这个变量中的文件(没有则默认是stdin), yylex不断解析输入流,yyparse不断推导产生式并执行相应动作(在语法文件中定义),直到结束.
**注意：所以现在语法推导和lex解析算法大多都是O(N),只扫描一遍输入流不回退**
# lex
## hello world
这个框架实际上会根据给定的配置,来解析字符串
```helloworld.lex
%{
#include <stdio.h>
%}
%%
stop       printf(“Stop command received\n”);
start      printf(“Start command received\n”);
%%
``` 
第一部分，位于%{和%}对之间直接包含了输出程序(stdio.h).我们需要这个程序，因为使用了printf函数，它在stdio.h中定义.

第二部分用’%%’分割开来，所以第二行起始于’stop’，一旦在输入参数中遇到了’stop’，接下来的那一行(printf()调用)将被执行.
```
匹配的字符串名称 执行动作
```
我们这里是固定字符串stop,start

除此之外，还有’start’，其跟stop的行为差不多.
执行以下命令：
编译时，我们增加『-ll』编译选项，因为libl会提供main函数.

```
flex helloworld.lex
gcc lex.yy.c –o example –ll
```
![输出](/img/hello-lex.png)
## 正则表达式
```
%{
#include <stdio.h>
%}

%%
[0123456789]+     printf(“NUMBER\n”);
[a-zA-Z][a-zA-Z0-9]*  printf(“word\n”);
```
执行以下命令：
```
flex regex.lex
gcc lex.yy.c –o example –ll
```
![输出](/img/regex.png)
## lex中的函数和变量
- ytext char * 当前匹配的字符串
- yleng int 当前匹配的字符串长度
- yin FILE * lex当前的解析文件，默认为标准输出
- yout FILE * lex解析后的输出文件，默认为标准输入
- ylineno int 当前的行数信息
- 宏    
    CHO #define ECHO fwrite(yytext, yyleng, 1, yyout) 也是未匹配字符的默认动作
- int yylex(void) 调用Lex进行词法分析
- int yywrap(void) 在文件(或输入)的末尾调用。如果函数的返回值是1，就停止解析。 因此它可以用来解析多个文件。代码可以写在第三段，这样可以解析多个文件。 方法是使用 yyin 文件指针指向不同的文件，直到所有的文件都被解析。最后，yywrap() 可以返回1来表示解析的结束。

# yacc
YACC可以解析输入流中的标识符(token)，这就描述了YACC和LEX的关系，YACC并不知道『输入流』为何物，它需要事先就将输入流预加工成标识符，我们也可以自己手工写一个Tokenizer.
## yacc中的函数
- yyerror()在YACC发现一个错误的时候被调用，我们只是简单的输出错误信息.
- yywrap()函数用于不断的从一个文件中读取数据，当遇到EOF时，你可以再输入一个文件，然后返回0，你也可以使得其返回1，暗示着输入结束
- 这里有一个main()函数，它基本什么也不做，只是调用一些函数.可以单独定义也可以就放在grammer.y中
- yyparse() 会解析输入流中的token,并结合产生式推导
## 产生式
编译原理中的表达语法的上下文无关文法,确定了字符串的生成变换规则.这里需要将固定的语法拆分成字符串变换规则来理解,如果一个字符串没办法从终结符号结合变换规则得到，则该字符串不符合语法,需要抛出sytnax error.常见的语法分析算法有递归下降,LALR等
# 实例(实现一个固定语法的温度控制器) 
我们希望实现下面的功能,如果用户输入heat on 就打开温度控制器,输入heat off就关闭温度控制器,用户还可以通过target temperature set xxx来设定温度值
## lexer
```heat.lex
%{
#include <stdio.h>
#include "y.tab.h"
%}
%%
[0-9]+         { yylval = atoi(yytext); return NUMBER;}
heat           return TOKHEAT;
on|off         return STATE;
target         return TOKTARGET;
temperature    return TOKTEMPERATURE;
\n             /* ignore end of line */;
[ \t]+         /* ignore whitespace */
%%
```
lex根据正则来确定TOKEN,这个也是在yacc产生式中的终结符
## grammer
```heat-grammer.y
%{
#include <stdio.h>
#include <string.h>
void yywrap()
{
    return  1;
}
void yyerror(char * errmsg){
    printf("%s\n",errmsg);
}

int main()
{
    yyparse();
}
%}
//这里需要和lex中返回的token一致
%token  NUMBER TOKHEAT STATE TOKTARGET TOKTEMPERATURE
%%
commands: /* empty */
    | commands command
;

command: heat_switch
    | target_set
;

heat_switch:
    TOKHEAT STATE
    {
        printf("\tHeat turned on or off\n");
    }
;

target_set:
    TOKTARGET TOKTEMPERATURE NUMBER
    {
        //展示区别,这里给输入参数+5
        printf("\tTemperature set %d\n",$3 + 5);
    }
;
%%
```

## 输出
对于设定的温度,我为了展示有运算逻辑而不是直接拷贝字符串,我特意对参数做了一个加5的逻辑
执行下列命令
```
flex heat.lex
bison -d heat-grammer.y  --file-prefix y
gcc y.tab.c lex.yy.c
```
![温度控制](/img/heat-output.png)

## yacc中的union类型(这种是实际工程中才使用的)

### 温度控制器的新语法
用下面方式控制，先选择heater，再设置温度
heater mainbuilding

    Selected ‘mainbuilding’ heater

Target temperature 23

    ‘mainbuilding’ heater target temperature now 23

### yacc中的变量
```
%{
#include <stdio.h>
#include <string.h>
extern int yydebug = 1;
char * heater=NULL;
void yywrap()
{
    return  1;
}
void yyerror(char * errmsg){
    printf("%s\n",errmsg);
}

int main()
{
    yyparse();
}
%}

%token  TOKHEAT TOKTARGET TOKTEMPERATURE  TOKHEATER 
%union
{

    int number;

    char *string;
}

%token <number> STATE

%token <number> NUMBER

%token <string> WORD

%%
commands: /* empty */
    | commands command
;

command: heat_switch
    | target_set | heater_select
;

heat_switch:
    TOKHEAT STATE
    {
         if ($2)
            printf("\tHeat turned on\n");
         else
            printf("\tHeat turned off\n");
    }
;

target_set:
    TOKTARGET TOKTEMPERATURE NUMBER
    {
        //展示区别,这里给输入参数+5
        printf("\tHeater '%s' temperature set to %d\n", heater, $3+5);
    }
;
heater_select :

    TOKHEATER WORD

    {

       printf("\tSelected heater ‘%s’\n", $2);

       heater = $2;

}

    ;
%%
```
这次我们在command产生式增加了heater_select,并且定义了一个全局变量来接收选择的温度控制器.token处，我们定义了一个union变量,用于处理输入中不同的数据类型.此处表示yylval是一个union类型,可用来处理不同的数据类型


### 语法
```
%{
#include <stdio.h>
#include "y.tab.h"
%}
%%
[0-9]+         { yylval.number = atoi(yytext); return NUMBER;}
heater         return TOKHEATER;
heat           return TOKHEAT;
on|off         { yylval.number = !strcmp(yytext,  "on"); return STATE; }
target         return TOKTARGET;
temperature    return TOKTEMPERATURE;
[a-z0-9]+     yylval.string = strdup(yytext); return WORD;
\n             /* ignore end of line */;
[ \t]+         /* ignore whitespace */
%%
```
细心的同学可能已经发现了,这次对于yylval的赋值已经是具体到成员了

### 输出
对于设定的温度,我为了展示有运算逻辑而不是直接拷贝字符串,我特意对参数做了一个加5的逻辑
![输出](/img/yacc-with-union.png)


# yacc debug
当调试你的语法时，在YACC命令行中添加—debug和—verbose选项，在你的C文件头中添加以下语句：
int yydebug = 1;这将生成一个y.output文件，其中说明了所创建的那个状态机.
可以开启debug语法信息,详细查看yacc状态机和根据产生式推导的过程.这里我mark一下,以便后续自己查阅.
```
-t(--debug) -v
bison -t -v -d heat-grammer.y -b y 
```
# 测试代码见
[我的github代码](https://github.com/dingweiqings/study/tree/master/compiler_study/src/yacc)

# 推广
我们发现lex和yacc可以用来处理有固定语法的字符串,那么我们可以用其来解析配置文件,解析sql,解析json等等.好啦,现在你可以去看mysql和pg的sql解析啦,为学到新知识开心.
[mysql yacc](https://github.com/mysql/mysql-server/blob/8.0/sql/sql_yacc.yy)
[Postgres yacc](https://github.com/postgres/postgres/blob/master/src/backend/parser/gram.y)

# 引用
1. [flex和bison](https://cnblogs.com/itech/archive/2012/03/04/2375746.html)
2. [flex和bison实现计算器](https://blog.csdn.net/u014015972/article/details/51480680)