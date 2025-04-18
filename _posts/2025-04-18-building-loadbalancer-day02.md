---
title: Building a load balancer (day two)
date: 2025-04-18 15:18:00 +600
categories: [technical]
tags: [java, loadbalancer]
---

## Building the load balancer

We will implement very simple loadbanacer that takes a request, select a server using round robin method and forwad the request to that server

### lets build the Load balancer class first

```java
    public class Loadbalancer {

    private final ExecutorService executor = Executors.newCachedThreadPool();

    private final List<Host> hosts = new ArrayList<>();

    int currentHostIndex = 0;

    public synchronized void add(Host host) {
        hosts.add(host);
    }

    private synchronized Host getNextHost() {
        Host host = this.hosts.get(currentHostIndex);
        currentHostIndex = (currentHostIndex + 1) % this.hosts.size();

        return host;
    }

    public void forward(Socket serverSocket) {
        executor.submit(() -> {
            Host host = getNextHost();
            System.out.println("Forwarding to -> host: " + host.getHost() + " port: " + host.getPort());
            try (Socket targetSocket = new Socket(host.getHost(), host.getPort());
                 InputStream clientInputStream = serverSocket.getInputStream();
                 OutputStream clientOutputStream = serverSocket.getOutputStream();
                 InputStream targetInputStream = targetSocket.getInputStream();
                 OutputStream targetOutputStream = targetSocket.getOutputStream();) {

                Future<?> upStreamFuture = executor.submit(() -> {
                    try {
                        copyStream(clientInputStream, targetOutputStream, "request");
                    } catch (IOException e) {
                        System.out.println("error copying stream");
                    }
                });

                Future<?> downStreamFuture = executor.submit(() -> {
                    try {
                        copyStream(targetInputStream, clientOutputStream, "response");
                    } catch (IOException e) {
                        System.out.println("error copying stream");
                    }
                });

                upStreamFuture.get();
                downStreamFuture.get();

            } catch (IOException | ExecutionException | InterruptedException e) {
                System.out.println("error: " + e.getMessage());
            }
        });
    }

    private void copyStream(InputStream inputStream, OutputStream outputStream, String type) throws IOException {
        byte[] buffer = new byte[1024];
        int read;

        try (inputStream; outputStream) {
            while ((read = inputStream.read(buffer)) != -1) {
                System.out.println("--------" + type + "-------");
                System.out.println(new String(buffer, StandardCharsets.UTF_8));
                outputStream.write(buffer, 0, read);
                outputStream.flush();
            }
        } catch (SocketException e) {

        }
    }

}
```

```java
public class Host {

    private final String host;

    private final int port;

    public Host(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public String getHost() {
        return host;
    }

    public int getPort() {
        return port;
    }
}
```

* <strong> executor: </strong> In the Loadbalancing class we first creating a instance of <code>ExecutorService</code> using <code>newCachedThreadPool()</code> method. We will be using threds going forward. This executor will give us a thread on demand and will resues them.

* <strong> hosts: </strong> List of backend servers (Host objects with hostnames/IPs and ports).

* <strong> currentHostIndex: </strong> Keeps track of the next backend to forward to (used for round-robin).


* <strong><code>add(Host host)</code> and <code>getNextHost()</code>:</strong> These methods are used to add new host and to get the next host in a thread safe way

### The Forwarder

In the <code>Loadbalancer</code> class we can see there is a forwarder method. This where load forwarding happens. 

```java
public void forward(Socket serverSocket) {
    executor.submit(() -> {
        Host host = getNextHost();
        ...
    });
}
```

Here we are accepting a connection and strarting a new thread to handle the connection. Inside the thread we are getting the nextHost. 

```java
    try (Socket targetSocket = new Socket(host.getHost(), host.getPort());
                 InputStream clientInputStream = serverSocket.getInputStream();
                 OutputStream clientOutputStream = serverSocket.getOutputStream();
                 InputStream targetInputStream = targetSocket.getInputStream();
                 OutputStream targetOutputStream = targetSocket.getOutputStream();) {
                    ...
                 }
```

Using ```new Socket(host.getHost(), host.getPort())``` we are connecting to the server where we want to forward the request. Then we are creating input and output streams to forward the request from client to server and return the response from  sever to client

Then we are setting up the bi-directional data transfer using two threads

```java
Future<?> upStreamFuture = executor.submit(() -> {
    copyStream(clientInputStream, targetOutputStream, "request");
});

Future<?> downStreamFuture = executor.submit(() -> {
    copyStream(targetInputStream, clientOutputStream, "response");
});
```

* upStreamFuture: Forwards data from client → target server (request).

* downStreamFuture: Forwards data from target server → client (response).

Then it waits for both to finish using get().

The copyStream(...) method Copies data from an InputStream to an OutputStream. and Logs contents of request and response

### The main method

```java
public class Main {
    public static void main(String[] args) {
        try (ServerSocket serverSocket = new ServerSocket(8888);) {
            Loadbalancer lb = new Loadbalancer();
            Host host1 = new Host("localhost", 9090);
            Host host2 = new Host("localhost", 9091);

            lb.add(host1);
            lb.add(host2);

            while (true) {
                Socket socket = serverSocket.accept();
                lb.forward(socket);
            }

        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

In the main method we are creating the instance of out Loadbalancer and ServerSocket.

* serverSocket: It acceeps connections fron client
* loadbalancer: Forward the response to server and return the response to client

### Testing the load balancer

```bash
curl --location http://localhost:8888/
curl --location http://localhost:8888/
```

output
```bash
{"message":"Hello From app 01"}
{"message":"Hello From app 02"}
```

So if we send two request first request will server from server 01 and second request will server from server 02

### Conclusion

We’ve successfully built a simple load balancer, and you can find the full code along with the app server implementations in the [repository](https://github.com/sahariardev/building-simple-load-balancer). For testing, we’ve used curl instead of a browser. This is because browsers often perform a “connection preflight” or a probe to check if the server is reachable—initiating a TCP connection without immediately sending HTTP data. After that, the actual request with headers is sent. As a result, a single hit in the browser can generate two separate requests under the hood. In the next blog post, we’ll explore how to parse both the request and the response, and modify them to include custom headers.
