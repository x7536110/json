## 记录一个简易的C语言 json库开发过程
>lptjson
>本文根据Milo Yip的开源教程-[从零开始的 JSON 库教程](https://github.com/miloyip/json-tutorial) 加工而成，以记录学习中的疑虑与自我解答为目的，如有错误请您及时指出，我会非常感谢您的指点。
### 1.什么是JSON
JSON(JavaScript Object Notation)-JS对象标记，是一种用于交换数据的文本格式，主要用于数据的储存、交换、传播等等。JSON即易于人类阅读，也易于机器解析和生成，提高了工作效率。
相比较于XML,YAML等，JSON的语法最为简单。举个例子，现在我们要表示中国的部分省及城市的数据：
```JavaScript
{
    "name": "中国",
    "province": [{
        "name": "黑龙江",
        "cities": {
            "city": ["哈尔滨", "大庆"]
        }
    }, {
        "name": "广东",
        "cities": {
            "city": ["广州", "深圳", "珠海"]
        }
    }, {
        "name": "台湾",
        "cities": {
            "city": ["台北", "高雄"]
        }
    }, {
        "name": "新疆",
        "cities": {
            "city": ["乌鲁木齐"]
        }
    }]
}
```
我们可以很方便的由这个JSON解析到中国黑龙江省有哈尔滨和大庆这两个城市，广东有广州、深圳、珠海三个城市…
### 2.json怎么用
> 首先我们需要了解一些json的语法，json的语法非常简单，json中只包含6种数据类型/标记：
- `{}`表示对象
- `[]`表示数组
- `".."`表示字符串
- `true`或`false`表示布尔值
- 浮点数采用一般的浮点数表示法
- `null`表示null
基于以上6种表示类型，我们基本上可以描述所有的对象。
> 数据保存在键值对中
形如`key:value`的形式存储。如
``` JavaScript
"name":"Zemin Jiang"`
```
> 数据之间由逗号进行分隔
``` JavaScript
"age":"+∞",
"color":"black"
```
对JSON对象的解析可以参考[ECMA-404](http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf)

### 3.实现目标
我们的目的是，假设已知目标json结构的情况下，能够为我们的程序提供一个直接从json中提取数据的方法。
1.把json文本解析为一个树状结构（parse）。
2.提供接口访问该树状结构（access）。
3.把数据结构转化为json文本（stringify）。

### 4.实现过程

#### 4.1 实现基本的解析
本节内容将实现对null、true、false类型的解析。

首先我们需要手撸一个测试框架，用来测试每一个单元的功能。

```c
static int main_ret=0;
static int test_count=0;
static int test_pass=0;

#define EXPECT_EQ_BASE(equality,expect,actual,format)\
    do{\
        test_count++;\
        if(equality)\
            test_pass++;\
        else{\
            fprintf(stderr,"%s:%d: expect:"format " actual:" format"\n",__FILE__,__LINE__,expect,actual);\
            main_ret=1;\
        }\
    }while(0)
/*
这段宏的意思是，如果程序的执行结果通过测试（equality==true），则通过样例数+1。如果没有通过，则将没有通过测试的测试样例文件名（file）、行号（line）、预期结果（expect）、实际结果（actual）输出到stderr供debug使用
*/
```

我们首先实现对null、true、false类型的解析，因为三种类型都被定义为了枚举类型，所以我们使用一个测试宏`EXPECT_EQ_INT(expect,actual)`来进行测试
```c
#define EXPECT_EQ_INT(expect,actual) EXPECT_EQ_BASE((expect)==(actual),expect,actual,"%d")
```

接下来我们进入正题，开始编写工作代码。
我们使用C语言，因此将声明放置于`leptjson.h`文件中，将实现放置于`leptjson.c`文件中

首先我们需要在头文件中加入以下定义：
```c
typedef enum{
    LEPT_NULL,
    LEPT_FALSE,
    LEPT_TRUE,
    LEPT_NUMBER,
    LEPT_STRING,
    LEPT_ARRAY,
    LEPT_OBJECT
}lept_type;
```
以上枚举类型声明了Json中的全部7种对象类型，然后为了方便我们起一个别名
```c
typedef struct{
    double n;
    lept_type type;
}lept_value;
```
同时我们需要声明异常掩码为枚举类型,供测试使用
```c
enum {
    LEPT_PARSE_OK = 0,              //解析成功
    LEPT_PARSE_EXPECT_VALUE,        //解析到了非期望值
    LEPT_PARSE_INVALID_VALUE,       //输入内容非法
    LEPT_PARSE_ROOT_NOT_SINGULAR,   //输入对象过长
    LEPT_PARSE_NUMBER_TOO_BIG,      //解析的数字过大
};
```
接下来是我们要用到的函数
```c
int lept_parse(lept_value* v,const char* json);
lept_type lept_get_type(const lept_value* v);
```
`lept_parse()`需要内部调用如`lept_parse_null();lept_parse_true();lept_parse_false()...`的几个内部调用函数，我们将其实现在`leptjson.c`文件中
```c
//lept_parse()的实现
typedef struct {
    const char* json;
}lept_context;
/*...*/
int lept_parse(lept_value* v,const char* json){
    lept_context c;
    int ret;
    assert(v!=NULL);
    c.json=json;
    v->type=LEPT_NULL;
    lept_parse_whitespace(&c);
    if((ret=lept_parse_value(&c,v))==LEPT_PARSE_OK){
        lept_parse_whitespace(&c);
        if(*c.json!='\0')
            ret=LEPT_PARSE_ROOT_NOT_SINGULAR;
    }
    return ret;
}
```
我们将数据都放入一个叫做`lept_context`的结构体方便传递参数，之后开始对该结构体进行处理。首先我们使用assert保证传入了一个有效的对象（非空指针），之后我们将它设为null类型，并且调用`lept_parse_value()函数进行解析。
```c
static int lept_parse_value(lept_context* c,lept_value* v){
    switch(*c->json){
        case 'n':return lept_parse_null(c,v);
        case 't':return lept_parse_true(c,v);
        case 'f':return lept_parse_false(c,v);
        /*......*/
        case '\0':return LEPT_PARSE_EXPECT_VALUE;
        default: return LEPT_PARSE_INVALID_VALUE;
    }
}
```
根据JSON的语法我们可以得知，我们可以直接通过检测json对象的首字符判断该对象的类型，因此`lept_parse_value`根据检测到的类型调用对应的函数对JSON对象进行解析

> * `'n'-->null`
> * `'t'-->true`
> * `'f'-->false`
> * `'"'-->string`
> * `'['-->array`
> *  `'{'-->object`
> *  `'0'-'9'/'-'-->number`
```c
static int lept_parse_null(lept_context* c,lept_value* v){
    EXPECT(c,'n');
    if(c->json[0]!='u'||c->json[1]!='l'||c->json[2]!='l')
        return LEPT_PARSE_INVALID_VALUE;
    c->json+=3;
    v->type=LEPT_NULL;
    return LEPT_PARSE_OK;
}
```
以`lept_parse_null`为例，我们输入待解析的对象context，与结点v；只要依次检测字符串是否是"null"，即可输入的合法性。当输入不合法我们返回`LEPT_PARSE_INVALID_VALUE`,合法我们将v的值设置为null并返回`LEPT_PARSE_OK`

`true`与`false`的解析方式可以参考`null`写出来……
```c
static int lept_parse_true(lept_context* c,lept_value* v){
    EXPECT(c,'t');
    if(c->json[0]!='r'||c->json[1]!='u'||c->json[2]!='e')
        return LEPT_PARSE_INVALID_VALUE;
    c->json+=3;
    v->type=LEPT_TRUE;
    return LEPT_PARSE_OK;
}

static int lept_parse_false(lept_context* c,lept_value* v){
    EXPECT(c,'f');
    if(c->json[0]!='a'||c->json[1]!='l'||c->json[2]!='s'||c->json[3]!='e')
        return LEPT_PARSE_INVALID_VALUE;
    c->json+=4;
    v->type=LEPT_FALSE;
    return LEPT_PARSE_OK;
}
```
因为`true`与`false`与`null`都可以使用枚举类型进行掩码表示，所以写起来相对简单一些，接下来我们开始尝试解析数字。
#### 4.2对数字的解析
对数字的解析与对前文中类型的解析没有本质区别，也是根据[ECMA-404](http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf)中的Numbersb部分直接解析即可，整个逻辑非常简单，更直观的逻辑可以参考下图。
![ECMA-404-NUMBER](http://www.songlinglu99.com/pic/number.png)
