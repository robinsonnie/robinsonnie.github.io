# Lua/Luajit正则引擎分析

## 前言

所谓正则表达式，就是一种描述字符串结构模式的形式化表达方法。

一方面，因为正则表达式处理的对象是字符串，或者抽象为一个对象序列，而这恰恰是计算机体系的本质数据结构，我们围绕计算机所做的大多数工作，都归结为在序列上的操作，因此正则表达式用途广阔；另一方面，正则表达式拥有超强的结构描述能力，描述了结构本身，就具备描述了应用结构的系统的语义能力。

正因为这两点，在现在的软件开发和日常数据处理工作中，正则表达式已经成为必不可少的工具。但比较遗憾的是，对于很多软件开发者（包括作者本人），常常以散弹枪编程的方式，照猫画虎的拼装正则表达式以达到需求，但对其背后的原理知之甚少，这会导致：

* 正则表达式的性能多数情况下未被评估
   * 指数级回溯
   * 测试时仅测试最优情况，忽略了生产环境的最坏情况
   * 常见的正则表达式性能优化方法未被考虑
* 陷入正则表达式的各种细节和陷阱
   * 不同正则流派
   * 不同正则引擎实现
   * 元字符及各种细节的差异

## 动机

正则表达式在Unix发展历史、及Posix标准化过程中，衍生出了多种不同的正则引擎，不同正则引擎所提供的能力各不相同。

引擎类型|程序
---|:--:
DFA|awk（大多数版本）、egrep（大多数版本）、flex、lex、Mysql
NFA|Emacs、Java、grep、more、PCRE library、PERL、PHP、Python、sed、vi
Posix NFA|mawk
DFA/NFA混合|GNU awk、GNU grep/grep、Tcl

很多文章用来分析不同正则引擎之间的差异，比如：

* DFA匹配很迅速，但提供的特性少，比如不支持模式提取、环视
* NFA功能很强大，但性能需要关注
* 有些工具会集成双引擎，在正则表达式本身不需要使用NFA时，使用DFA来加快匹配速度
* 不同引擎的原理分析、性能优化方法

在了解了很多如上知识后，很多时候仍然觉得似懂非懂。本文直接从CODE出发，尝试剖析Lua/Luajit的正则表达式引擎，为进阶打下代码基础。

## 代码分析

### 核心数据结构

```
typedef struct MatchState {
  const char *src_init;  // 待匹配的目标字符串
  const char *src_end;   // 指向待匹配的目标字符串的尾部，便于判断是否结束
  const char *p_end;     // regex pattern，正则表达式本身
  lua_State *L;          // Lua虚拟机
  int matchdepth;        // 控制递归次数，避免C堆栈溢出
  unsigned char level;   // 模式匹配的捕获次数
  struct {
    const char *init;    // 指向成功模式匹配的字符串，init指向的src_init的某个段落
    ptrdiff_t len;       // 成功模式匹配的字符串长度
  } capture[LUA_MAXCAPTURES];
                         // 每次成功的模式匹配，均会由capture数组中的一个元素记录其
                         // 相关信息。
} MatchState;

#define MAXCCALLS   200
#define L_ESC       '%'
#define SPECIALS    "^$*+?.([%-"
```

* MatchState是NFA状态机，每次进行一次string.match()时，均会生成一个MatchState对象，用于记录NFA分析过程的状态。
* 在多选结构（```例如：a|b 表示a或者b```）、量词（```例如：a*b 表示0个或多个a后，紧跟着一个b```）的匹配过程中，均会涉及尝试、当尝试失败后的回溯至原位置、继续下一尝试。稍后源码分析会看到，尝试-回溯本质是一次递归调用，当出现一直尝试的情况下，可能会造成C堆栈溢出，故matchdepth记录的递归次数不得超过MAXCCALLS。
* 与很多其它正则流派使用```\```转义不同，Lua/Luajit使用```%```进行转义，L_ESC定义方便后续引用。
* SPECIALS定义了Lua/Luajit正则引擎支持的元字符，可以看到，不支持多选结构```|```，是个比较严重的硬伤，但可以通过其它方法弥补。


### 传动装置
```
    MatchState ms;
    const char *s1 = s + init - 1;  // 若匹配起始位置init=1，则s=s1=目标字符串首部
    int anchor = (*p == '^');       // p指向regex pattern首部
    if (anchor) {
      p++; lp--;  /* skip anchor character */
    }
    prepstate(&ms, L, s, ls, p, lp);
    do {
      const char *res;
      reprepstate(&ms);
      if ((res=match(&ms, s1, p)) != NULL) {
        if (find) {
          lua_pushinteger(L, (s1 - s) + 1);  /* start */
          lua_pushinteger(L, res - s);   /* end */
          return push_captures(&ms, NULL, 0) + 2;
        }
        else
          return push_captures(&ms, s1, res);
      }
    } while (s1++ < ms.src_end && !anchor);
```

* 一次string.match()调用的主体逻辑，便是```do {} while ()```，其会将目标字符串 && 正则表达式作为match()的输入，进行匹配。
   * 若匹配成功，则将匹配结果压入Lua堆栈后返回给调用者。
   * 若匹配失败，则传动装置工作，将目标字符串从下一个字符起始 && 正则表达式继续作为match()的输入，进行下一轮匹配。
   * 注意：若存在```^```锚点，则失败后传动装置直接终止循环，不再向后继续比较。
      * **优化：```^```锚点很重要的结构，如有起始标记，一定要使用锚点；如无起始标记，可尝试对目标字符串进行拆分后使用锚点。**

      
### 匹配主体
```
static const char *match (MatchState *ms, const char *s, const char *p) {
  if (ms->matchdepth-- == 0)
    luaL_error(ms->L, "pattern too complex");
  init: /* using goto's to optimize tail recursion */
  if (p != ms->p_end) {  /* end of pattern? */
    switch (*p) {
    }  
  }
  ms->matchdepth++;
  return s;
}
```
* match()函数是正则匹配逻辑的主体，其输入为目标字符串 + 正则表达式。
* ms->matchdepth用于控制迭代次数不超过MAXCCALLS(200)的限制。
   * NFA中的尝试 - 回溯，在Lua/Luajit中的实现，正是一次对match()自身的递归调用，当失败后，回溯到调用前的C堆栈位置，继续其它尝试。
   * MAXCCALLS的限制，从正则角度来看，可以避免过于复杂的正则表达式。 

### 匹配单独字符

```
static const char *match (MatchState *ms, const char *s, const char *p) {
  if (ms->matchdepth-- == 0)
    luaL_error(ms->L, "pattern too complex");
  init: /* using goto's to optimize tail recursion */
  if (p != ms->p_end) {  /* end of pattern? */
    switch (*p) {
      default: dflt: {
        const char *ep = classend(ms, p);  /* points to optional suffix */
        /* does not match at least once? */
        if (!singlematch(ms, s, p, ep)) {
          if (*ep == '*' || *ep == '?' || *ep == '-') {  /* accept empty? */
            p = ep + 1; goto init;  /* return match(ms, s, ep + 1); */
          } else { /* '+' or no suffix */
            s = NULL;  /* fail */
          }
        } else {
        
           .. 省略若干代码，用于处理匹配单独字符成功的处理逻辑 ..
        
        }
        
        break;
      }    
    }
  }
  ms->matchdepth++;
  return s;
}
```

* 对于单独字符的匹配，Lua/Luajit的正则引擎提供了3种支持:
   * 原始匹配
   * ```[]```字符组支持
   * ```%```转义支持，具体支持的特殊转义字符见下
   
```
%a: represents all letters.
%c: represents all control characters.
%d: represents all digits.
%g: represents all printable characters except space.
%l: represents all lowercase letters.
%p: represents all punctuation characters.
%s: represents all space characters.
%u: represents all uppercase letters.
%w: represents all alphanumeric characters.
%x: represents all hexadecimal digits.
%x: (where x is any non-alphanumeric character) represents the character x.
```

* singlematch()函数是匹配单独字符的主体逻辑，在讲解其这前，可以先观察匹配失败的处理：
   * 当匹配失败后，若该字符由容忍0次量词修饰（```*、?、-```），则正则表达式从该量词后继续调用match()进行匹配
   * 当匹配失败后，若没有容忍0次量词修改，出错返回，顶层传动装置开始工作。

   
```
static int singlematch (MatchState *ms, const char *s, const char *p,
                        const char *ep) {
  if (s >= ms->src_end)
    return 0;
  else {
    int c = uchar(*s);
    switch (*p) {
      case '.': return 1;  /* matches any char */
      case L_ESC: return match_class(c, uchar(*(p+1)));
      case '[': return matchbracketclass(c, p, ep-1);
      default:  return (uchar(*p) == c);
    }
  }
}
```
* 从signlematch()对单独字符匹配处理逻辑来看：
   * ```.```永远匹配任意字符 
   * ```%转义```调用match_class()进行处理
   * ```[字符组]```调用matchbracketclass()进行处理
   * ```原始字符```直接比较

```
static int match_class (int c, int cl) {
  int res;
  switch (tolower(cl)) {
    case 'a' : res = isalpha(c); break;
    case 'c' : res = iscntrl(c); break;
    case 'd' : res = isdigit(c); break;
    case 'g' : res = isgraph(c); break;
    case 'l' : res = islower(c); break;
    case 'p' : res = ispunct(c); break;
    case 's' : res = isspace(c); break;
    case 'u' : res = isupper(c); break;
    case 'w' : res = isalnum(c); break;
    case 'x' : res = isxdigit(c); break;
    case 'z' : res = (c == 0); break;  /* deprecated option */
    default: return (cl == c);
  }
  return (islower(cl) ? res : !res);
}
```
* 从match_class()对```%转义字符```的处理来看，是直接调用c库进行特殊字符范围的判定。

```
static int matchbracketclass (int c, const char *p, const char *ec) {
  int sig = 1;
  if (*(p+1) == '^') {
    sig = 0;
    p++;  /* skip the '^' */
  }
  while (++p < ec) {
    if (*p == L_ESC) {
      p++;
      if (match_class(c, uchar(*p)))
        return sig;
    }
    else if ((*(p+1) == '-') && (p+2 < ec)) {
      p+=2;
      if (uchar(*(p-2)) <= c && c <= uchar(*p))
        return sig;
    }
    else if (uchar(*p) == c) return sig;
  }
  return !sig;
}
```
* 从matchbracketclass()对```[字符组]```的处理来看：
   * ```^```支持排除型字符组
   * ```[]```字符组内仍支持```%转义字符```
   * ```[-]```字符组对范围的支持，是基于ASCII。

### 量词支持
```
static const char *match (MatchState *ms, const char *s, const char *p) {
  if (ms->matchdepth-- == 0)
    luaL_error(ms->L, "pattern too complex");
  init: /* using goto's to optimize tail recursion */
  if (p != ms->p_end) {  /* end of pattern? */
    switch (*p) {
      default: dflt: {
        const char *ep = classend(ms, p);  /* points to optional suffix */
        /* does not match at least once? */
        if (!singlematch(ms, s, p, ep)) {
          if (*ep == '*' || *ep == '?' || *ep == '-') {  /* accept empty? */
            p = ep + 1; goto init;  /* return match(ms, s, ep + 1); */
          } else { /* '+' or no suffix */
            s = NULL;  /* fail */
          }
        } else {
          switch (*ep) {  /* handle optional suffix */
            case '?': {  /* optional */
              const char *res;
              if ((res = match(ms, s + 1, ep + 1)) != NULL)
                s = res;
              else {
                p = ep + 1; goto init; 
              }
              break;
            }
            case '+':  /* 1 or more repetitions */
              s++;  /* 1 match already done */
              /* FALLTHROUGH */
            case '*':  /* 0 or more repetitions */
              s = max_expand(ms, s, p, ep);
              break;
            case '-':  /* 0 or more repetitions (minimum) */
              s = min_expand(ms, s, p, ep);
              break;
            default:  /* no suffix */
              s++; p = ep; goto init;  /* return match(ms, s + 1, ep); */
          }
        }
        break;
      }    
    }
  }
  ms->matchdepth++;
  return s;
}
```

* Lua/LuaJit共支持4种量词，分别是：
   * ```?``` : 控制前导元素出现0次或1次 
   * ```+``` : 控制前导元素出现1次或多次 
   * ```*``` : 控制前导元素出现0次或多次（贪婪模式，也叫最长匹配模式）
   * ```-``` : 控制前导元素出现0次或多次（懒惰模式，也叫最短匹配模式）
  
* 当单独字符匹配成功后，会查看该字符后是否有量词出现：
   * 若单独字符后为```?```，则调用match()尝试对下一个字符进行匹配，即以前导元素出现1次进行尝试
      * 若尝试成功，则记录匹配结果并返回
      * 若尝试失败，则回溯至调用前的状态，即正则引擎会以前导元素出现0次继续进行匹配。
      * ***这是第1次看到正则引擎的 尝试 - 回溯 机制，```?```至多进行1次回溯，但其它量词、特别是多个量词叠加，存在引起指数级回溯的很大风险，将CPU Hang住***
   * 若单独字符后为```+```，则继续对下一个字符进行最长匹配
   * 若单独字符后为```*```，则调用max_expand()进行最长匹配
   * 若单独字符后为```-```，则调用min_expand()进行最短匹配
   * 若单独字符后无量词出现，则继续向后匹配。