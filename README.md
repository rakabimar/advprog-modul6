# Advance Programming Module 6: Concurrency

## Table of Contents
* [Milestone 1: Single-threaded Web Server Reflection](#milestone-1-single-threaded-web-server-reflection)
* [Milestone 2: Returning HTML Reflection](#milestone-2-returning-html-reflection)
* [Milestone 3: Validating Requests and Selectively Responding Reflection](#milestone-3-validating-requests-and-selectively-responding-reflection)

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

## Milestone 2: Returning HTML Reflection

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

## Milestone 3: Validating Requests and Selectively Responding Reflection

In this milestone, I've enhanced the web server to validate incoming requests and respond selectively with different content based on the request path. The server now properly handles both valid and invalid routes by serving appropriate HTML content and status codes.

The key improvement in this milestone is the updated `handle_connection()` function:

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };
    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();
    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```

This updated function introduces several important changes:

1. Instead of collecting all HTTP headers, I now extract only the first line (the request line) using `buf_reader.lines().next()`. This line contains the HTTP method, path, and version information that's needed for routing.

2. I've implemented a `match` expression to determine the appropriate response based on the request path:
   - For requests to the root path (`GET / HTTP/1.1`), the server returns a 200 OK status with "hello.html"
   - For all other requests, the server returns a 404 NOT FOUND status with "404.html"

3. The `match` expression returns a tuple of values `(status_line, filename)`, which elegantly handles the relationship between the status code and the content to be served.

4. The response generation logic remains consistent, reading the appropriate HTML file and constructing the HTTP response with the correct status line, Content-Length header, and body content.

The server now also serves a custom 404 error page:

![Commit 3 screen capture](/assets/images/commit3.png)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>404</title>
</head>
<body>
<h1>Oops!</h1>
<p>Sorry, I don't know what you're asking for.</p>
<p>Rust is running from Rakabima's machine.</p>
</body>
</html>
```

This provides a more user-friendly experience when visitors attempt to access non-existent routes.

The refactoring from traditional if-else statements to a `match` expression offers several benefits:

1. It provides a more concise and readable code structure
2. It ensures all variables are properly initialized for all possible cases
3. The tuple return pattern reduces the chance of inconsistency between related variables
4. Adding new routes becomes cleaner and requires less code duplication
5. It allows the use of immutable variables, which is preferred in Rust for safety and clarity

At this stage, the server has made notable progress:
- It validates incoming HTTP requests and extracts the request line
- It routes requests to different responses based on the requested path
- It returns appropriate status codes (200 for valid routes, 404 for invalid routes)
- It serves different HTML content based on the request validity

However, the server still has limitations:
- It only handles GET requests to the root path specifically
- It uses a simple string match for routing rather than a more flexible router
- All HTML content is loaded from disk for each request without caching
- Error handling remains minimal (using `unwrap()`)
- It continues to process connections sequentially (single-threaded)
