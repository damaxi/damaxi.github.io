---
layout: post
title:  "Print last line comparing solutions"
date:   2019-03-11 214:28:00 +0100
categories: programming notes
---

I'd like to introduce in that post two approaches for
{% highlight shell %}
tail -n 20
{% endhighlight %}
bash function - *print last n lines for given file*.

In first approach maybe more common. I follow some hints, where we have to implement kind of circular buffer. Aftewards read the file, line by line and feed our fixed circular buffer with following lines.

{% highlight c++ %}
template <typename T>
class circular_buffer
{
public:
    circular_buffer(std::size_t size) :
        m_max_size(size),
        m_buffer(m_max_size)
    {}

    void push_back(T item)
    {
        if (m_buffer.size() == m_max_size)
        {
            m_buffer.pop_front();
        }

        m_buffer.push_back(item);
    }

    using const_iterator = typename std::list<T>::const_iterator;
    using value_type = T;

    const_iterator begin() const
    {
        return m_buffer.begin();
    }

    const_iterator end() const
    {
        return m_buffer.end();
    }

private:
    std::size_t m_max_size;
    std::list<T> m_buffer;
};

struct line 
{
    std::string m_line;
    operator std::string const& () const { return m_line; }
    friend std::istream& operator>>(std::istream& stream, line& l)
    {
        return std::getline(stream, l.m_line);
    }
};

void print_last_lines_using_circular_buffer(std::ifstream& stream, int lines)
{
    circular_buffer<std::string> buffer(lines);

    std::copy(std::istream_iterator<line>(stream),
              std::istream_iterator<line>(),
              std::back_inserter(buffer));

    std::copy(buffer.begin(), buffer.end(),
            std::ostream_iterator<std::string>(std::cout));
}
{% endhighlight %}

Thanks for stack [same implementation](https://stackoverflow.com/questions/5794945/print-out-the-last-10-lines-of-a-file), I enrich my code with istream_iterator. Now it looks more neat and functional.

But.... Why do we need to read a whole file just to print a few lines from the end? I just started to think if I can improve performance a little more here. I wind thorough some ideas like counting new lines when unexpectedly found something: *ate seek to the end of stream immediately after open*.
It appears that we can implement it a little bit differently. First jump to the end and then start to remmember line by line until the n-last buffer will be filled. Here how it comes:
{% highlight c++ %}
std::ifstream hndl(filename, std::ios::in | std::ios::ate);

...

void print_last_lines(std::ifstream& stream, int lines)
{
    std::stack<char> last_lines; 
    for (int position = stream.tellg(); position >= 0; )
    {
        stream.seekg(--position);
        char single_char = stream.peek();
        if (single_char == '\n' && --lines == 0) break;
        last_lines.push(single_char);
    }

    while (!last_lines.empty())
    {
        std::cout << last_lines.top();
        last_lines.pop();
    }
}
{% endhighlight %}

Finally I tested it on huge text file (45K and 2048 lines). Comparing both solutions, the backward file iteration is more performent (3-5x).

---
##### Final thinking
I tried to check also both solutions for very huge file like more than 1Gb. And what suprise me, *ate* mode is lagging here. So the question is can we take file handler and then read the size of file from its attributes and use it to calculate end of file?
Also I am curious if someone implement parallel file look up. :)