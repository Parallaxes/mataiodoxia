---
layout: post
title: Protohackers Challenge 00: Smoke Test in Rust and Setting up DigitalOcean
description: >
  How I solved the zeroth challenge of Protohackers (Smoke Test) in Rust using a server hosted on DigitalOcean.
sitemap: false
hide_last_modified: true
categories: [rust, protohackers]
---

# Protohackers Challenge 00: Smoke Test in Rust and Setting up DigitalOcean

## [Problem Statement](https://protohackers.com/problem/0)

<blockquote>
0: Smoke Test

Deep inside Initrode Global's enterprise management framework lies a component that writes data to a server and expects to read the same data back. (Think of it as a kind of distributed system delay-line memory). We need you to write the server to echo the data back. <br><br>

Accept TCP connections. <br><br>

Whenever you receive data from a client, send it back unmodified. <br><br>

Make sure you don't mangle binary data, and that you can handle at least 5 simultaneous clients. <br><br>

Once the client has finished sending data to you it shuts down its sending side. Once you've reached end-of-file on your receiving side, and sent back all the data you've received, close the socket so that the client knows you've finished. (This point trips up a lot of proxy software, such as ngrok; if you're using a proxy and you can't work out why you're failing the check, try hosting your server in the cloud instead). <br><br>

Your program will implement the TCP Echo Service from RFC 862.
</blockquote>

## Hosting a Server

The most important problem at hand is to figure out whether I want to host this server locally or on some VPS. I recently came to find out that my GitHub Education License actually provides me $200 worth of credits for DigitalOcean.

I went onto DigitalOcean, opened an account, and set up a VM:

```console
2 GB Memory / 1 AMD vCPU / 50 GB Disk / SFO3 - Ubuntu 24.10 x64
```

For our task, these specs are completely fine (if you actually write safe code, as you'll see when I try to solve later iterations of the challenges). As with all open ports on the internet, it was caught in massive netscans and promptly attacked by dozens of other IPs trying to authenticate. To combat this, I set up `ufw` and `fail2ban` to drop all connections not from explicitly defined IPs and to ban IPs that tried to authenticate incorrectly more than 3 times.

Initially, I did have trouble with my first two droplets, which I ended up bricking or screwing up network settings pretty badly. I ended up destroying those droplets and setting up the one mentioned above.

My updated config:

```console
root@rs-protohackers:~# ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    <My IP>
4040/tcp                   ALLOW IN    <My IP>
22/tcp                     ALLOW IN    206.189.113.124
4040/tcp                   ALLOW IN    206.189.113.124
```

## Solving the Task

I decided to write this simple TCP echo server using Rust's `std::net` library which provides `TcpListener` and `TcpStream`, and run these on spawned threads -- which, frankly, is a really bad design choice, but the level of testing this first challenge performs won't cause anything too unfavorable to happen.

```rs
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write};
use std::thread;
```

Let's write our main function and generate a listener.

```rs
pub fn run() -> std::io::Result<()> {
    println!("Port opened on 4040");
    let listener = TcpListener::bind("0.0.0.0:4040")?;

    for stream in listener.incoming() {
        println!("Received stream from {:?}", stream);
        handle_client(stream?);
    }

    Ok(())
}
```

I wrote this function in a `pub fn run() -> std::io::Result<()>` because I've attached it to a larger crate where I can specify which challenge to run: `cargo r 0`. I'm working on improving the modularity and placing each challenge in a different crate, but we're getting there. For the scope of this first task, this approach is completely fine.

Now as for how the code works:

- `pub fn run() -> std::io::Result<()>` will act as our de facto `main()` function, and returns a `Result` which will tell us whether our program panics or successful completes (returns an `Ok(())`).

- `let listener = TcpListener::bind("0.0.0.0:4040")?;` tells the program to open a port with full access from the internet on port `4040`. We require the `?` to handle the `Result` returned by `bind()` quietly.

- `for stream in listener.incoming() {}` just catches each stream opened to our port and allows us to iterate over them.

- `handle_client(stream?);` is a user defined function that will contain our code to handle the "echo" part of our TCP echo server.

For `handle_client()`:

```rs
fn handle_client(mut stream: TcpStream) {
    thread::spawn(move || {
        let mut buffer = [0; 1024];
        loop {
            match stream.read(&mut buffer) {
                Ok(0) => {
                    println!("Connection close");
                    break;
                }
                Ok(n) => {
                    if let Err(e) = stream.write_all(&buffer[..n]) {
                        eprintln!("Write error: {:?}", e);
                        break;
                    }
                }
                Err(e) => {
                    eprintln!("Read error: {:?}", e);
                    break;
                }
            }
        }
    });
}
```

- We take a parameter `mut stream: TcpStream`, which is our TCP stream (as the name suggests). We need to take this mutably so we can edit data in the stream to render to the sender.

- Now for the "unsafe" part of this code. We use `thread::spawn()` to spawn threads for each stream taken up by our TCP server, allocating a lot of resource to handle each request (I say "for each stream" because we're executing this function in the `for` loop of the `run()` function).

- We then make a buffer object: `let mut buffer: [u8; 1024] = [0; 1024]`, which is a `1024` byte array. The `[u8; 1024]` part is actually implicit because the Rust compiler is smart and understands we are trying to write raw bytes. Anyhow, this `buffer` will allow us to write raw data into it.

- For each thread spawned, we do a `loop` until termination. In this loop, we're handling the request of the sender. We use a `match stream.read(&mut buffer) {}` to read requests. The `read()` function reads data from the stream into some parameter variable, which we've set as a mutable reference to `buffer`. So, for the stream, we're reading its data mutably into our `buffer` variable.

- We have three branches for our `match` statement, so let's go over them individually. 

    - `Ok(0) => { break; }` indicates an EOF (end of file). The problem statements tells us to break the connection and return the data our server received. You might have noticed that in this branch we're not actually returning any data? Well, that's because if we're at an EOF, we've already returned all the data we need to, so we just have to close the connection as we'll see in the second branch.

    - `Ok(n) => { }` is where we actually receive data that can be read. The `n` is a `usize` that tells us the number of bytes of data that were read from the stream. The ensuing `if let Err(e) = stream.write_all(&buffer[..n]) {}` basically says "if writing into the stream fails, do the thing in the if statement," which is to write an error connection and break the message. The "default" behavior, I guess you could say, is to `write_all` into the buffer. Except, if we just write into `&buffer`, we don't know how many bytes we're supposed to be writing in, leading to some memory issues. It follows that we must specify how many bytes we want to write, which turns out the be editing the first `n` bytes of that `buffer` using the `[..n]`.

    - `Err(e) => { }` just tells us that if we have an error reading the stream, print an error message and break the connection.

And thus, that concludes our code!

## Test 

Let's open the server:

```console
root@rs-protohackers:~/tcp# cargo r 1
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/challenges 1`
Port opened on 4040
```

Then trying using `netcat` to connect to port 4040 + run a few echo tests:

```console
parallaxis@ASOOS:~$ nc 24.144.82.153 4040
a
a
hello
hello
^C
parallaxis@ASOOS:~$
```

The echo tests worked!

```console
root@rs-protohackers:~/tcp# cargo r 1
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/challenges 1`
Port opened on 4040
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 68.248.128.73:50168, fd: 10 })
Connection close
^C
root@rs-protohackers:~/tcp#
```

Nice! Now let's run it against the actual testcases.


## Results

```console
root@rs-protohackers:~/tcp# ./target/release/challenges 1
Port opened on 4040
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42500, fd: 10 })
Connection close
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42502, fd: 11 })
Connection close
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42504, fd: 10 })
Connection close
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42506, fd: 11 })
Connection close
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42510, fd: 10 })
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42512, fd: 11 })
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42514, fd: 12 })
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42516, fd: 13 })
Received stream from Ok(TcpStream { addr: 24.144.82.153:4040, peer: 206.189.113.124:42518, fd: 14 })
Connection close
Connection close
Connection close
Connection close
Connection close
```

![Protohackers Results 0: PASS](/assets/img/protohackers-0-results.png)
