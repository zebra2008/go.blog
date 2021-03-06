这个scheme语言实现中，所有的东西都是一个对象，这就是它的类型系统。

一个对象可能是一个立即值也可能是一个sexp。由于通过各种tag识别出一个对象真实的类型。

现在只要明白，32位机器上，对象就是一个32位的黑箱。

要么这32位本身存了它的真实数据，这种就是立即值。

要么这是一个指针，这种就是一个sexp。

## 第一部分:immediate(立即值)

立即值包括整数，char，唯一的立即符号等。

我们知道，一个字节是8位，所以一个指针的最后三个bit一定是0。立即数就是利用了这一点，将这三位作为tag位。

一个对象，也就是这个32位的东西，先识别其最后3个bit，如果不是全0，它就是一个立即值，否则它是一个指针。

文件 ./include/chibi/sexp.h 中的注释

```C
/* tagging system
 *   bits end in    00:  pointer
 *                  01:  fixnum
 *                 011:  immediate flonum (optional)
 *                 111:  immediate symbol (optional)
 *              000110:  char
 *              001110:  unique immediate (NULL， TRUE， FALSE)
 */
```

因此可以很容易的知道一个对象是否为立即值

```C
#define SEXP_FIXNUM_MASK 3
#define SEXP_IMMEDIATE_MASK 7
#define SEXP_EXTENDED_MASK 63

#define SEXP_POINTER_TAG 0
#define SEXP_FIXNUM_TAG 1
#define SEXP_ISYMBOL_TAG 7
#define SEXP_CHAR_TAG 6

#define sexp_pointerp(x) (((sexp_uint_t)(x) & SEXP_FIXNUM_MASK) == SEXP_POINTER_TAG)
#define sexp_fixnump(x)  (((sexp_uint_t)(x) & SEXP_FIXNUM_MASK) == SEXP_FIXNUM_TAG)
#define sexp_isymbolp(x) (((sexp_uint_t)(x) & SEXP_IMMEDIATE_MASK) == SEXP_ISYMBOL_TAG)
#define sexp_charp(x)    (((sexp_uint_t)(x) & SEXP_EXTENDED_MASK) == SEXP_CHAR_TAG)
sexp_uint_t就是unsigned int，32位机器上就是一个32位的东西。
```

这里的整型只使用了其中的30位，也就是表示的范围是0-2的30次方减1(如果是无符)，大多数时候30位的数都够用的。

另外，chibi-scheme还实现了大数，可以支持更大的数，具体可以去看源代码，本文不做具体分析。

## 第二部分:sexp(s表达式)

如果一个对象不是立即值(最后3bit不为0)，它就是一个sexp。

typedef struct sexp_struct *sexp; sexp_struct是由一个头部加上一个union的各种类型的值组成的。头部tag字段用来确定类型，markedp等等字段是为垃圾回收设置的。

```C
struct sexp_struct {
  sexp_tag_t tag;
  char markedp;  
  ......
  union {
    double flonum ;
    struct {
      sexp car， cdr;
      sexp source;
    } pair;
    struct {
      sexp_uint_t length;
      sexp data[];
    } vector;
    ......
    }value;
};
```

除了立即值，所有的chibi-scheme使用的类型都被抽象成了sexp，不限于scheme语言所定义的基本数据类型，还包括了运行时类型如堆，上下文，还有ast树的结构 如SEXP_CND， SEXP_REF， SEXP_SET， SEXP_SEQ， SEXP_LIT，等都是sexp。

sexp_tag_t的那个tag可取值为下面的枚举:

```C
enum sexp_types {
  SEXP_OBJECT，
  SEXP_TYPE，
  SEXP_FIXNUM，
  SEXP_NUMBER，
  SEXP_CHAR，
  SEXP_BOOLEAN，
  ......
  SEXP_IPORT，
  SEXP_OPORT，
  SEXP_EXCEPTION，
  SEXP_PROCEDURE，
  SEXP_MACRO，
  SEXP_SYNCLO，
  SEXP_ENV，
  SEXP_BYTECODE，
  SEXP_CORE，
  SEXP_OPCODE，
  SEXP_LAMBDA，
  SEXP_CND，
  SEXP_REF，
  SEXP_SET，
  SEXP_SEQ，
  SEXP_LIT，
  SEXP_STACK，
  SEXP_CONTEXT，
  SEXP_CPOINTER，
  SEXP_NUM_CORE_TYPES
};
```

## 第三部分:sexp相关操作

为了更方便在操作，./include/chibi/sexp.h中提供了许多的宏，提供类型的谓词判断，空间分配，各种类型各个域成员的访问宏。

有些复杂一些的在sexp.h中只是一个函数声明，实现放在sexp.c和gc.c中。

还有一些是scheme对象的算术运算宏，以及各类型大小计算的宏。例如下面是其中一部分源代码片断:

```C
#define sexp_check_tag(x，t)  (sexp_pointerp(x) && (sexp_pointer_tag(x) == (t)))
#define sexp_typep(x)       (sexp_check_tag(x， SEXP_TYPE))
#define sexp_pairp(x)       (sexp_check_tag(x， SEXP_PAIR))
#define sexp_stringp(x)     (sexp_check_tag(x， SEXP_STRING))
...
#define sexp_field(x， type， id， field) ((x)->value.type.field)
#define sexp_pred_field(x， type， pred， field) ((x)->value.type.field)

#define sexp_vector_length(x) (sexp_field(x， vector， SEXP_VECTOR， length))
#define sexp_vector_data(x)   (sexp_field(x， vector， SEXP_VECTOR， data))
...
#define sexp_string_length(x) (sexp_field(x， string， SEXP_STRING， length))
#define sexp_string_data(x)   (sexp_field(x， string， SEXP_STRING， data))
...
```

注意到sexp的数据部分是用union定义的，union的占用的内存大小是其中最大的成员小大小。这样岂不会造成很大的内存浪费?

后面我注意到，在实际的内存分配时不是分配sizeof(union)的大小，而是对每种类型分配其真实大小。

```C
#define sexp_sizeof(x) (offsetof(struct sexp_struct， value) \
                         + sizeof(((sexp)0)->value.x))
```

比如 sexpmakeflonum 创建一个浮点数对象，其调用链是这样的:

```
sexp_make_flonum(sexp ctx ， double f)

==> sexp_alloc_type(ctx，flonum，SEXP_FLONUM)

==> sexp_alloc_tagged(ctx，sexp_sizeof(flonum)，SEXP_FLONUM)

==> sexp_alloc_tagged_aux

==> sexp_alloc(ctx，size)
```

以上各函数的声明都在sexp.h中，`sexp_alloc` 函数的实现在文件 gc.c 中，其作用是在上下文中分配size字节大小的空间。

## 第四部分：类型对象

scheme是动态类型语言，这里所谓"动态"，按我的理解是符号本身是没有类型的，但符号绑定的值是有类型的。一个符号可以绑定到任意类型的值，所以是动态的。这里可能用英语单词更精确，我所说的符号，更倾向于是identifier，或者symbol，甚至就叫name，但不是指variable。

`(define a 100) ;;` a是一个符号，符号绑定到整型的值 100 `(define a "hello world") ;;` a 被绑定到一个字符串，注意，a 本身是不具有类型的 值是有类型的，100 这个值是整型。“hello world” 这个值是一个字符串。仅仅一个 tag 域，想要获得全部的运行时类型信息显然是不够的。比如说，我想知道一个对象占用的真实字节是多少。我很容易通过宏 `sexppointertag()` 得到对象的 tag。但想使用宏 `sexpsizeof(tag)` 得到大小是会报错的，因为参数需要是一个类型比如 flonum，但 tag 只是一个编号，放在这个具体情况，tag 只是代表 flonum 类型的编号。

很自然地联想到，需要一张全局的表，将类型编号和类型的真实信息联系起来。

这张表中的对象，就是类型对象，类型对象用来记录每种类型的具体信息。chibi-scheme中，这个设计确实存在。

sexp的union其中的一个是type对象：

```C
struct sexp_struct{
    ...
    union {
        ...
        struct sexp_type_struct type;
        ...
    }values;
};
```

结构体sexp_type_struct就是用来记录各种类型的信息的：

```C
struct sexp_type_struct {
  sexp_tag_t tag;
  short field_base， field_eq_len_base， field_len_base， field_len_off;
  unsigned short field_len_scale;
  short size_base， size_off;
  unsigned short size_scale;
  short weak_base， weak_len_base， weak_len_off， weak_len_scale， weak_len_extra;
  short depth;
  sexp name， cpl， slots， dl， id， print;
  sexp_proc2 finalize;
};
```

这张表也是真实存在的，它其实是context->globals[SEXP_G_TYPES]。这里姑且简单的使用table来表示。

举个的例子，假设有一个对象x，先通过sexp_pointer_tag(x)得到其tag，再通过table[tag]就可查到x的类型对象了。

再对这个类型对象进行操作就可以知道x的全部类型信息，这些操作都是封装好了的。

```C
#define sexp_object_type(ctx，x)        (sexp_type_by_index(ctx， ((x)->tag)))
#define sexp_type_by_index(ctx，i)  (sexp_context_types(ctx)[i])
#define sexp_context_types(ctx)    sexp_vector_data(sexp_global(ctx， SEXP_G_TYPES))
#define sexp_global(ctx，x)      (sexp_vector_data(sexp_context_globals(ctx))[x])
```

## 小结

本篇讲解了chibi-scheme的类型系统。主要代码对应于文件./include/chibi/sexp.h

类型系统非常重要，所有的scheme对象在本质上就是一个32位的黑箱，它或者是一个立即值，或者是一个s表达式。

立即值中的一些bits可作为标识区别出是哪种立即值，而s表达式则是一个指向结构体的指针，结构体头部含有的tag决定其类型。

sexp.h中提供的宏为操作s表达式提供了非常好的支持，在后面的程序中可以很方便地使用，它同时也是整个程序都要include的头文件。

这些操作具体包括了类型的判断谓词，大小，各个类型各个域的访问宏，各种类型的创建等等。

仅仅通过tag获取全部运行时类型信息是不够的。对应的，有类型对象是专门来存各种各样的类型信息的，包括类型名称，类型占用字节数，类型的对齐方式，类型在gc是否调用某个析构函数等等。还有一张表，表中存的各种类型对象。有了这张表，就可以把tag和类型对象关联起来了。
