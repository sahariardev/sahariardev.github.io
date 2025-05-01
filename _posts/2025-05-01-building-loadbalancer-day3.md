---
title: Building a load balancer (day three)
date: 2025-05-01 12:00:00 +600
categories: [technical]
tags: [java, loadbalancer]
---

## Parse HTTP Request and HTTP Response
Our load balancer can distribute traffic across different servers. But what if we want to modify the headers in the request or response?

Right now, it's just forwarding TCP requests and returning the raw response from the server back to the client. Our application doesn’t actually understand what’s inside those requests or responses. Now, we’re going to parse them so the application can make sense of what’s happening.

Let’s first take a look at what the request and response actually look like.

## Understanding HTTP Request

```bash
GET / HTTP/1.1
Accept: */*
User-Agent: curl/8.11.1
Host: localhost:8888
```
This is a simple example of an HTTP GET request. The first line is called the request line—it contains the method, request target, and HTTP version, all separated by spaces.

Next come the headers. Each header is on its own line and follows this format: HeaderKey: HeaderValue. After the headers, there’s a blank line, and then the body starts. In the example above, there's no body section—because the body is optional.

Lets write a parser for this

```java
public static HttpRequest parsetHttpRequest(InputStream inputStream) throws IOException {
    HttpRequest httpRequest = new HttpRequest();

    BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));

    String startLine = reader.readLine();
    if (startLine == null) {
        throw new IOException("Empty request");
    }

    StringTokenizer tokenizewr = new StringTokenizer(startLine);
    if (tokenizewr.countTokens() < 3) {
        throw new IOException("Invalid request line");
    }

    httpRequest.setMethod(tokenizewr.nextToken().toUpperCase());
    httpRequest.setUri(tokenizewr.nextToken());
    httpRequest.setHttpVersion(tokenizewr.nextToken().toUpperCase());

    String headerLine;

    while ((headerLine = reader.readLine()) != null && !headerLine.isEmpty()) {
        
        int colotnIndex = headerLine.indexOf(':');
        if (colotnIndex == -1) {
            throw new IOException("Invalid header line");
        }

        String headerName = headerLine.substring(0, colotnIndex).trim();
        String headerValue = headerLine.substring(colotnIndex + 1).trim();
        
        httpRequest.addHeader(headerName, headerValue);
    }

    if (httpRequest.getHeaders().containsKey("Content-Length")) {
        int contentLength = Integer.parseInt(httpRequest.getHeaders().get("Content-Length"));
        char[] body = new char[contentLength];
        reader.read(body, 0, contentLength);
        httpRequest.setBody(new String(body));
    }

    return httpRequest;
}
```

```java
public class HttpRequest {
    private String method;
    private String uri;
    private String httpVersion;
    private Map<String, String> headers;
    private String body;

    public HttpRequest() {
        this.headers = new HashMap<>();
    }

    public HttpRequest(String method, String uri, String httpVersion, Map<String, String> headers, String body) {
        this.method = method;
        this.uri = uri;
        this.httpVersion = httpVersion;
        this.headers = headers;
        this.body = body;
    }

    public String getMethod() {
        return method;
    }

    public void setMethod(String method) {
        this.method = method;
    }

    public String getUri() {
        return uri;
    }

    public void setUri(String uri) {
        this.uri = uri;
    }

    public String getHttpVersion() {
        return httpVersion;
    }

    public void setHttpVersion(String httpVersion) {
        this.httpVersion = httpVersion;
    }

    public Map<String, String> getHeaders() {
        return headers;
    }

    public void setHeaders(Map<String, String> headers) {
        this.headers = headers;
    }

    public String getBody() {
        return body;
    }

    public void setBody(String body) {
        this.body = body;
    }

    public void addHeader(String name, String value) {
        this.headers.put(name, value);
    }

    public byte[] toByteArray() {
        StringBuilder request = new StringBuilder();
        request.append(method).append(" ").append(uri).append(" ").append(httpVersion).append("\r\n");
        for (Map.Entry<String, String> entry : headers.entrySet()) {
            request.append(entry.getKey()).append(": ").append(entry.getValue()).append("\r\n");
        }
        request.append("\r\n");

        if (body != null) {
            request.append(body);
        }

        return request.toString().getBytes();
    }

    @Override
    public String toString() {
        return "HttpRequest [method=" + method + ", uri=" + uri + ", httpVersion=" + httpVersion + ", headers="
                + headers + ", body=" + body + "]";
    }
    
    
}
```

This request parser takes an InputStream, wraps it with a BufferedReader, and starts reading from it.

First, it reads the request line and extracts the HTTP method, URI, and version. Then it reads each header line one by one, until it hits an empty line (which indicates the end of the headers). For each header, it splits the line into a key and value using the : character, trims them, and stores them in the HttpRequest object.

After the headers, it checks if the request contains a Content-Length header. If it does, that means the request has a body. It then reads the body based on the length specified and sets it in the request object.

## Understadning HTTP Response

```bash
HTTP/1.1 200 OK
date: Thu, 1 May 2025 13:37:13 GMT
content-length: 31
content-type: application/json

{"message":"Hello From app 02"}
```
This is a sample HTTP response coming from our app server. The first line is called the status line—it contains the HTTP version, status code, and reason phrase, all separated by spaces.

Next, we have the response headers, which work just like request headers. After the headers, there’s a blank line, followed by the response body.

In this example, you’ll notice a Content-Length header. It tells us how many bytes long the response body is.

Now lets write a parser for this

```java
public static HttpResponse parseHttpResponse(InputStream inputStream) throws IOException {
    HttpResponse httpResponse = new HttpResponse();
    BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));

    String startLine = reader.readLine();
    if (startLine == null) {
        throw new IOException("Empty response");
    }

    StringTokenizer tokenizer = new StringTokenizer(startLine);
    if (tokenizer.countTokens() < 3) {
        throw new IOException("Invalid response line");
    }

    httpResponse.setHttpVersion(tokenizer.nextToken().toUpperCase());
    httpResponse.setStatusCode(Integer.parseInt(tokenizer.nextToken()));
    httpResponse.setStatusText(tokenizer.nextToken());

    String headerLine;

    while ((headerLine = reader.readLine()) != null && !headerLine.isEmpty()) {
        int colonIndex = headerLine.indexOf(':');
        if (colonIndex == -1) {
            throw new IOException("Invalid header line");
        }

        String headerName = headerLine.substring(0, colonIndex).trim();
        String headerValue = headerLine.substring(colonIndex + 1).trim();

        httpResponse.addHeader(headerName, headerValue);
    }


    if (httpResponse.getHeaders().containsKey("content-length")) {
        int contentLength = Integer.parseInt(httpResponse.getHeaders().get("content-length"));
        char[] body = new char[contentLength];
        reader.read(body, 0, contentLength);                
        httpResponse.setBody(new String(body).getBytes("UTF-8"));
    }

    return httpResponse;
}

```
```java
public class HttpResponse {
    
    private String httpVersion;

    private int statusCode;

    private String statusText;

    private final Map<String, String> headers;

    private byte[] body;

    public static final int STATUS_OK = 200;
    public static final int STATUS_NOT_FOUND = 404;
    public static final int STATUS_INTERNAL_SERVER_ERROR = 500;
    public static final int STATUS_BAD_REQUEST = 400;
    public static final int STATUS_UNAUTHORIZED = 401;  

    public HttpResponse() {
        this.httpVersion = "HTTP/1.1";
        this.statusCode = STATUS_OK;
        this.statusText = "OK";
        this.headers = new HashMap<>();
        this.body = new byte[0];
    }

    public int getStatusCode() {
        return statusCode;
    }

    public void setStatusCode(int statusCode) {
        this.statusCode = statusCode;
    }

    public String getStatusText() {
        return statusText;
    }

    public void setStatusText(String statusText) {
        this.statusText = statusText;
    }

    public void setStatusText(int statusCode) {
        this.statusText = getStatusText(statusCode);
    }
    public String getStatusText(int statusCode) {
        switch (statusCode) {
            case STATUS_OK:
                return "OK";
            case STATUS_NOT_FOUND:
                return "Not Found";
            case STATUS_INTERNAL_SERVER_ERROR:
                return "Internal Server Error";
            case STATUS_BAD_REQUEST:
                return "Bad Request";
            case STATUS_UNAUTHORIZED:
                return "Unauthorized";
            default:
                return "Unknown Status";
        }
    }

    public void setHttpVersion(String httpVersion) {
        this.httpVersion = httpVersion;
    }

    public byte[] getBody() {
        return body;
    }

    public void setBody(byte[] body) {
        this.body = body;
    }

    public void addHeader(String name, String value) {
        this.headers.put(name, value);
    }

    public Map<String, String> getHeaders() {
        return headers;
    }

    public byte[] toByteArray() {
        StringBuilder response = new StringBuilder();
        response.append(httpVersion).append(" ").append(statusCode).append(" ").append(statusText).append("\r\n");
        for (Map.Entry<String, String> header : headers.entrySet()) {
            response.append(header.getKey()).append(": ").append(header.getValue()).append("\r\n");
        }
        response.append("\r\n");
        byte[] headerBytes = response.toString().getBytes();
        byte[] fullResponse = new byte[headerBytes.length + body.length];
        System.arraycopy(headerBytes, 0, fullResponse, 0, headerBytes.length);
        System.arraycopy(body, 0, fullResponse, headerBytes.length, body.length);
        return fullResponse;
    }

    @Override
    public String toString() {
        return "HttpResponse [httpVersion=" + httpVersion + ", statusCode=" + statusCode + ", statusText=" + statusText
                + ", headers=" + headers + ", body=" + Arrays.toString(body) + "]";
    }
}
```

This code is a simple HTTP response parser. It takes an InputStream, wraps it with a BufferedReader, and starts reading from it.

First, it reads the status line—this is the first line of the response—and extracts the HTTP version, status code, and status text (like OK or Not Found). If the line is missing or invalid, it throws an exception.

Then it starts reading the headers line by line. Each header is split using :, trimmed, and stored in the HttpResponse object. It keeps reading headers until it hits an empty line, which marks the end of the headers section.

Once headers are done, it checks if there's a Content-Length header. If that’s present, it reads that many characters from the stream as the response body and stores it in the response object as a byte array.

At the end, it returns the fully parsed HttpResponse object.

## Testing the load balancer

```bash
curl --location http://localhost:8888/
curl --location http://localhost:8888/
```

output
```bash
{"message":"Hello From app 01"}
{"message":"Hello From app 02"}
```

Along with these output you will see the request and response log in the load balancer console

## Conclution

This wraps up the Building a Load Balancer from Scratch series. We've built a simple HTTP load balancer from the ground up.

There's a lot more we can do with it. For example, we can plug in different load balancing algorithms—like round robin, least connections, or IP hash. We could also add sticky session support, so the same user always gets routed to the app server they first logged into, based on session cookies or user identifiers.

To make things even faster and more scalable, we could move to non-blocking I/O with I/O multiplexing (like Java NIO or epoll). This would let us handle many connections with a small number of threads, instead of creating one thread per request. Thread-per-request is costly and doesn't scale well, so going non-blocking helps us get better performance and resource usage.

Another area to explore is TLS termination, so our load balancer can handle HTTPS traffic securely. We'd need to manage certificates and keys properly, but it’s totally doable.

And honestly, there’s a ton more we could add.

If you’ve got some crazy ideas or want to explore this further, feel free to knock—let’s dive in and hack on it together

You can find the full code [here](https://github.com/sahariardev/building-simple-load-balancer/blob/master/day03/src/main/java/org/sahariardev/Loadbalancer.java)
