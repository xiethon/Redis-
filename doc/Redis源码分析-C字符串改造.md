# 《Redis部分源码分析-C字符串改造》

  Redis没有直接使用C语言传统的字符串表示（以空字符结尾的字符数组，以下简称C字符串），而是自己构建了一种名为简单动态字符串（simple dynamic string，SDS）的抽象类型，并将SDS用作Redis的默认字符串表示。

## 1. Redis字符串定义  
```C  
	struct sdshdr{  
		int len;  //buf中已使用的字节数量  
		int free;  //未使用的字节数量  
		char buf[];  //字节数组，用于保存字符串  
	}；  
```  
图1中展示了一个SDS示例  
！[](phots/SDS字符串.png)  

* free属性的值为0，表示这个SDS没有分配任何未使用空间。  
* len属性的值为5，表示这个SDS保存了一个五字节长的字符串。  
* buf属性是一个char类型的数组，数组的前五个字节分别保存了'R'、'e'、'd'、 'i'、's'五个字符，而最后一个字节则保存了空字符'\0'。  

SDS遵循C字符串以空字符结尾的惯例，保存空字符的1字节空间不计算在SDS的len属性里面，并且为空字符分配额外的1字节空间，以及添加空字符到字符串末尾等操作，都是由SDS函数自动完成的，所以这个空字符对于SDS的使用者来说是完全透明的。遵循空字符结尾这一惯例的好处是，SDS可以直接重用一部分C字符串函数库里面的函数  

## 2. SDS与C字符串的区别 
### 2.1 对求字符串长度(strlen)的改造
图2中展示了标准C库计算一个C字符串长度的过程。
！[](phots/C字符串长度.png)  

和C字符串不同，因为SDS在len属性中记录了SDS本身的长度，所以获取一个SDS长度的复杂度仅为O(1)。 
sdslen实现相关源码如下(sds/sds.h)  
```C
	/* 计算sds的长度，返回的size_t类型的数值 */
	/* size_t,它是一个与机器相关的unsigned类型，其大小足以保证存储内存中对象的大小。 */
	static inline size_t sdslen(const sds s) 
	{
    	struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    	return sh->len;
	}
```

### 2.2 杜绝缓冲溢出
因为C字符串不记录自身的长度，所以strcat假定用户在执行这个函数时，已经为dest分配了足够多的内存，可以容纳src字符串中的所有内容，而一旦这个假定不成立时，就会产生缓冲区溢出。  

与C字符串不同，SDS的空间分配策略完全杜绝了发生缓冲区溢出的可能性：当SDS API需要对SDS进行修改时，API会先检查SDS的空间是否满足修改所需的要求，如果不满足的话，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用SDS既不需要手动修改SDS的空间大小，也不会出现前面所说的缓冲区溢出问题。  

SDS的sdscat在执行拼接操作之前检查s的长度是否足够，在发现s目前的空间不足以拼接字符串之后，sdscat就会先扩展s的空间，然后才执行拼接的操作  

sdscat实现相关源码如下(sds/sds.c)：  
```C
	sds sdscat(sds s, const char *t) 
	{
    	return sdscatlen(s, t, strlen(t));
	}
	sds sdscatlen(sds s, const void *t, size_t len)  
	{  
		struct sdshdr *sh;  
		size_t curlen = sdslen(s);  
		s = sdsMakeRoomFor(s,len);  //为原字符串扩展len长度空间
		if (s == NULL) return NULL;
		sh = (void*) (s-(sizeof(struct sdshdr)));
		memcpy(s+curlen, t, len); 	//多余的数据以t作初始化
		sh->len = curlen+len;//更改相应的len,free值
		sh->free = sh->free-len;
		s[curlen+len] = '\0';
		return s;
	}  
```
##  2.3 减少修改字符串时带来的内存重分配次数
因为内存重分配涉及复杂的算法，并且可能需要执行系统调用，所以它通常是一个比较耗时的操作  
为了避免C字符串的这种缺陷，SDS通过未使用空间解除了字符串长度和底层数组长度之间的关联：在SDS中，buf数组的长度不一定就是字符数量加一，数组里面可以包含未使用的字节，而这些字节的数量就由SDS的free属性记录。  
通过空间预分配策略，Redis可以减少连续执行字符串增长操作所需的内存重分配次数。  
## 2.4 二进制安全
C字符串中的字符必须符合某种编码（比如ASCII），并且除了字符串的末尾之外，字符串里面不能包含空字符，否则最先被程序读入的空字符将被误认为是字符串结尾，这些限制使得C字符串只能保存文本数据，而不能保存像图片、音频、视频、压缩文件这样的二进制数据。如果有一种使用空字符来分割多个单词的特殊数据格式。

## 2.5 总结
图3中展示了SDS与C字符串的区别总结  
！[](C字符串与SDS字符串的区别.png)
图4中展示了SDS主要API  
！[](SDS主要接口函数.png)


# SDS其他函数注释
* sdsMakeRoomFor：字符串扩容
```C
/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for nul term.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
/* 在原有字符串中取得更大的空间，并返回扩展空间后的字符串 */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    struct sdshdr *sh, *newsh;
    //获取当前字符串的可用长度
    size_t free = sdsavail(s);
    size_t len, newlen;

	//如果当前可用空间已经大于需要值，直接返回原字符串
    if (free >= addlen) return s;
    len = sdslen(s);
    sh = (void*) (s-(sizeof(struct sdshdr)));
    //计算要获取新字符串所要的长度大小=原长度+addlen
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);
    if (newsh == NULL) return NULL;

	//新字符串可用空间等于新长度减去原使用的长度
    newsh->free = newlen - len;
    //返回洗字符串中的buf字符串数组
    return newsh->buf;
}
```








