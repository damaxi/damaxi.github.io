---
layout: post
title:  "Logging exercise"
date:   2018-12-18 09:12:00 +0100
categories: programming notes
---
Last time I have found a **C** language pattern for logging which roughly looks like that.
For given sample external interface:

{% highlight c++ %}
extern "C"
{
    static bool foo(bool magic_parameter) {
        if (magic_parameter) return true;

        return false;
    }

    static void boo(bool magic_parameter) {
        (void)magic_parameter;
    }
}
{% endhighlight %}

We create a caller function which leaves a trail if a callee function fails.
{% highlight c++ %}
void execute_foo()
{
    if (!foo(true))
        std::cout << "Foo error";
}
{% endhighlight %}

Additionally we can forward result of *foo* to be handled from outside of *execute_foo*. That becomes a little bit cumbersome in view of many external api function calls. Each declared function *bool(...)* must be handled with appropriate error message and forward or not its return type, error code etc.

For that reason I created a small snippet which can be extended in the future.

{% highlight c++ %}
template <typename Func, typename ...Params,
    typename = std::enable_if_t<
        std::is_function<std::remove_pointer_t<Func>>::value &&
        std::is_same<std::invoke_result_t<Func, Params...>, bool>::value
    >
>
void execute_with_custom_error(Func f, std::string&& error_message, Params ...p)
{
    static_assert(std::is_function<std::remove_pointer_t<Func>>::value &&
    std::is_same<std::invoke_result_t<Func, Params...>, bool>::value,
    "Only function which return bool can fail");

    if (!f(p...)) {
        std::cerr << error_message;
    }
}
{% endhighlight %}

Now I want to share you my small notes. Instead of creating a new function. We can call our external function within template function with extra error message. To guarantee that passed function would have a proper return type. I check whenever *Func* argument is a function and the result of that function is type of boolean. In addition we have to prepare checked function type for *is_function* template call by removing function pointer type and left only bare function declaration (*bool(bool)*).