---
layout: post
---
This series of blog posts is based on gamozo  FuzzWeek, I wrote this series because I was taking notes for his videos step by step so I thought it would be useful to share it with the community for people who struggle to watch long videos like me and try to simplify his content.

if you don't know his channel which I don't think so if you are here for Fuzzing, you will find really cool stuff about fuzzing, emulation, hypervisors, and exploitation, I think his content is really better than a lot of 4 digits $ trainings in big conferences.  


so let's start Day1.

we started with some motivation that security research is about continuously failing like when you get a target that's too hard or you can't find any bugs or you don't have a way of emulating the target or you can't get the coverage or AFL doesn't work on it or it uses a custom compiler that you can't use existing instrumentation so it is okay to fail.

in this series, he wanted to show how it easy to write your own fuzzer  and how you don't really need to be tied to a tool that's tied to a system (like AFL) because you can quickly make a new fuzzer for a different environment 

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
