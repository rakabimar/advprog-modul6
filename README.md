# Advance Programming Module 6: Concurrency

## Table of Contents
* [Milestone 1: Single-threaded Web Server Reflection](#milestone-1-single-threaded-web-server-reflection)

## Milestone 1: Single-threaded Web Server Reflection

This is a simple single-threaded web server implemented in Rust. The web server consists of two main functions:

1. `main()`: Sets up a TCP listener and handles incoming connections
2. `handle_connection()`: Processes each individual TCP stream (connection)

First, I established the TCP Listener setup:

```rust
let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
```

This code creates a TCP listener bound to localhost (127.0.0.1) on port 7878. The `unwrap()` method extracts the success value or panics if there's an error (e.g., if the port is already in use). By binding to localhost, the server only accepts connections from the same machine.

Next, I implemented the connection handling loop:

```rust
for stream in listener.incoming() { 
    let stream = stream.unwrap();
    
    handle_connection(stream);
}
```

`listener.incoming()` returns an iterator over connection attempts, and for each successful connection (`stream.unwrap()`), I call `handle_connection()`. This creates a simple loop that processes one connection at a time, making this a single-threaded server. Because the server handles requests sequentially, it can only process one client request before moving on to the next.

For processing HTTP requests, I implemented the `handle_connection()` function:

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    println!("Request: {:#?}", http_request);
}
```

This function processes incoming HTTP requests by:
1. Creating a buffered reader from the TCP stream using `BufReader::new(&mut stream)`, which is more efficient for reading line by line
2. Processing the HTTP request headers through a chain of iterator operations:
    - `.lines()` returns an iterator over the lines in the buffered reader
    - `.map(|result| result.unwrap())` extracts the string from each Result (or panics if there's an error)
    - `.take_while(|line| !line.is_empty())` continues taking lines until an empty line is encountered, which marks the end of the HTTP headers
    - `.collect()` gathers all these lines into a vector
3. Finally, printing the HTTP request headers with pretty formatting using `println!("Request: {:#?}", http_request)`

At this stage, the server has several limitations:
- It simply receives and prints HTTP requests without sending any responses
- It processes connections sequentially (single-threaded), which limits throughput
- It uses minimal error handling (relies on `unwrap()` which will crash the program on errors)
- It doesn't parse or route requests based on paths or methods
- It lacks response generation functionality