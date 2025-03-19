# Advance Programming Module 6: Concurrency

## Table of Contents
* [Milestone 1: Single-threaded Web Server Reflection](#milestone-1-single-threaded-web-server-reflection)
* [Milestone 2: Returning HTML Reflection](#milestone-2-returning-html-reflection)

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

# Milestone 2: Returning HTML Reflection

This milestone focuses on enhancing the server to generate and return proper HTTP responses with HTML content. The server now not only receives requests but also responds with a complete HTTP response containing a status line, headers, and HTML content loaded from a file.

The main improvement is in the `handle_connection()` function:

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();
    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```

The updated function now:

1. First reads the HTTP request headers using the same method as before
2. Creates a status line with "HTTP/1.1 200 OK" indicating a successful response
3. Reads HTML content from a file named "hello.html" using `fs::read_to_string()`
4. Calculates the length of the content to use in the Content-Length header
5. Constructs a complete HTTP response by formatting the status line, headers, and content
6. Writes the response back to the client using `stream.write_all()`

The HTML file "hello.html" contains a simple webpage with a heading and paragraph:

![Commit 2 screen capture](/assets/images/commit2.png)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Hello!</title>
</head>
<body>
<h1>Hello!</h1>
<p>Hi from Rust, running from Rakabima's machine.</p>
</body>
</html>
```

At this stage, the server has made significant progress by:
- Generating proper HTTP responses with a status line, headers, and body
- Serving static HTML content from a file
- Following the HTTP/1.1 protocol by including required headers like Content-Length

However, the server still has several limitations:
- It returns the same response (200 OK) regardless of the request path or method
- It only serves a single HTML file ("hello.html")
- Error handling is minimal (using `unwrap()` which will crash on errors)
- It processes connections sequentially (single-threaded), limiting throughput
- It lacks routing capabilities to serve different content based on request paths
