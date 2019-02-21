---
layout: post
title:  "Trying to implement same STL algorithm in different manners"
date:   2019-02-21 20:28:00 +0100
categories: programming notes
---

This time I decided to take first served algorithm and reimplement it on my own. 
Measure performance and share my results in the following note.

The algorithm which I want to talk about is **any_of** (checks if unary predicate is true at least for one element).
Let's start from most ordinary implementation:

{% highlight c++ %}
template <typename Iterator, typename UnaryPredicateFunction>
bool any_of(Iterator begin, Iterator end, UnaryPredicateFunction predicate) {
    for (; begin != end; ++begin) {
        if (predicate(*begin)) return true;
    }
    return false;
}
{% endhighlight %}

There is nothing bad with that implementation however I wanted somehow specify *folding* concept.
Generally that's not best example for folding ("reduction") as we have no any temporary variables here. The whole concept about
processing the whole collection one by one and reduce it using external *predicate* function into single value or statement.
What correspond to **std::acumulate** algorithm. It also promotes the use of specialized highly composable functions. 
For the last reason I decided to try recursive solution and look closer for the aesthetic and performance value alike.

{% highlight c++ %}
template <typename Iterator, typename UnaryPredicateFunction>
bool any_of(Iterator begin, Iterator end, UnaryPredicateFunction predicate) {
    if (predicate(*begin)) return true;
    else if (begin != end) return any_of(++begin, end, predicate);
    else return false;
}
{% endhighlight %}

Now we entirely get rid of loops here. We have to pay attention here, to properly implement recursive conditions to not
get into endless loop. So whenever we meet first condition that will be enough to leave recursive calls. Recursion give
us ability for creation function which reuse its own functionality however it carry on a risk of performance lose. 
So it always good to know the term *tail recursion optimanization*. In my case there is no option for it as compiler
wasn't able to extract subsequent recursive calls into one function.

In the next step I decided to implement my first parallel approach which appears later completly wrong idea :(

{% highlight c++ %}
template <typename Iterator, typename UnaryPredicateFunction>
bool any_of(Iterator begin, Iterator end, UnaryPredicateFunction predicate) {
    const int threads_num = std::thread::hardware_concurrency();
    std::thread pool[threads_num];
    bool results[threads_num];
    while (begin != end) {
        int counter = threads_num;
        while (counter-- && begin != end) {
            pool[counter] = std::thread([&results, &predicate](Iterator iter, int indx) {
                results[indx] = predicate(*iter);
            }, begin, counter);
            ++begin;
        }

        std::for_each(pool, pool + threads_num, [](auto&& thread) {
            if (thread.joinable()) thread.join();
        });

        for (int i = 0; i < counter; ++i) {
            if (results[i]) return true;
        }
    }

    return false;
}
{% endhighlight %}

Here I tried to create a thread pool and create and execute predicate for every iterator. After all I shared results into
another array and check whenever it fullfil final end requirement. Anyway after checking some measurements I find out that
creating a lot of threads even without synchronized memory is very very heavy. Thus I decided to reuse my thread pool and
implement something better.

{% highlight c++ %}
template <typename Iterator, typename UnaryPredicateFunction>
bool any_of(Iterator begin, Iterator end, UnaryPredicateFunction predicate) {
    if (begin != end && predicate(*begin)) return true;

    const int threads_num = std::thread::hardware_concurrency();
    std::mutex mut;
    std::thread pool[threads_num];
    bool result = false;
    std::for_each(pool, pool + threads_num, [&](auto& thread) {
        thread = std::thread([&begin, &end, &predicate, &result, &mut]() {
            Iterator temp = begin;
            do {
                if (begin == end) break;
                mut.lock();
                temp = ++begin;
                mut.unlock();
                if (temp == end) break;
                if (predicate(*temp)) {
                    result = true;
                    mut.lock();
                    begin = end;
                    mut.unlock();
                }
            } while (true);
        });
        thread.join();
    });

    return result;
}
{% endhighlight %}

I introduce some mutexes and try a new idea in reality. It gives to me what I expected a huge performance improvement
comparing to the previous solution *but* it is still slower then my single core solution. Below I put results for *10000* iterations:

```
Reulsts:
~~~any_of using folding~~~
start: 0
stop: 94
~~~any_of using recursion~~~
start: 0
stop: 131
~~~any_of using parallel <<- that's for only 1 iteration!!!~~~
start: 0
stop: 111
~~~any_of using parallel improvement~~~
start: 0
stop: 1551
```

----
###### Now it is a time for final conclusions :)
I believed blindly in parallelism however I also just started to investigate that topic and was interested in trying it by myself.
As there is already parallel stl algorithms implementation added to C++17. So the rudimentary question was how efficient is 
to use concurrent algorithms. Now I think, that's wide topic which I have to learn more. But we can see it is multithread
program performed worse due to the overhead of thread creation. So for sure it is better to reuse old one if we have kind of
continuous work to do. The next thing, all the staff like synchronizing memory on shared variables may not benefit concurrent
programs. If we dig deeper we have to deal with context switching and probably more. That all makes a feeling that I have to
use a heavy predicate operation here to achieve any privileges of using threads. But yet how heavy must be that calculation
that's a future question.

Thanks for reading this. If you read this post until the end :)
