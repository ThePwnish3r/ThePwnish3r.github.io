---
layout: post
---

This series of blog posts is based on <a href="https://twitter.com/gamozolabs">@gamozolabs</a> <a href="https://www.youtube.com/watch?v=2xXt_q3Fex8&list=PLSkhUfcCXvqHsOy2VUxuoAf5m_7c8RqvO">FuzzWeek</a>, I wrote this series because I was taking notes for his videos step by step so I thought it would be useful to share it with the community for people who struggle to watch long videos like me and try to simplify his content.

If you don’t know his channel which I don’t think so if you are here for Fuzzing, you will find really cool stuff about fuzzing, emulation, hypervisors, and exploitation, I think his content is really better than a lot of 4 digits $ trainings in big conferences.






so let's start Day1.

We started with some motivation that security research is about continuously failing like when you get a target that's too hard or you can't find any bugs or you don't have a way of emulating the target or you can't get the coverage or AFL doesn't work on it or it uses a custom compiler that you can't use existing instrumentation so it is okay to fail.

In this series, He wanted to show how it easy to write your own fuzzer and how you don't really need to be tied to a tool that's tied to a system (like AFL) because you can quickly make a new fuzzer for a different environment.



## What is Fuzzing?


<b>Fuzzing</b> or <b>fuzz testing</b> is an automated software testing technique that involves providing invalid, unexpected, or random data as inputs to a computer program. The program is then monitored for exceptions such as crashes, failing built-in code assertions, or potential memory leaks.

{It just run random input through the program}


## objdump from binutils 


Objdump displays information about one or more object files. The options control what particular information to display. This information is mostly useful to programmers who are working on the compilation tools, as opposed to programmers who just want their program to compile and work, it is an easy target and has a lot of bugs so we will be working on it.




#### Building Binutils 
 
 We will be working on this version 

 <a href="https://ftp.gnu.org/gnu/binutils/binutils-2.14.tar.bz2">binutils-2.14.tar.bz2</a>


Run these commands to get it.

{% highlight bash %}
wget https://ftp.gnu.org/gnu/binutils/binutils-2.14.tar.bz2
{% endhighlight %}

{% highlight bash %}
tar xf binutils-2.14.tar.bz2 binutils-2.14
{% endhighlight %}

Then open the directory and run configure script and make to build it.

{% highlight bash %}
./configure
{% endhighlight %}
{% highlight bash %}
make
{% endhighlight %}

We got some errors but we can see that objdump has been built successfully and that is what we want so it does not matter.

BUT! we will have some issues that there is no debugging information because it is considered something new so we want to add them.

So, we rebuild it with DWARF 2.
{% highlight bash %}
make clean
{% endhighlight %}
{% highlight bash %}
CFLAGS="-O0 -g -gdwarf-2" ./configure{% endhighlight %}
{% highlight bash %}
make
{% endhighlight %}

It still doesn't show debugging information due to some linker stuff, we will try to make it works like this.



Now ,we will write a fuzzer that tries different options on objdump. 

## objdump fuzzer

So, the first thing we gonna start with is the <b>HARNESS</b>

#### THE HARNESS 

The part of the program which gonna run the program and observe the crashes 

#### THE CORPUS

A set of minimal test inputs that generate maximal code coverage.



Here it gonna be any ELF file we have and we will take them from usr/bin
{% highlight bash %}
mkdir corpus

cd corpus

cp usr/bin/*
{% endhighlight %}


Now we have a lot of files in the corpus but we only need ELF, so let's remove any file, not ELF.
{% highlight bash %}
find * | xargs file | grep -v ELF | cut -d: -f1 | xargs rm
{% endhighlight %}

From here, he will start to write a fuzzing script in python to cover the basics of fuzzing but I am interested here in the Rust fuzzer. so you could watch the first video from minute 30 <a href="https://youtu.be/2xXt_q3Fex8?list=PLSkhUfcCXvqHsOy2VUxuoAf5m_7c8RqvO&t=2681">python fuzzer</a> and come back.


<a href="TBD">Day2</a>



















