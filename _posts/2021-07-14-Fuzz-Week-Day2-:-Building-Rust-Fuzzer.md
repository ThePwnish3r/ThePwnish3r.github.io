---
layout: post
---

### What we did at <a href="https://thepwnish3r.github.io/2021/07/14/Fuzz-Week-Day1-configure-objdump-from-binutils.html">Day 1</a> ?
- built objdump 
- we copied usr/bin as a corpus
- building a simple fuzzer and harness in python 



In the day 2 video from  <a href="https://youtu.be/iM3s8-umRO0?list=PLSkhUfcCXvqHsOy2VUxuoAf5m_7c8RqvO&t=2336">38:08</a> -> 01:03 there is a really good discussion about VR and bug-bounty if you are interested you should see it.





Now let's continue FUZZING.
We are going to rewrite the fuzzer in rust to have way better performance and control over threads.

<h1>Rust Fuzzer</h1>

We start with building a new project with cargo new and copy our corpus in it to use it later.

Then, we write Fuzz function to take input and put it into a temp file, and run objdump against this file.
```rust
// save inp to  disk with unique filename and thr id and run it through objdump once and returning status from objdump

fn fuzz<P: AsRef<Path>>(filename: P, inp: &[u8]) -> io::Result<ExitStatus> {

    //write out the input to a temp file
    std::fs::write(filename.as_ref(), inp)?;

    let runner = Command::new("./objdump")
        .args(&["-x", filename.as_ref().to_str().unwrap()])
        .output()?;

    Ok(runner.status)
}
```

Now, we build a worker function that will loop forever and run fuzz function.
```rust
fn worker(thr_id: usize, statistics:Arc<Statistics>)-> io::Result<()>{

    let filename = format!("tempinput{}", thr_id);

    loop{
        for _ in 0..BATCH_SIZE{
        
            fuzz(&filename,b"abcdef")?;
        }
        //update statistics 
        statistics.fuzz_cases.fetch_add(BATCH_SIZE,Ordering::SeqCst);       

    }
    

}
```
Here you can see that we added a constant called BATCH_SIZE , this is the number of iteration to run ber thread before reporting statistics to global statistics.


Then we try to add a safe way to report statistics about all threads.
```rust
//statsitcs during fuzzing 
    let stats = Arc::new(Statistics::default());


    for thr_id in 0..1{

        let stats =stats.clone();
        //spwan the thread
       std::thread::spawn(move || { worker(thr_id,stats)});

    }
    //start timer 
    let start = Instant::now();
   
    loop{
        //compute and print statistics 
        std::thread::sleep(Duration::from_millis(1000));
        let elapsed = start.elapsed().as_secs_f64();
        let cases = stats.fuzz_cases.load(Ordering::SeqCst);
        
        print!("[{:10.6}] cases{:10} | fcps{:10.2} \n",elapsed,cases,cases as f64/elapsed);
    }
```
We did it like this, using  Atomically Reference Counted 'ARC',  Arc<T> uses atomic operations for its reference counting. This means that it is thread-safe. The disadvantage is that atomic operations are more expensive than ordinary memory accesses.


Now we are ready to fuzz, our main will looks like that.
```rust
fn main() -> io::Result<()> {

    
    //statsitcs during fuzzing 
    let stats = Arc::new(Statistics::default());


    for thr_id in 0..4{

        let stats =stats.clone();
        //spwan the thread
       std::thread::spawn(move || { worker(thr_id,stats)});

    }
    //start timer 
    let start = Instant::now();
   
    loop{
        //compute and print statistics 
        std::thread::sleep(Duration::from_millis(1000));
        let elapsed = start.elapsed().as_secs_f64();
        let cases = stats.fuzz_cases.load(Ordering::SeqCst);
        
        print!("[{:10.6}] cases{:10} | fcps{:10.2} \n",elapsed,cases,cases as f64/elapsed);
    }


    Ok(())
}

```

And let's run it using  
{% highlight bash %}
cargo run --release
{% endhighlight %}

 <img src="/assets/images/Screenshot from 2021-06-06 10-05-25.png" alt="first atpcc">

 YAAY WE ARE FUZZING !!


So, here I am running it with 4 cores laptop, and it was about 500 cases per second, if you try it with 196 cores as Brandon did in the video you should get on a linear scale about 48000 fuzz cases per second. But sadly, it is like you have 4 cores running !!

That happens when you try to do a live fuzzing, you can't scale. So, you need to use an emulator or hypervisor, don't ever fuzz on a live system.

We found the best performance at 10 core, more than this it gets slow.





Next, we will load our corpus and try to do real fuzzing with real corpus.

We will load the corpus and deduplicate it using a set like BTresSet, then return the set back in a vector and put it in ARC to use it between threads safely as we talked about that earlier. That is why we are writing in rust here you can feel the concurrency, we can use corpus between threads without worrying about race conditions and stuff like this.

```rust
fn main() -> io::Result<()> {

    //load initial corpus

    let mut corpus = BTreeSet::new();

    for filename in std::fs::read_dir("corpus")?{

        let filename =filename?.path();

        corpus.insert(std::fs::read(filename)?);


    }

    let corpus:Arc<Vec<Vec<u8>>> = Arc::new(corpus.into_iter().collect());

    print!("loaded{} this files into corpus",corpus.len());

```
We run it and check everything is ok.
 <img src="/assets/images/Screenshot from 2021-06-06 13-46-19.png" alt="first atpcc">

Nice, we have about 1363 files in the corpus.


Now, we want to select corpus through a random number generator, we will implement one called xor-shift which is very fast you can read about it here  <a href="https://docs.rs/rand_xorshift/0.3.0/rand_xorshift/struct.XorShiftRng.html">XorShifting</a>

```rust
struct Rng(u64);


impl Rng{
    //create new number generator
    fn new()-> Self{
        // hex(random.randint(0, 2**64 - 1))
        // Also XORing seed by current uptime of processor so each
        // thread's rng is unique

        Rng(0x8644d6eb17b7ab1a ^ unsafe{ std::arch::x86_64::_rdtsc() })
    }
    #[inline]

    //generate random number 
    fn rand(&mut self) -> usize {

        let val = self.0;

        self.0 ^= self.0 << 13 ;
        self.0 ^= self.0 >> 17 ;
        self.0 ^= self.0 << 43 ;

        val as usize 
    }


}
```
It is just a snip you can follow other changes through Github repo we here just try to understand each part of the code.

Now, we will edit the worker function like this to select random input from the corpus and corrupt it as we did in python .
```rust
fn worker(thr_id: usize, statistics:Arc<Statistics>,corpus:Arc<Vec<Vec<u8>>>)-> io::Result<()>{
    

    let mut rng = Rng::new();


    let filename = format!("tempinput{}", thr_id);


    // input for fuzz case
    let mut fuzz_input = Vec::new();

    loop{
        //pick a random entry from the corpus 
        let sel = rng.rand() % corpus.len();


        // copy random input from corpus into fuzz input
        fuzz_input.clear();
        fuzz_input.extend_from_slice(&corpus[sel]);

        // Randomly corrupt the input
        for _ in 0..(rng.rand() % 8) + 1 {
            let sel = rng.rand() % fuzz_input.len();
            fuzz_input[sel] = rng.rand() as u8;
        }
        //
        for _ in 0..BATCH_SIZE{
        
            fuzz(&filename,&fuzz_input)?;
        }
        //update statistics 
        statistics.fuzz_cases.fetch_add(BATCH_SIZE,Ordering::SeqCst);       

    }
    

}
```

Let's try to run it.
 <img src="/assets/images/Screenshot from 2021-06-06 14-33-04.png" alt="first atpcc">

Nice, it worked fine.




This is the full code until now.
```rust
//use std::fs;
use std::io;
use std::path::Path;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize,Ordering};
use std::process::{Command, ExitStatus};
//use std::result::Result;
use std::time::{Instant,Duration};
use std::collections::BTreeSet;



//number of iteration to run ber thread before reborting statisticst to global statistics
const BATCH_SIZE :usize = 100;


#[derive(Default)]


struct Statistics{
    //number of fuzz cases performed
    fuzz_cases : AtomicUsize,


}
struct Rng(u64);


impl Rng{
    //create new number generator
    fn new()-> Self{
        // hex(random.randint(0, 2**64 - 1))
        // Also XORing seed by current uptime of processor so each
        // thread's rng is unique

        Rng(0x8644d6eb17b7ab1a ^ unsafe{ std::arch::x86_64::_rdtsc() })
    }
    #[inline]

    //generate random number 
    fn rand(&mut self) -> usize {

        let val = self.0;

        self.0 ^= self.0 << 13 ;
        self.0 ^= self.0 >> 17 ;
        self.0 ^= self.0 << 43 ;

        val as usize 
    }


}




// save inp to  disk with uniqe filename and thr id and run it through objdump once and returning status from objdump
fn fuzz<P: AsRef<Path>>(filename: P, inp: &[u8]) -> io::Result<ExitStatus> {

    //write out the input to a temp file
    std::fs::write(filename.as_ref(), inp)?;

    let runner = Command::new("./objdump")
        .args(&["-x", filename.as_ref().to_str().unwrap()])
        .output()?;

    Ok(runner.status)
}




fn worker(thr_id: usize, statistics:Arc<Statistics>,corpus:Arc<Vec<Vec<u8>>>)-> io::Result<()>{
    

    let mut rng = Rng::new();


    let filename = format!("tempinput{}", thr_id);


    // input for fuzz case
    let mut fuzz_input = Vec::new();

    loop{
        //pick a random entry from the corpus 
        let sel = rng.rand() % corpus.len();


        // copy random input from corpus into fuzz input
        fuzz_input.clear();
        fuzz_input.extend_from_slice(&corpus[sel]);

        // Randomly corrupt the input
        for _ in 0..(rng.rand() % 8) + 1 {
            let sel = rng.rand() % fuzz_input.len();
            fuzz_input[sel] = rng.rand() as u8;
        }
        //
        for _ in 0..BATCH_SIZE{
        
            fuzz(&filename,&fuzz_input)?;
        }
        //update statistics 
        statistics.fuzz_cases.fetch_add(BATCH_SIZE,Ordering::SeqCst);       

    }
    

}




fn main() -> io::Result<()> {

    //load initial corpus

    let mut corpus = BTreeSet::new();

    for filename in std::fs::read_dir("corpus")?{

        let filename =filename?.path();

        corpus.insert(std::fs::read(filename)?);


    }

    let corpus:Arc<Vec<Vec<u8>>> = Arc::new(corpus.into_iter().collect());

    print!("loaded {} this files into corpus",corpus.len());

    //statsitcs during fuzzing 
    let stats = Arc::new(Statistics::default());

    let corpus = corpus.clone();

    for thr_id in 0..4{

        let stats =stats.clone();

        let corpus = corpus.clone();
        //spwan the thread
       std::thread::spawn(move || { worker(thr_id,stats ,corpus)});

    }
    //start timer 
    let start = Instant::now();
   
    loop{
        //compute and print statistics 
        std::thread::sleep(Duration::from_millis(1000));
        let elapsed = start.elapsed().as_secs_f64();
        let cases = stats.fuzz_cases.load(Ordering::SeqCst);
        
        print!("[{:10.6}] cases{:10} | fcps{:10.2} \n",elapsed,cases,cases as f64/elapsed);
    }


    Ok(())

}
```

A bit missy but it is ok.







Now, let's take the return code from the fuzz case and save inputs to observe crashes. We will do that through ExitStatusExt and check if the signal equals 11 so it means SIGSEGV.

```rust
//to get exit status 
            let exit = fuzz(&filename,&fuzz_input)?;


            if let Some(11) = exit.signal() {
                //SIGSEGV
                print!("Crash !!\n"); 
        
       
            }
```

Let's run it.
 <img src="/assets/images/Screenshot from 2021-06-07 10-41-27.png" alt="first atpcc">

Yay! It worked, it started with unexpected high fcps, and my laptop screen froze after 2 seconds, maybe because I am running on all cores and opening Mozilla and VS code at the same time, that is a lot of stuff for my laptop so I will continue with one core for now. 



<img src="/assets/images/Screenshot from 2021-06-07 10-58-25.png" alt="first atpcc">

It worked better with one core and didn't freeze but I don't know why I am getting high fcps, it is okay let's continue to add some crash statistics.


We add crashes in our struct for statistics and take statistics through our loop in worker function, then we add crashes next to fcps.
```rust
if let Some(11) = exit.signal() {
                //SIGSEGV
                 //prin!("crash!");

                statistics.crashes.fetch_add(1,Ordering::SeqCst);

            }
```


Let's try

<img src="/assets/images/Screenshot from 2021-06-07 11-16-32.png" alt="first atpcc">

Perfect, expected more crashes but we fuzzing on a live system so we can't expect anything.





So, as you can see it is still very slow and we can't scale on a live system. To solve this problem we either use an emulator or hypervisor so we can deal with every fuzz instance as an independent instance from each other, this will help us to scale linearly and give us way better performance.



Next blog post we are going to write a RISCV emulator.