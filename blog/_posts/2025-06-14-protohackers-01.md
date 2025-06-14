---
layout: post
title: Protohackers Challenge 01: Prime Time in Rust
description: >
  How I solved the first challenge of Protohackers (Prime Time) in Rust using a server hosted on DigitalOcean.
sitemap: false
hide_last_modified: true
categories: [rust, protohackers]
---

# Protohackers Challenge 01: Prime Time in Rust

## [Problem Statement](https://protohackers.com/problem/1)

In simple words, receive JSON objects, check fields, return a response. Ok, I can do that!


## Solving the Task

Let's start with some imports of what we'll be using.

```rs
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{BufReader, AsyncBufReadExt, AsyncWriteExt};
use serde::{Deserialize, Serialize};
```

We're using `tokio` for async runtimes and networking, then `serde` for JSON **Ser**ialization and **De**serialization.

Let's first begin by defining what a `request` and `response` is:

```rs
#[derive(Debug, Serialize, Deserialize)]
struct Request {
    method: String,
    number: f64,
}

#[derive(Debug, Serialize, Deserialize)]
struct Response {
    method: String,
    prime: bool,
}

#[derive(Debug)]
enum Answer {
    Malformed,
}
```

- A `Request` is a `struct` with a `method` (a `String`) and a `number` (an `f64`), as defined in the problem statement

- A `Response` is a `struct` with a `method` (a `String`) and a `prime` (a `bool`), as defined in the problem statement as well.

- The problem statement also defined three types of answers: Correct (a normal response), Malformed (a malformed response), and Incorrect (a normal response with incorrect contents). Although it would be more idiomatic, I decided to simply ignore the first and last. In fact, you don't even need to define any of the answer types in your code, you can just execute what the problem wants to you do without definition.

Now let's take a look at our driver function:

```rs
pub async fn run() -> std::io::Result<()> {
    let listener = TcpListener::bind("0.0.0.0:4040").await?;
    println!("Server listening port 0.0.0.0:4040");

    loop {
        let (socket, addr) = listener.accept().await?;
        println!("New connection from {}", addr);

        tokio::spawn(async move {
            // Wrap socket in BufReader for line by line reading
            let mut reader = BufReader::new(socket);
            let mut line = String::new();

            loop {
                line.clear();

                let _bytes_read = match reader.read_line(&mut line).await {
                    Ok(0) => {
                        println!("Connection closed by client");
                        return;
                    }
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("read error: {:?}", e);
                        return;
                    }
                };

                // Trim whitespace and newline
                let trimmed = line.trim_end();

                // Handle the request, pass mutable TcpStream (unwrap from BufReader)
                let stream = reader.get_mut();

                if let Err(e) = handle_request(trimmed.as_bytes(), stream).await {
                    eprintln!("Request error: {:?}", e);
                    // Malformed request -> disconnect client
                    return;
                }
            }
        });
    }
}
```

I'm sure the comments explain enough here, so if you understand this code you can skip this explanation. Although, there is a very important part (starting after the second point).

- We begin by defining our `listener` (like Problem 00) and binding to `0.0.0.0:4040` under an `async` runtime.

- Then, we `loop` until the server is closed. Under this first `loop`, we `accept()` a stream under `listener` which includes a `socket` and an `addr`. Now here's a real important part. 

### I Screwed Up

I first began this implementation using Rust's native `thread::spawn`. Now, running this in an async runtime with a small amount of input is fine, but handling 1000s of simultaneous requests gets real messy real fast. I later switched to `tokio::spawn`.

Why? Well, for starters, `thread::spawn` isn't free. Every time you use it, you're spinning up a full OS thread, with each thread taking a decent chunk of memory (stack space), and the kernel has to schedule every thread independently. On paper, that sounds okay... until you try to handle simultaneous clients, each sending 1000s of messages, and suddenly your little 2 GB / 1vCPU VPS starts sweating a little (and by little I mean a lot).

On this setup, you're not getting a whole lot of thread performance. You're getting a *traffic jam*. Long story short, server kind of died and I had to do a little recovery debugging.

Meanwhile, `tokio::spawn` is built for this. It runs on a small pool of async workers that cooperate instead of compete. Instead of whatever living hell `thread::spawn` is, you get lightweight tasks that sleep and wake *efficiently*. They're stackless, don't eat up memory, and don't trigger the kernel to context switch every time one of them blinks.

### Back to Explaining Code

- Using `tokio::spawn(async move {`, we start a new async task (not a system thread) to keep things lightweight and non-blocking.

- `async move` captures the `socket` by value and runs it in a separate task. Noe need to worry about lifetimes or thread safety. The task owns its own data.

- `let mut reader = BufReader::new(socket);` wraps the raw `TcpStream` in a `BufReader` so we can read line by line instead of by byte or chunk. Why? Because the Protohackers protocol uses newline delimited JSON. Reading lines keep parsing sne and avoids awkward partial reads.

- `let mut line = String::new()` is our buffer for incoming lines. We are reusing the same `String` every loop iteration to avoid allocating repeatedly.

- `loop { line.clear();` is the core of the connection handler: we enter a loop to continuously read and process lines from the client. We `.clear()` the buffer to reuse it for the next read (no unnecessary heap allocations),

- `let _bytes_read = match reader.read_line(&mut line).await {` awaits a line of input from the client, then `read_line` returns the number of bytes read. If it's `0`, that means the client closed the connection (this would be `Ok(0) => { return ;});

- `Ok(n) => n` and `Err(e) => { return; }` are our next two branches. If we read successfully, we proceed. If something goes wrong, log the error and bail.

```rs
let trimmed = line.trim_end();
let stream = reader.get_mut();
```

- Clean the line then grab a mutable reference to the original `TcpStream` from the `BufReader`. This is necessary because we want to write out response directly back to the socket, and `BufReader` only wraps the read half.

- `if let Err(e) = handle_request(trimmed.as_bytes(), stream).await {` calls our async `handle_request` function, which takes the raw bytes and a writable stream. If `handle_request` returns an error, it's probably due to bad input -- time to close the connection.

The following code for handling client requests is relatively simple, so I've added comments which I hope will suffice:

```rs
async fn handle_request(data: &[u8], stream: &mut TcpStream) -> std::io::Result<()> {
    // Convert bytes to &str
    let input = match std::str::from_utf8(data) {
        Ok(s) => s,
        Err(_) => {
            // Send malformed response with newline and return error to close connection
            stream.write_all(b"{\"answer\":\"Malformed\"}\n").await?;
            return Err(std::io::Error::new(std::io::ErrorKind::InvalidData, "Invalid UTF-8"));
        }
    };

    match parse_request(input) {
        Ok(response) => {
            // Write response followed by newline
            stream.write_all(response.as_bytes()).await?;
            stream.write_all(b"\n").await?;
            Ok(())
        }
        Err(Answer::Malformed) => {
            // Malformed response, then close connection
            stream.write_all(b"{\"answer\":\"Malformed\"}\n").await?;
            Err(std::io::Error::new(std::io::ErrorKind::InvalidData, "Malformed request"))
        }
    }
}

fn parse_request(input: &str) -> Result<String, Answer> {
    // Parse JSON request
    let request: Request = serde_json::from_str(input).map_err(|_| Answer::Malformed)?;

    // Check required fields: method == "isPrime", number is a number (any JSON number, floating point allowed)
    if request.method != "isPrime" {
        return Err(Answer::Malformed);
    }

    // According to spec: non-integers cannot be prime, so prime = false if fractional part != 0
    let is_prime_result = if request.number.fract() != 0.0 {
        false
    } else {
        is_prime(request.number as u64)
    };

    let response = Response {
        method: request.method,
        prime: is_prime_result,
    };

    // Serialize response JSON string
    serde_json::to_string(&response).map_err(|_| Answer::Malformed)
}

fn is_prime(n: u64) -> bool {
    if n <= 1 {
        return false;
    }
    if n == 2 {
        return true;
    }
    if n % 2 == 0 {
        return false;
    }
    let sqrt_n = (n as f64).sqrt() as u64;
    for i in (3..=sqrt_n).step_by(2) {
        if n % i == 0 {
            return false;
        }
    }
    true
}
```

And that's a wrap! Let's see how it performs.

## Results

![01 Result](/assets/protohackers/1-result.png)