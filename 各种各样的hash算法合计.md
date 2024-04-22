# 各种各样的hash算法

## Thomas Wang's 32 Bit / 64 Bit Mix Function
整型的Hash算法使用的是Thomas Wang's 32 Bit / 64 Bit Mix Function ，这是一种基于位移运算的散列方法。基于移位的散列是使用Key值进行移位操作。通常是结合左移和右移。每个移位过程的结果进行累加，最后移位的结果作为最终结果。这种方法的好处是避免了乘法运算，从而提高Hash函数本身的性能。
~~~c++
unsigned int dictIntHashFunction(unsigned int key)
{
    key += ~(key << 15);
    key ^=  (key >> 10);
    key +=  (key << 3);
    key ^=  (key >> 6);
    key += ~(key << 11);
    key ^=  (key >> 16);
    return key;
}
~~~

## MurmurHash算法
字符串使用的MurmurHash算法，MurmurHash算法具有高运算性能，低碰撞率的特点，由Austin Appleby创建于2008年，现已应用到Hadoop、libstdc++、nginx、libmemcached等开源系统。2011年Appleby被Google雇佣，随后Google推出其变种的CityHash算法。
murmur是 multiply and rotate的意思，因为算法的核心就是不断的乘和移位（x *= m; k ^= k >> r;）

~~~c++
unsigned int dictGenHashFunction(const void *key, int len) {
    /* 'm' and 'r' are mixing constants generated offline.
     They're not really 'magic', they just happen to work well.  */
    uint32_t seed = dict_hash_function_seed;
    const uint32_t m = 0x5bd1e995;
    const int r = 24;
 
    /* Initialize the hash to a 'random' value */
    uint32_t h = seed ^ len;
 
    /* Mix 4 bytes at a time into the hash */
    const unsigned char *data = (const unsigned char *)key;
 
    while(len >= 4) {
        uint32_t k = *(uint32_t*)data;
 
        k *= m;
        k ^= k >> r;
        k *= m;
 
        h *= m;
        h ^= k;
 
        data += 4;
        len -= 4;
    }
 
    /* Handle the last few bytes of the input array  */
    switch(len) {
    case 3: h ^= data[2] << 16;
    case 2: h ^= data[1] << 8;
    case 1: h ^= data[0]; h *= m;
    };
 
    /* Do a few final mixes of the hash to ensure the last few
     * bytes are well-incorporated. */
    h ^= h >> 13;
    h *= m;
    h ^= h >> 15;
 
    return (unsigned int)h;
}
~~~


## 加法哈希
加法哈希是通过遍历数据中的元素然后每次对某个初始值进行加操作，其中加的值和这个数据的一个元素相关。
~~~c++
int additiveHash(string key, int prime)
{
    int hash, i;
    for(hash = key.length(), i = 0; i < key.length(); ++i)
    {
        hash += int(key.at(i));
    }
    return (hash%prime);
}
~~~

## 乘法哈希

~~~C
/********************************
 *乘法哈希
 *@key   输入字符串
 *@prime 素数
 ********************************/
int bernstein(string key)
{
    int hash, i;
    for(hash = 0, i = 0; i < key.length(); ++i)
    {
        hash = 33*hash + int(key.at(i));
    }
    return hash;
}
 
/********************************
 *32位FNV算法(乘法)
 *@key   输入字符串
 *@prime 素数
 ********************************/
int M_SHIFT = 0;
int M_MASK = 0x8765fed1;
int FNVHash(string key)
{
    int hash = (int)2166136261L;
    for(int i = 0; i < key.length(); ++i)
    {
        hash = (hash * 16777619)^int(key.at(i));
    }
    if(M_SHIFT == 0)
        return hash;
    return (hash ^ (hash >> M_SHIFT)) & M_MASK;
}
~~~

java中使用的就是这种乘法hash

~~~java
/**
 * JAVA自己带的算法
 */
public static int java(String str) {
    int h = 0;
    int off = 0;
    int len = str.length();
    for (int i = 0; i < len; i++)
    {
        h = 31 * h + str.charAt(off++);
    }
    return h;
}
~~~

## 移位哈希
移位哈希是通过遍历数据中的元素然后每次对初始值进行移位操作。
~~~C
/********************************
 *旋转哈希（移位）
 *@key   输入字符串
 *@prime 素数
 ********************************/
int rotatingHash(string key, int prime)
{
    int hash, i;
    for(hash = key.length(), i = 0; i < key.length(); ++i)
    {
        hash = (hash << 4)^(hash >> 28)^int(key.at(i));
    }
    return (hash%prime);
}
~~~

~~~java
static int hash(int h) {
      // This function ensures that hashCodes that differ only by
      // constant multiples at each bit position have a bounded
      // number of collisions (approximately 8 at default load factor).
      h ^= (h >>> 20) ^ (h >>> 12);
      return h ^ (h >>> 7) ^ (h >>> 4);
  }
~~~

##  PHP中出现的字符串Hash函数

~~~php
static unsigned long hashpjw(char *arKey, unsigned int nKeyLength)
{
　　unsigned long h = 0, g;
　　char *arEnd=arKey+nKeyLength;
　　while (arKey < arEnd) {
    　　h = (h << 4) + *arKey++;
    　　if ((g = (h & 0xF0000000))) {
    　　h = h ^ (g >> 24);
    　　h = h ^ g;
    　　}
　　}
　　return h;
　}
~~~

##  OpenSSL中出现的字符串Hash函数

~~~C
/* The following hash seems to work very well on normal text strings
　　* no collisions on /usr/dict/words and it distributes on %2^n quite
　　* well, not as good as MD5, but still good.
　　*/
　　unsigned long lh_strhash(const char *c)
　　{
	　　unsigned long ret=0;
	　　long n;
	　　unsigned long v;
	　　int r;
	　　if ((c == NULL) || (*c == ”))
	　　    return(ret);
	　　/*
		　　unsigned char b[16];
		　　MD5(c,strlen(c),b);
		　　return(b[0]|(b[1]<<8)|(b[2]<<16)|(b[3]<<24));
	　　*/
	　　n=0×100;
	　　while (*c)
	　　{
		　　v=n|(*c);
		　　n+=0×100;
		　　r= (int)((v>>2)^v)&0×0f;
		　　ret=(ret(32-r));
		　　ret&=0xFFFFFFFFL;
		　　ret^=v*v;
		　　c++;
	　　}
	　　return((ret>>16)^ret);
　　}
~~~

##  MySql中出现的字符串Hash函数
~~~C
#ifndef NEW_HASH_FUNCTION
　　/* Calc hashvalue for a key */
　　static uint calc_hashnr(const byte *key,uint length)
　　{
	　　register uint nr=1, nr2=4;
	　　while (length–)
	　　{
		　　nr^= (((nr & 63)+nr2)*((uint) (uchar) *key++))+ (nr << 8);
		　　nr2+=3;
	　　}
	　　return((uint) nr);
　　}
　　/* Calc hashvalue for a key, case indepenently */
　　static uint calc_hashnr_caseup(const byte *key,uint length)
　　{
	　　register uint nr=1, nr2=4;
	　　while (length–)
	　　{
		　　nr^= (((nr & 63)+nr2)*((uint) (uchar) toupper(*key++)))+ (nr << 8);
		　　nr2+=3;
	　　}
	　　return((uint) nr);
　　}
　　#else
　　/*
　　* Fowler/Noll/Vo hash
　　*
　　* The basis of the hash algorithm was taken from an idea sent by email to the
　　* IEEE Posix P1003.2 mailing list from Phong Vo ( kpv at research.att.com) and
　　* Glenn Fowler ( gsf at research.att.com). Landon Curt Noll ( chongo at toad.com)
　　* later improved on their algorithm.
　　*
　　* The magic is in the interesting relationship between the special prime
　　* 16777619 (2^24 + 403) and 2^32 and 2^8.
　　*
　　* This hash produces the fewest collisions of any function that we’ve seen so
　　* far, and works well on both numbers and strings.
　　*/
　　uint calc_hashnr(const byte *key, uint len)
　　{
	　　const byte *end=key+len;
	　　uint hash;
	　　for (hash = 0; key < end; key++)
	　　{
		　　hash *= 16777619;
		　　hash ^= (uint) *(uchar*) key;
	　　}
	　　return (hash);
　　}
　　uint calc_hashnr_caseup(const byte *key, uint len)
　　{
	　　const byte *end=key+len;
	　　uint hash;
	　　for (hash = 0; key < end; key++)
	　　{
		　　hash *= 16777619;
		　　hash ^= (uint) (uchar) toupper(*key);
	　　}
	　　return (hash);
　　}
　　#endif
~~~
