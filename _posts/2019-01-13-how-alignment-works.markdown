---
layout: post
title:  "How alignment works"
date:   2019-01-13 09:12:00 +0100
categories: programming notes
---
I always wanted to find a time to satisfy my curiosity about how alignment works. Previously I knew only a few
general details and listened my helpmate but I never fully understand that topic.

One of the fundamental questions during recruitment process is how much memory **X** structure takes?
Let's analyze some examples:

{% highlight c++ %}
struct A
{
    int a;
    char b;
};
{% endhighlight %}

**sizeof** **int** is 4 and **char** 1. Each type is *region of storage* that has alignment requirement which can be checked with
**alignof**. The minimum alignment for **POD** type is its minimum size so for **int** is 4 and for **char** will be 1 respectively.
Now we need remark some important rules for object alignment calculation:
- The alignment of structure equals to the highest member alignment. In particular example it is alignment of int.
- The whole structure size must be divided by its alignment.
- Compiler analyze compound type member by member if the next member has bigger alignment than previous then actual size
must be extended by specific number of bytes to fill requirement *max member alignment divisibility*
- Empty struct/class has 0 size with alignment 1

And that's basically all what we have to know about alignment calculation. Looking back on our example the size equation
should look like this: **4 + 1 + ( 3 padding ) = 8**

Now lets look for next examples:

{% highlight c++ %}
struct B
{
    int a;
    char b;
    char c;
};
{% endhighlight %}

- **4 + 1 + 1 + ( 2 padding ) = 8**

{% highlight c++ %}
struct C
{
    char c;
    int a;
    char b;
};
{% endhighlight %}

- **1 + ( 3 padding ) + 4 + 1 + ( 3 padding ) = 12**

{% highlight c++ %}
struct D
{
    short d;
    char c;
    int a;
    char b;
};
{% endhighlight %}

- **2 + 1 + ( 1 padding ) + 4 + 1 + ( 3 padding ) = 12**

{% highlight c++ %}
struct E
{};
{% endhighlight %}

- **0 + ( 1 padding ) = 1**

{% highlight c++ %}
struct alignas(8) F : D
{};
{% endhighlight %}

- **12 + ( 4 padding ) = 16**

{% highlight c++ %}
struct G
{
    B b;
    char a;
};
{% endhighlight %}

- **8 + 1 + ( 3 padding ) = 12**

{% highlight c++ %}
struct H
{
    char c;
    int a;
    char b;
} __attribute__ ((__packed__));
{% endhighlight %}

- **1 + 4 + 1 = 6**

**packed** enforces alignment to be equal 1.

Now it can make you confused. Why do we need alignment at all? There are a few very important ones:
- Access to memory is very expensive, modern processors have multilevel memories through data must be pulled in.
For that reason CPU is trying to minimalize number of data fetching.
So in example above for fetched 4 bytes word size. During fetching **int a** member CPU must fetch it twice and shift
data of each part, merging all into final result register. What's quite unefficient.
*The question why do we need to fetch data using word fixed address I left here as it is more complex*
For that reason *unaligned address access* is less efficient and less memory consuming what can have benefits in network
packet transmission.

A dumb (not perfect) example:

{% highlight c++ %}
    C c;
    H h;
    std::cout << c.b << std::endl;
    /*
        movabs  rdi, offset std::cout
        movsx   esi, byte ptr [rbp - 56]
        mov     qword ptr [rbp - 144], rax # 8-byte Spill
        call    std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char)
        mov     rdi, rax
        movabs  rsi, offset std::basic_ostream<char, std::char_traits<char> >& std::endl<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&)
        call    std::basic_ostream<char, std::char_traits<char> >::operator<<(std::basic_ostream<char, std::char_traits<char> >& (*)(std::basic_ostream<char, std::char_traits<char> >&))
    */
    std::cout << h.b << std::endl;
    /*
        movabs  rdi, offset std::cout
        movsx   esi, byte ptr [rbp - 67]
        mov     qword ptr [rbp - 152], rax # 8-byte Spill
        call    std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char)
        mov     rdi, rax
        movabs  rsi, offset std::basic_ostream<char, std::char_traits<char> >& std::endl<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&)
        call    std::basic_ostream<char, std::char_traits<char> >::operator<<(std::basic_ostream<char, std::char_traits<char> >& (*)(std::basic_ostream<char, std::char_traits<char> >&))
        xor     ecx, ecx
        mov     qword ptr [rbp - 160], rax # 8-byte Spill
    */
{% endhighlight %}

Some additional informations:
=============================
- the maximum available alignment could be determined by **std::max_align_t**
- in C++ we can create also a certain type with custom size and alignment.
Below I put small snippet:

{% highlight c++ %}
template <std::size_t Len, std::size_t Align>
struct aligned_storage {
    struct type {
        alignas(Align) unsigned char data[Len];
    };
};
{% endhighlight %}

*Additional resources:*
-----------------------
[A nice explanation why do we need alignment at all](https://www.ibm.com/developerworks/library/pa-dalign/)
[A future topic about memory performance impact](http://igoro.com/archive/gallery-of-processor-cache-effects/)
[Russian article about concurrency and memory alignment](https://habr.com/post/195948/)