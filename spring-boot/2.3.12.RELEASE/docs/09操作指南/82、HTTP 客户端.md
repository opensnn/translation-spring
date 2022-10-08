# 82、HTTP 客户端

Spring Boot 提供了许多与 HTTP 客户端一起使用的 starters。本节回答与使用它们相关的问题。

## 82.1、配置 RestTemplate 以使用代理

如[第 15.1 节：RestTemplate 定制](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-resttemplate-customization)中所述，你可以使用 RestTemplateCustomizer 和 RestTemplateBuilder 来构建定制的 RestTemplate。这是创建配置为使用代理的 RestTemplate 的推荐方法。

代理配置的确切细节取决于正在使用的底层客户端请求工厂。以下示例使用 HttpClient 配置 HttpComponentsClientRequestFactory，该 HttpClient 为除 192.168.0.5 之外的所有主机使用代理：
```
static class ProxyCustomizer implements RestTemplateCustomizer {

    @Override
    public void customize(RestTemplate restTemplate) {
        HttpHost proxy = new HttpHost("proxy.example.com");
        HttpClient httpClient = HttpClientBuilder.create().setRoutePlanner(new DefaultProxyRoutePlanner(proxy) {

            @Override
            public HttpHost determineProxy(HttpHost target, HttpRequest request, HttpContext context)
                    throws HttpException {
                if (target.getHostName().equals("192.168.0.5")) {
                    return null;
                }
                return super.determineProxy(target, request, context);
            }

        }).build();
        restTemplate.setRequestFactory(new HttpComponentsClientHttpRequestFactory(httpClient));
    }

}
```

## 82.2、配置基于 Reactor Netty 的 WebClient 使用的 TcpClient

当 Reactor Netty 在类路径上时，会自动配置一个基于 Reactor Netty 的 WebClient。 要自定义客户端对网络连接的处理，请提供 ClientHttpConnector bean。 以下示例配置了 60 秒的连接超时并添加了 ReadTimeoutHandler：
```
@Bean
ClientHttpConnector clientHttpConnector(ReactorResourceFactory resourceFactory) {
    TcpClient tcpClient = TcpClient.create(resourceFactory.getConnectionProvider())
            .runOn(resourceFactory.getLoopResources()).option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 60000)
            .doOnConnected((connection) -> connection.addHandlerLast(new ReadTimeoutHandler(60)));
    return new ReactorClientHttpConnector(HttpClient.from(tcpClient));
}
```

提示：注意 ReactorResourceFactory 用于连接提供者和事件循环资源。 这确保了接收请求的服务器和发出请求的客户端有效地共享资源。

