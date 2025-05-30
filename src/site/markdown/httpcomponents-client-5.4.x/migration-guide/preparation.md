<!--
    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at
    
      http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.
-->

# Migration from Apache HttpClient 4.x APIs

It is strongly encouraged to follow the best practices and common use patterns in programming with Apache HttpClient
4.x. They remain largely unchanged between 4.x and 5.x release series. Correctly written code will be easier to port to
5.x APIs and from the classic I/O model to the async I/O model.

## Preparation steps

1. Make sure there are no references to deprecated functions or classes

1. HttpClient uses tries to use sensible defaults but it is generally recommended to adjust SSL/TLS parameters and
   timeout settings for the specific application context.

1. Explicitly specify TLSv1.2 in order to disable older, less secure versions of the SSL/TLS protocol. Please note
   HttpClient 4.5 disables all SSL versions by default.

1. Set finite socket and connect timeouts.

1. Set finite connection total time to live (TTL).

1. Favor the `strict` cookie policy.

1. IMPORTANT: Always re-use `CloseableHttpClient` instances. They are expensive to create, but they are also fully
   thread safe, so multiple threads can use the same instance of `CloseableHttpClient` to execute multiple requests
   concurrently taking full advantage of persistent connection re-use and connection pooling.

    ```java
    CloseableHttpClient client = HttpClients.custom()
            .setSSLSocketFactory(new SSLConnectionSocketFactory(
                    SSLContexts.createSystemDefault(),
                    new String[] { "TLSv1.2" },
                    null,
                    SSLConnectionSocketFactory.getDefaultHostnameVerifier()))
            .setConnectionTimeToLive(1, TimeUnit.MINUTES)
            .setDefaultSocketConfig(SocketConfig.custom()
                    .setSoTimeout(60000)
                    .build())
            .setDefaultRequestConfig(RequestConfig.custom()
                    .setConnectTimeout(60000)
                    .setSocketTimeout(60000)
                    .setCookieSpec(StandardCookieSpec.STRICT)
                    .build())
            .build();
    ```   

1. Cookie store and credentials providers can be shared by multiple execution threads. They can still be shared by
   multiple message exchnages without being set at
   `CloseableHttpClient` construction time.

    ```java
    CookieStore cookieStore = new BasicCookieStore();
    CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
    ```

1. While `CloseableHttpClient` should have the default configuration applicable to all all messages exchanges one can
   use `HttpContext` to customize individual request execution parameters.

    ```java
    HttpClientContext clientContext = HttpClientContext.create();
    clientContext.setCookieStore(cookieStore);
    clientContext.setCredentialsProvider(credentialsProvider);
    clientContext.setRequestConfig(RequestConfig.custom()
            .setConnectTimeout(60000)
            .setSocketTimeout(60000)
            .setCookieSpec(StandardCookieSpec.STRICT)
            .build());
    ```

1. Always set content type and content encoding on the entity and let HttpClient generate request headers automatically.

    ```java
    JsonFactory jsonFactory = new JsonFactory();
    ObjectMapper objectMapper = new ObjectMapper(jsonFactory);

    HttpPost httpPost = new HttpPost("https://httpbin.org/post");
    List<NameValuePair> requestData = Arrays.asList(
            new BasicNameValuePair("name1", "value1"),
            new BasicNameValuePair("name2", "value2"));
    
    EntityTemplate requestEntity = new EntityTemplate(outstream -> {
        objectMapper.writeValue(outstream, requestData);
        outstream.flush();
    
    });
    requestEntity.setContentType(ContentType.APPLICATION_JSON.toString());
    httpPost.setEntity(requestEntity);
    ```

1. Consume response content directly from the content stream and convert it to a higher level object without converting
   it to an intermediate string or byte array.

1. Favor HTTP response handlers for response processing, thus making connection release automatic.

    ```java
    JsonNode responseData = client.execute(httpPost, response -> {
        if (response.getStatusLine().getStatusCode() >= 300) {
            throw new ClientProtocolException(Objects.toString(response.getStatusLine()));
        }
        final HttpEntity responseEntity = response.getEntity();
        if (responseEntity == null) {
            return null;
        }
        try (InputStream inputStream = responseEntity.getContent()) {
            return objectMapper.readTree(inputStream);
        }
    });
    System.out.println(responseData);
    ```
1. `CloseableHttpClient` instances should be closed when no longer needed or about to go out of scope.

    ```java
    client.close();
    ```       
