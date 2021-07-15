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