---
title: "Mini HTTP Server in C"
date: 2023-07-15T16:08:57+05:30
draft: false
toc: true
images:
tags:
  - Writeups
  - C
  - Socket programming
  - Multithreading
---
I came across this challenge on [codecrafters](https://app.codecrafters.io/catalog) and this really helped me to write meaningful applications and a lot about general code structure and practices instead of just grinding codeforces problems(lol)

This is the github repo where I've uploaded the full code :: [GitHub](https://github.com/ishXD/http-server-mini)

## TASK 1 : Bind to a port

We used the `sys/socket.h` library and so it is advisable to go through its various functions here [sys/socket.h](https://pubs.opengroup.org/onlinepubs/7908799/xns/syssocket.h.html).

This stage is pretty straightforward, just uncomment the code block to pass it. It is advisable to understand the functions used.

## TASK 2 : Respond with 200

The `accept()` function is used to ensure connection with the client and upon successful completion, it returns the nonnegative file descriptor of the accepted socket. Otherwise, -1 is returned.

The response is sent through the `send()` or `write()` function. Technically both functions have the same functionality but `send()` is used with sockets while `write(`) is used with local files.

## TASK 3 : Extract URL path

The `read()` function is used to read data from the file descriptor `fd` into the `buffer`. The `buffer` is an array that can hold 1024 characters, which should be enough for a simple HTTP request.
```C
int msg_Read = read(fd, buffer, 1024);
if (msg_Read < 0) {
  printf("read failed");
  return 1;
}
printf("Received HTTP request:\n%s\n", buffer);
```
Then I used the `sscanf()` function to parse the HTTP request that was received from the client. The `sscanf()` function reads formatted input from a string — in this case, the buffer which contains the HTTP request.

```C
char method[16], url[256], protocol[16];
sscanf(buffer, "%s %s %s", method, url, protocol);
```
The function will read from buffer and store the extracted values into `method`, `url`, and `protocol`. You can also use a structure for this.

This is a crucial step for the server to understand what action is being requested (the method), which resource is being requested (the URL path), and what version of the HTTP protocol the client is using.

```C
char response[BUFFER_SIZE];

if (strcmp(url, "/") == 0) {
    snprintf(response, sizeof(response), "HTTP/1.1 200 OK\r\n\r\n\r\n");
} else {
    snprintf(response, sizeof(response), "HTTP/1.1 404 Not Found\r\n\r\n\r\n");
}
printf("response : %s\n", response);

write(fd, response, sizeof(response) - 1);
```

The `response` buffer is used to store the HTTP response message. It then compares the extracted URL path (url) with the root path `/`. If the URL path is the root path, it uses `snprintf()` to write a `200 OK` status message into the response buffer. If the URL path is anything else, it writes a `404 Not Found` status message instead.

## TASK 4 : Respond with Body

I've included a new conditional block that checks if the requested URL starts with /echo/. If it does, the server prepares a response to echo back the string that follows the /echo/ path.

```C
} else if (strncmp(url, "/echo/", 6) == 0) {
  char *echo_msg = url + 6;
  snprintf(response, sizeof(response),
           "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nContent-Length: "
           "%d\r\n\r\n%s",
           strlen(echo_msg), echo_msg);
}
```

## TASK 5 : Read Header

The following code snippet checks if the requested URL is `/user-agent`. If it is, it searches for the User-Agent header in the request buffer using `strstr()`. Once found, it skips past the `"User-Agent: "`, which is 12 characters, to point to the actual user agent string.

If the end of the line (\r\n) is found after the user agent string, it replaces the carriage return with a null terminator to properly end the string. If the end of the line is not found, it returns an error code, assuming the request is malformed.

If the User-Agent header is not found at all, it sets the user agent string to a default message indicating the header was not found.
```C
else if (strncmp(url, "/user-agent", 11) == 0) {
    char *user_agent = strstr(buffer, "User-Agent:");
    if (user_agent) {
      user_agent += 12;
      char *eol = strstr(user_agent, "\r\n");
      if (eol)
        *eol = '\0';
      else
        return 1;
    } else {
      user_agent = "User-Agent not found";
    }
    printf("user-agent: %s", user_agent);
    snprintf(response, sizeof(response),
             "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nContent-Length: "
             "%d\r\n\r\n%s",
             strlen(user_agent), user_agent);
  }
  else {
```

## TASK 6 : Concurrent Connections 
First time dealing with multithreading for me.

Include the `phread.h` header file as we use the POSIX threads (pthreads) library to handle concurrent connections. The pthreads library provides a set of functions for creating, managing, and synchronizing threads, which allows the server to handle multiple client requests at the same time. Each client connection can be processed in a separate thread, ensuring that the server can continue to accept new connections while it's processing existing ones.

```C
while (1) {
  // Accept incoming connections and handle them in separate threads
  fd = malloc(sizeof(int));
  *fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_addr_len);
  pthread_t thread_id;
  if (pthread_create(&thread_id, NULL, handle_request, (void *)fd) < 0) {
    free(fd);
  }
  pthread_detach(thread_id);
}
close(server_fd);
```
The main loop of the server `while (1)` waits for client connections. Upon accepting a new connection, it allocates memory for the file descriptor and creates a new thread using `pthread_create` to handle the request. The `handle_request` function (all we've typed for the previous tasks) is passed to the thread, which will process the incoming HTTP request.

The thread is then detached using `pthread_detach`, allowing it to run independently from the main thread. This means the main thread can continue accepting new connections without waiting for the thread to finish.

Finally, the server socket is closed outside the loop, which will only be reached if the server is shutting down.

## TASK 7 : Return a file
We are parsing command line arguments so do the changes accordingly in `main()`. 

```C
int main(int argc, char *argv[]) {
    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "--directory") == 0) {
        strncpy(directory, argv[i + 1], sizeof(directory) - 1);
        directory[sizeof(directory) - 1] = '\0';
        }
    }
    printf("directory: %s\n", directory);
  ```
The main function now accepts arguments (`argc` for the count and `argv` for the actual arguments). A loop goes through each argument to check if it matches the `--directory` flag. If it does, the next argument is assumed to be the directory path, which is then copied into the directory variable using `strncpy()`. To ensure the string is null-terminated and to avoid buffer overflow, the size is limited to one less than the buffer size, and the last character is explicitly set to the null character `\0`.

The `handle_request` function introduces logic to handle requests for files. When a `GET` request is made to the `/files/{filename}` endpoint, the server now checks if the requested file exists within the specified directory. If the file exists, it reads the file contents into a buffer, closes the file, and constructs an HTTP response with a 200 OK status, setting the `Content-Type` to `application/octet-stream` and including the Content-Length header with the size of the file. The response body contains the file contents.

## TASK 8 : Read Request Body

We have to make changes for handling `POST` requests to the `/files/{filename}` endpoint. When the server receives a POST request, it extracts the filename from the URL and constructs the full file path using the provided directory. It then searches for the start of the request body, which follows the \r\n\r\n sequence.

If the body is found, the server opens the file for writing. If the file cannot be opened, an error message is printed. Otherwise, the contents of the body are written to the file, and the file is closed. After successfully creating the file, the server responds with a `201 Created` status.

```C
else if (strcmp(method, "POST") == 0) {
    char *file_requested = url + 7;
    char file_path[BUFFER_SIZE];
    snprintf(file_path, sizeof(file_path), "%s%s", directory, file_requested);
    char *body = strstr(buffer, "\r\n\r\n");
    if (body == NULL) {
        snprintf(response, sizeof(response), "HTTP/1.1 400 Bad Request\r\nContent-Type: text/plain\r\n\r\n400 Bad Request");
    } else {
        body += 4;
        FILE *file = fopen(file_path, "w");
        if (file == NULL) {
            printf("Error creating file: %s\n", strerror(errno));
        } else {
            fprintf(file, "%s", body);
            fclose(file);
            snprintf(response, sizeof(response), "HTTP/1.1 201 Created\r\n\r\n");
        }
    }
}
```

## TASK 9 : Compression Headers

The updated code checks if the incoming HTTP request contains the `Accept-Encoding: gzip` header. If it does, the server prepares a response with the `Content-Encoding: gzip` header to indicate that the content would be compressed using gzip (although actual compression is not implemented at this stage). If the header is not present or contains a value other than gzip, the server responds without the Content-Encoding header, indicating that no compression is applied

```C
if (strstr(buffer, "Accept-Encoding: gzip") != NULL) {
  snprintf(response, sizeof(response),
           "HTTP/1.1 200 OK\r\nContent-Encoding: gzip\r\nContent-Type: "
           "text/plain\r\nContent-Length: %d\r\n\r\n%s",
           strlen(echo_msg), echo_msg);
}else {
  snprintf(response, sizeof(response),
           "HTTP/1.1 200 OK\r\nContent-Type: "
           "text/plain\r\nContent-Length: %d\r\n\r\n%s",
           strlen(echo_msg), echo_msg);
}
```
## TASK 10 : Multiple Compression Schemes

Instead of only checking for the presence of "gzip" in the entire request buffer, we now have to for the specific header `"Accept-Encoding: "` and then extracts the list of encodings provided by the client.

Here's a breakdown of the changes:

- The code now searches for the start of the Accept-Encoding header and finds the end of the line with \r\n.
- It then copies the list of encodings into a separate buffer(`enccodings`), ensuring it's properly terminated with a null character.
- The server checks if "gzip" is one of the encodings listed by the client.
- If "gzip" is found, the server responds with a `Content-Encoding: gzip` header.
- If "gzip" is not found, the server responds without the Content-Encoding header.

```C
char *encoding_header = strstr(buffer, "Accept-Encoding: ");
if (encoding_header != NULL) {
    char *encoding_crlf = strstr(encoding_header, "\r\n");
    char encodings[BUFFER_SIZE];
    strncpy(encodings, encoding_header + 17, encoding_crlf - (encoding_header + 17));
    encodings[encoding_crlf - (encoding_header + 17)] = '\0';

    if (strstr(encodings, "gzip") != NULL) {
        // Respond with gzip encoding
    } else {
        // Respond without gzip encoding
    }
} else {
    // Respond without gzip encoding
}
```
## TASK 11 : Gzip Compression

I added an include directive for the `zlib.h` header file, as it provides the functions and types necessary for compression and decompression using the gzip format.

```C
static char *gzip_deflate(char *data, size_t data_len, size_t *gzip_len) {
  z_stream stream = {0};
  deflateInit2(&stream, Z_DEFAULT_COMPRESSION, Z_DEFLATED, 0x1F, 8,
               Z_DEFAULT_STRATEGY);
  size_t max_len = deflateBound(&stream, data_len);
  char *gzip_data = malloc(max_len);
  memset(gzip_data, 0, max_len);
  stream.next_in = (Bytef *)data;
  stream.avail_in = data_len;
  stream.next_out = (Bytef *)gzip_data;
  stream.avail_out = max_len;
  deflate(&stream, Z_FINISH);
  *gzip_len = stream.total_out;
  deflateEnd(&stream);
  return gzip_data;
}
```
The function `gzip_deflate()` compresses a given string using gzip compression. 
- It initializes a `z_stream` structure, which is used by the zlib library to manage compression streams. 
- `The deflateInit2()` function sets up the stream for gzip compression with default settings. 
- The `deflateBound()` function estimates the maximum compressed size of the input data and allocates memory for the compressed output. 
- The input data and its length are set in the stream, and `deflate()` is called to perform the actual compression. 
- After compression, the total size of the compressed data is stored in the provided `*gzip_len` pointer. 
- Finally, `deflateEnd()` cleans up the stream structure, and the function returns a pointer to the compressed data. 

```C
if (strstr(encodings, "gzip") != NULL) {
    char *compressed_buffer;
    long unsigned int compressed_len;

    compressed_buffer = gzip_deflate(echo_msg, strlen(echo_msg), &compressed_len);

    snprintf(response, sizeof(response),
             "HTTP/1.1 200 OK\r\nContent-Encoding: gzip\r\nContent-Type: "
             "text/plain\r\nContent-Length: %ld\r\n\r\n",
             compressed_len);

    send(fd, response, strlen(response), 0);
    send(fd, compressed_buffer, compressed_len, 0);

    return NULL;
}
```
Instead of sending the response in one go, we first send the headers using `send()`, and then the compressed body in a separate send call.
For some reason `write()` threw an error here but send() works fine.

And that's THE END. I do need to polish this code a bit and look into multithreading as well.
Overall this challenge is great for people who want to start writing actual meaningful code. There are other Write  your own.... challenges as well which cover popular devtools. Highly recommend checking them out!



 