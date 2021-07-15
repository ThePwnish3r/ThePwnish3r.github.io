---
layout: post
---

### what we did at Day 1?
- built objdump 
- we copied usr/bin as a corpus
- building a simple fuzzer and harness in python 


In the day 2 video from 38:08 -> 01:03 there is a really good discussion about VR and bug-bounty if you are interested you should see it.



now let's continue FUZZING.
we are going to rewrite the fuzzer in rust to have way better performance and control over threads.

<h1>Rust Fuzzer</h1>

We start with building a new project with cargo new and copy our corpus in it to use it later.

then, we write Fuzz function to take input and put it into a temp file, and run objdump against  this file.
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

now, we build a worker function that will loop forever and run fuzz function.
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
here you can see that we added a constant called BATCH_SIZE , this is the number of iteration to run ber thread before reporting statistics to global statistics.


then we try to add a safe way to report statistics about all threads.
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
we did it like this, using  Atomically Reference Counted 'ARC',  Arc<T> uses atomic operations for its reference counting. This means that it is thread-safe. The disadvantage is that atomic operations are more expensive than ordinary memory accesses.


now we are ready to fuzz, our main will looks like that.
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

and let's run it using  
{% highlight bash %}
cargo run --release
{% endhighlight %}
    
 <img src="_images/Screenshot from 2021-06-06 10-05-25.jpg" alt="first attemp">

 YAAY WE ARE FUZZING !!
