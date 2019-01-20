---
layout: post
title:  "Type erasure or how I implemented a simple container without having an idea what's type erasure"
date:   2019-01-20 20:28:00 +0100
categories: programming notes
---
At the beginning I want tell you that I don't expect very proffesional code here. Rather it is working example
how we can achieve a simple safe container which hold static storage overwritten by random data.
A small remind how I actually understand a type erasure idea.
{% highlight c++ %}
class var {
public:
    template <typename T> var(T src) : m_inner(new inner<T>(std::forward<T>(src)))
    {}

    template <typename T> var& operator=(T src) {
        m_inner = std::make_unique<inner<T>>(std::forward<T>(src));
        return *this;
    }

    class inner_base {
    public:
        using ptr = std::unique_ptr<inner_base>;
    };

    template <typename T> class inner : inner_base {
    public:
        inner(T new_val) : m_value(new_val) {}
    private:
        T m_value;
    };
private:
    inner_base::ptr m_inner;
};
{% endhighlight %}
The idea behind presented implementation is to hide type of assigned variable in inner class. And using inheritance
get an access to base pointer which hold variable data. With that we are trying to get rid of instantiation main var class
with certain type. To do that the inner storage class is used and templarized **operator=** where we store a new variable data.
The problem come when we want access to compile *unknown* type. And replace it and .... retreive it thorough pointer.
That coerce type earase to live.
---
Here I want to paste my own version of **any** implementation without use of **virtual** functions.
{% highlight c++ %}
namespace stl {

using Buffer = std::aligned_storage_t<3*sizeof(void*), std::alignment_of<void*>::value>;

class any {
public:
    explicit any() noexcept;
    template <typename T,
        typename = std::enable_if_t<sizeof(T) <= sizeof(Buffer)>
    >
    any(T&& value);

    ~any();

    template <typename T,
        typename = std::enable_if_t<sizeof(T) <= sizeof(Buffer)>
    >
    void replace(T&& value);

    bool has_value() noexcept;

    void reset() noexcept;

    template <typename T>
    friend T& any_cast(any& value);

private:
    union Storage {
        explicit Storage() : m_ptr(nullptr)
        {}

        Buffer m_buffer;
        void* m_ptr;
    };

    Storage m_storage;
    std::function<void()> m_deallocate;
};

any::any() noexcept
{}

template <typename T, typename>
any::any(T&& value)
{
    ::new(&m_storage.m_buffer) T(std::forward<T>(value));
    m_deallocate = [this](){
        reinterpret_cast<T&>(m_storage.m_buffer).~T();
    };
}

any::~any()
{
    reset();
}

bool any::has_value() noexcept
{
    return m_storage.m_ptr != nullptr;
}

void any::reset() noexcept
{
    if (m_storage.m_ptr != nullptr)
    {
        m_deallocate();
        m_storage.m_ptr = nullptr;
    }
}

template <typename T, typename>
void any::replace(T&& value)
{
    ::new(&m_storage.m_buffer) T(std::forward<T>(value));
    m_deallocate = [this](){
        reinterpret_cast<T&>(m_storage.m_buffer).~T();
    };
}

template <typename T>
T& any_cast(any& value)
{
    return reinterpret_cast<T&>(value.m_storage);
}

}
{% endhighlight %}

What was done here:
- The main any class includes union storage type, which is size of 3 void pointers and it is controlled by pointer to void
which control if union is empty or not.
- With replacing value we recreate helper function to deallocate stored type.

What can be done here:
- The another task is to create function which will be able to facilitate any_cast by persisting current type and comparing
to demanded type.
- Also checking type could be important here.

Helpful type traits which I discovered during that exercise:
- *std::type_identity*
- *std::type_info*

The basic concept is to use given type during an initialization and using **type_info** store its representation to compare it later.