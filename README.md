# Advance Programming Module 6: Concurrency

## Table of Contents
* [Milestone 1: Single-threaded Web Server Reflection](#milestone-1-single-threaded-web-server-reflection)
* [Milestone 2: Returning HTML Reflection](#milestone-2-returning-html-reflection)
* [Milestone 3: Validating Requests and Selectively Responding Reflection](#milestone-3-validating-requests-and-selectively-responding-reflection)
* [Milestone 4: Simulating Slow Response Reflection](#milestone-4-simulating-slow-response-reflection)
* [Milestone 5: Multithreaded Server Reflection](#milestone-5-multithreaded-server-reflection)
* [Bonus Milestone: Improved Error Handling Reflection](#bonus-milestone-improved-error-handling-reflection)

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

## Milestone 4: Simulating Slow Response Reflection

In this milestone, I've enhanced the web server to demonstrate the impact of slow responses on a single-threaded server. By introducing a route that deliberately delays its response, the server now illustrates the blocking nature of single-threaded processing and how it affects concurrent request handling.

The key modification is in the `handle_connection()` function, which now includes a new route handler:

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(10));
            ("HTTP/1.1 200 OK", "hello.html")
        },
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };
    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();
    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```

This updated function introduces several important changes:

1. I've added a new route handler for `/sleep` that intentionally delays the response by 10 seconds using `thread::sleep(Duration::from_secs(10))`
2. After the sleep period, it serves the same "hello.html" content as the root route with a 200 OK status
3. The `match` expression is extended to handle this new route alongside the existing ones
4. The code now explicitly imports the `thread` module and `Duration` struct from the standard library

This implementation effectively demonstrates a key limitation of single-threaded servers: blocking behavior. When a client requests the `/sleep` route, the server becomes unresponsive to all other clients for 10 seconds. This happens because:

1. The server processes connections sequentially from the `listener.incoming()` iterator
2. The single thread can only handle one connection at a time
3. The current connection must complete (including any delays) before the next one starts
4. The `thread::sleep()` simulates a CPU-bound or I/O-bound operation that takes time to complete

This behavior accurately represents how many simple servers behaved historically and highlights why more sophisticated concurrency models were developed for web servers.

At this stage, the server has made the following progress:
- It routes requests to different handlers based on the path
- It demonstrates the impact of slow operations on single-threaded servers
- It provides a practical example of blocking behavior in concurrent request scenarios

However, the server still has the following limitations:
- It cannot handle multiple requests concurrently
- Long-running operations block all other client connections
- It remains vulnerable to denial-of-service situations where slow requests could make the server unresponsive
- Error handling remains minimal (using `unwrap()`)
- It lacks more advanced features like request headers parsing, query parameters, or POST request handling

## Milestone 5: Multithreaded Server Reflection

In this milestone, I've transformed the web server into a multithreaded application by implementing a thread pool pattern. This significant architectural change addresses the performance limitations identified in the previous milestone by enabling concurrent request handling.

The main enhancement is the creation of a `ThreadPool` implementation in a separate `lib.rs` file and updating the server to use this implementation:

```rust
// In main.rs
use hello::ThreadPool;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }
}
```

The key architectural components include:

1. **Thread Pool Creation**: The server initializes a thread pool with 4 worker threads using `ThreadPool::new(4)`
2. **Job Delegation**: Each incoming connection is wrapped in a closure and passed to the thread pool via the `execute()` method
3. **Concurrent Processing**: Worker threads handle connections independently, allowing parallel request processing

The core of the implementation is in the `lib.rs` file:

```rust
pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);
        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));
        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);
        self.sender.send(job).unwrap();
    }
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv().unwrap();
            println!("Worker {id} got a job; executing.");
            job();
        });

        Worker { id, thread }
    }
}
```

This implementation uses several key Rust concurrency primitives:

1. **MPSC Channel**: A multiple-producer, single-consumer channel (`mpsc::channel()`) that allows sending jobs from the main thread to worker threads
2. **Arc (Atomic Reference Counter)**: Enables sharing the receiver across multiple threads with `Arc::clone(&receiver)`
3. **Mutex (Mutual Exclusion)**: Ensures that only one worker can access the receiver at a time with `receiver.lock()`
4. **Worker Threads**: Each worker spawns a thread that continuously polls for new jobs via the shared receiver
5. **Job Type**: Uses a boxed trait object (`Box<dyn FnOnce() + Send + 'static>`) to store closures that can be executed across thread boundaries

The architecture significantly improves server performance:

- It can handle multiple client connections simultaneously (up to 4 with the current configuration)
- When processing a time-intensive request (like the `/sleep` endpoint), only one worker thread is occupied while others remain available
- This eliminates the head-of-line blocking problem present in single-threaded servers
- The server remains responsive even when some requests take a long time to process

At this stage, the server has made substantial progress:
- It implements true concurrent request handling through a thread pool pattern
- It maintains the same routing functionality as before but with parallel processing capability
- It uses Rust's concurrency primitives to ensure thread-safe communication
- It provides debug output when workers receive jobs (`println!("Worker {id} got a job; executing.")`)

## Bonus Milestone: Improved Error Handling Reflection

In this bonus milestone, I've enhanced the thread pool implementation with more robust error handling by introducing a `build` function as an alternative to the `new` constructor. This improvement follows Rust's error handling best practices and provides callers with more flexibility when creating a thread pool.

The key enhancement is the addition of the `build` function in the `ThreadPool` implementation:

```rust
pub fn build(size: usize) -> Result<ThreadPool, PoolCreationError> {
    if size == 0 {
        return Err(PoolCreationError::ZeroSize);
    }

    let (sender, receiver) = mpsc::channel();
    let receiver = Arc::new(Mutex::new(receiver));
    let workers = (0..size)
         .map(|id| Worker::new(id, Arc::clone(&receiver)))
         .collect::<Vec<Worker>>();

    Ok(ThreadPool { workers, sender })
}
```

This implementation introduces several important improvements:

1. **Result Return Type**: The function returns a `Result<ThreadPool, PoolCreationError>` instead of directly returning a `ThreadPool`, following Rust's convention for operations that might fail
2. **Custom Error Type**: I've created a dedicated `PoolCreationError` enum to represent different failure modes
3. **Explicit Error Handling**: Instead of using `assert!` which causes a panic, the function returns an `Err` variant when invalid parameters are provided
4. **Functional Style**: The worker creation uses a more functional approach with `map` and `collect` instead of imperative loop construction

The `main.rs` file has been updated to use this new function:

```rust
let pool = ThreadPool::build(4).expect("Failed to create ThreadPool!");
```

This change shows how the caller can now handle potential creation failures. In this case, it still uses `expect` to terminate with a custom message if the pool creation fails, but in production code, proper error handling could be implemented.

The difference between the two approaches illustrates an important distinction in Rust's error handling philosophy:

1. **The `new` Function**:
   - Uses `assert!(size > 0)` which will panic if the condition fails
   - Follows Rust's convention that constructors named "new" should not fail or return Result
   - Appropriate when invalid inputs represent programming errors that should be caught early

2. **The `build` Function**:
   - Returns `Result<ThreadPool, PoolCreationError>` to represent possible failure
   - Allows the calling code to handle errors gracefully through the `Result` type
   - More appropriate when failures might be expected or recoverable
   - Enables more sophisticated error handling strategies

I've also introduced a proper error type:

```rust
#[derive(Debug)]
pub enum PoolCreationError {
    ZeroSize,
}
```

The `#[derive(Debug)]` attribute automatically implements the `Debug` trait, making the error printable and easier to work with during debugging.

At this stage, the server has made these additional improvements:
- It follows Rust's error handling best practices with Result types
- It provides a more robust alternative to thread pool creation
- It uses a custom error type for clearer error reporting
- It maintains all the concurrency benefits from the previous milestone