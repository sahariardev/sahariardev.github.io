---
title: Building a load balancer (day one)
date: 2025-04-01 12:00:00 +600
categories: [technical]
tags: [java, loadbalancer]
---

In this series of articles we are going to build a simple load balancer. With following features:

* <strong>Balance load accross multiple nodes</strong>
* <strong>Use different algorightm for load balancing</strong>
* <strong>Add headers for both request and response</strong>
* <strong>Log request and response timings</strong>


## Building a echo server
We will first start with building a simple echo server that returns what is received

```java
public class Main {
    public static void main(String[] args) {
        try (ServerSocket serverSocket = new ServerSocket(8888);
             ExecutorService executorService = Executors.newCachedThreadPool();) {

            while (true) {
                Socket socket = serverSocket.accept();
                executorService.execute(() -> {

                    try (InputStream inputStream = socket.getInputStream();
                         OutputStream outputStream = socket.getOutputStream()) {

                        byte[] buffer = new byte[8 * 1024];
                        int bytesRead;
                        while ((bytesRead = inputStream.read(buffer)) != -1) {
                            outputStream.write(buffer, 0, bytesRead);
                            outputStream.flush();
                        }

                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }

                });
            }

        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

In the above code we are using a <code>ServerSocket</code> to listen on port 8888. When a request comes in we are creating a new thread using <code>ExecutorService</code>. The request is read from the input stream and written back to the output stream

## Testing the echo server
We can test the echo server using <code>netcat</code>
```bash
echo "hello world" | nc 127.0.0.1 8888 
```
This should return "hello world"

You can find the full code [here](https://github.com/sahariardev/building-simple-load-balancer/blob/master/day01/src/main/java/org/sahariardev/Main.java)
