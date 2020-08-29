+++
title = "Scala and AWS managed ElasticSearch"
description = "Details how to use Elastic4s with AWS managed ElasticSearch"
keywords = [
 "scala", "aws", "elasticsearch", "iam", "authentication", "signing", "elastic4s"
]
categories = [
 "programming"
]
date = "2017-02-26T18:06:45Z"

+++

[AWS](https://aws.amazon.com/) offer a [managed ElasticSearch service](https://aws.amazon.com/elasticsearch-service/). It exposes an HTTP endpoint for interacting with ElasticSearch and requires authentication via [AWS Identity Access Management](https://aws.amazon.com/documentation/iam/).

[Elastic4s](https://github.com/sksamuel/elastic4s) offers a neat DSL and Scala client for ElasticSearch. This post details how to use it with [AWS's](https://aws.amazon.com/) [managed ElasticSearch service](https://aws.amazon.com/elasticsearch-service/).
## Creating a request signer

Using the [aws-signing-request-interceptor](https://github.com/inreachventures/aws-signing-request-interceptor) library its easy to create an [HttpRequestInterceptor](https://hc.apache.org/httpcomponents-core-ga/httpcore/apidocs/org/apache/http/HttpRequestInterceptor.html) which can be later added to the HttpClient used by [Elastic4s](https://github.com/sksamuel/elastic4s) for making the calls to ElasticSearch

```scala
private def createAwsSigner(config: Config): AWSSigner = {
  import com.gilt.gfc.guava.GuavaConversions._

  val awsCredentialsProvider = new DefaultAWSCredentialsProviderChain
  val service = config.getString("service")
  val region = config.getString("region")
  val clock: Supplier[LocalDateTime] = () => LocalDateTime.now(ZoneId.of("UTC"))
  new AWSSigner(awsCredentialsProvider, region, service, clock)
}
```

## Creating an HTTP Client and intercepting the requests

The ElasticSearch [RestClientBuilder](https://github.com/elastic/elasticsearch/blob/master/client/rest/src/main/java/org/elasticsearch/client/RestClientBuilder.java#L230) allows for registering a callback to modify the customise the [HttpAsyncClientBuilder](http://hc.apache.org/httpcomponents-asyncclient-dev/httpasyncclient/apidocs/org/apache/http/impl/nio/client/HttpAsyncClientBuilder.html#addInterceptorLast(org.apache.http.HttpResponseInterceptor)) enabling registering the interceptor to sign the requests.

The callback can be created by implementing the HttpClientConfigCallback interface as follows:

```scala
private val esConfig = config.getConfig("elasticsearch")

private class AWSSignerInteceptor extends HttpClientConfigCallback {
  override def customizeHttpClient(httpClientBuilder: HttpAsyncClientBuilder): HttpAsyncClientBuilder = {
    httpClientBuilder.addInterceptorLast(new AWSSigningRequestInterceptor(createAwsSigner(esConfig)))
  }
}
```

Finally, an [Elastic4s](https://github.com/sksamuel/elastic4s) client can be created with the interceptor registered:

```scala
private def createEsHttpClient(config: Config): HttpClient = {
  val hosts = ElasticsearchClientUri(config.getString("uri")).hosts.map {
    case (host, port) =>
      new HttpHost(host, port, "http")
  }

  log.info(s"Creating HTTP client on ${hosts.mkString(",")}")

  val client = RestClient.builder(hosts: _*)
    .setHttpClientConfigCallback(new AWSSignerInteceptor)
    .build()
  HttpClient.fromRestClient(client)
}
```

Full Example on [GitHub](https://github.com/imduffy15/scala-aws-hosted-es/blob/master/src/main/scala/Run.scala)

