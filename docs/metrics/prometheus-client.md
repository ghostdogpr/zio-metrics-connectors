---
id: prometheus-client
title: Prometheus Client
---

In a normal prometheus setup we will find prometheus agents which query configured endpoints 
at regular intervals. The endpoints are HTTP endpoints serving the current metric state in 
an encoding defined by [prometheus](https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format).

ZIO Metrics provides the Prometheus encoding for the captured metrics out of the box. To avoid enforcing 
a particular HTTP implementation, an instrumented application needs to expose the encoded format 
as an endpoint with the HTTP server of it´s choice. 

## ZIO Metrics in Prometheus 

Most of the ZIO metrics have a direct representation in the Prometheus encoding. 

### Counter

A counter is represented as a prometheus counter. 

```
# TYPE countAll counter
# HELP countAll 
countAll 460.0 1623586224730
```

### Gauge 

A gauge is represented as a prometheus gauge. 

```
# TYPE adjustGauge gauge
# HELP adjustGauge 
adjustGauge -1.2485836762095701 1623586224730
```

### Histogram 

A histogram is represented as a prometheus histogram. 

```
# TYPE myHistogram histogram
# HELP myHistogram 
myHistogram{le="0.0"} 0.0 1623586224730
myHistogram{le="10.0"} 8.0 1623586224730
myHistogram{le="20.0"} 18.0 1623586224730
myHistogram{le="30.0"} 30.0 1623586224730
myHistogram{le="40.0"} 44.0 1623586224730
myHistogram{le="50.0"} 51.0 1623586224730
myHistogram{le="60.0"} 59.0 1623586224730
myHistogram{le="70.0"} 65.0 1623586224730
myHistogram{le="80.0"} 76.0 1623586224730
myHistogram{le="90.0"} 88.0 1623586224730
myHistogram{le="100.0"} 95.0 1623586224730
myHistogram{le="+Inf"} 115.0 1623586224730
myHistogram_sum 6828.578655207023 1623586224730
myHistogram_count 115.0 1623586224730
```

### Summary 

A histogram is represented as a prometheus summary. 

```
# TYPE mySummary summary
# HELP mySummary 
mySummary{quantile="0.1",error="0.03"} 147.0 1623589839194
mySummary{quantile="0.5",error="0.03"} 286.0 1623589839194
mySummary{quantile="0.9",error="0.03"} 470.0 1623589839194
mySummary_sum 42582.0 1623589839194
mySummary_count 139.0 1623589839194
```

### Set 

A set is represented by a set of prometheus counters, distinguished from each other with an 
extra label as configured in the aspect. 

```
# TYPE mySet counter
# HELP mySet 
mySet{token="myKey-17"} 7.0 1623589839194
mySet{token="myKey-18"} 9.0 1623589839194
mySet{token="myKey-19"} 12.0 1623589839194
mySet{token="myKey-13"} 6.0 1623589839194
mySet{token="myKey-14"} 4.0 1623589839194
mySet{token="myKey-15"} 6.0 1623589839194
mySet{token="myKey-16"} 5.0 1623589839194
mySet{token="myKey-10"} 10.0 1623589839194
mySet{token="myKey-11"} 1.0 1623589839194
mySet{token="myKey-12"} 10.0 1623589839194
```

## Serving Prometheus metrics

```scala 
import zio.http._

import zio._
import zio.console._
import zio.metrics.connectors.{prometheusLayer, publisherLayer}

import sample.InstrumentedSample

val instrumentedSample = new InstrumentedSample() {}
```

ZIO Metrics connector provides a prometheus client that can be used to produce the prometheus encoded metric state 
upon request. The state is encoded in the `PrometheusPublisher` case class and the single attribute of 
type `Ref[String]` holds the prometheus encoded metric state. 

So, to retrieve the prometheus encoded state, the application can simply use `PrometheusPublisher#get` method:
```scala
val encodedContent: UIO[String] = publisher.get
```

In our example application we use [zio-http](https://github.com/zio/zio-http) to serve the metrics. Other application might choose another HTTP server framework if required.

```scala 
private val metricsConfig = ZLayer.succeed(MetricsConfig(5.seconds))

private lazy val indexPage =
  """<html>
    |<title>Simple Server</title>
    |<body>
    |<p><a href="/prometheus/metrics">Prometheus Metrics</a></p>
    |<p><a href="/insight/keys">Insight Metrics: Get all keys</a></p>
    |</body
    |</html>""".stripMargin

private lazy val static =
  Http.collect[Request] { case Method.GET -> Root => Response.html(Html.fromString(indexPage)) }

private lazy val prometheusRouter =
  Http
    .collectZIO[Request] { case Method.GET -> Root / "prometheus" / "metrics" =>
      ZIO.serviceWithZIO[PrometheusPublisher](_.get.map(Response.text))
    }
```

Now, using the HTTP server and the [instrumentation examples](instrumentation-examples.md) we can create an effect 
that simply runs the sample effects with their instrumentation. And within a `ZIO.App` we can override the run method,
which is now simply the execute method with a Prometheus client provided in it´s environment:

```scala
  private val serverInstall =
  Server.install(static ++ prometheusRouter)

private lazy val runHttp = (serverInstall *> ZIO.never).forkDaemon

private lazy val serverConfig = ZLayer.succeed(Server.Config.default.port(bindPort))

override def run: ZIO[Environment & ZIOAppArgs & Scope, Any, Any] = (for {
  f <- runHttp
  _ <- program
  _ <- f.join
} yield ())
  .provide(
    serverConfig,
    Server.live,

    // This is the general config for all backends
    metricsConfig,

    // The prometheus reporting layer
    prometheus.publisherLayer,
    prometheus.prometheusLayer,

    // Enable the ZIO internal metrics and the default JVM metricsConfig
    // Do NOT forget the .unit for the JVM metrics layer
    Runtime.enableRuntimeMetrics,
    DefaultJvmMetrics.live.unit,
  )
```

## Running the prometheus example 

Any of the examples can be run from a command line within the project checkout directory with 

```
sbt sampleApp/run
```

Out of the choices, select the option corresponding to `sample.SamplePrometheusStatsDApp`.

If everything works, we should be able to use a web browser to go to `http://localhost:8080/metrics` and should see something like 

```
# TYPE myCounter counter
# HELP myCounter
myCounter{effect="count2"} 46.0 1608982756235
# TYPE setGauge gauge
# HELP setGauge
setGauge 8.66004641453435 1608982756235
# TYPE changeGauge gauge
# HELP changeGauge
changeGauge 90.7178906485008 1608982756235
# TYPE myCounter counter
# HELP myCounter
myCounter{effect="count1"} 92.0 1608982756235
```

Once we see the metrics being served from our example, we can set up Prometheus and Grafana to create a simple dashboard displaying 
our metrics. 

## Simple Prometheus setup 

The following steps have been tested on Ubuntu 18.04 running inside the Windows Subsystem for Linux. 
Essentially, you need to download the prometheus binaries for your environment and start the server 
with our sample configuration located at 

```
${PROJECT_HOME}/examples/prometheus/promcfg.yml
``` 

This will just configure a prometheus job that regular polls `http://localhost:8080/metrics` for prometheus 
encoded metrics.

In addition, you need to download the Grafana binaries for your installation, start the Grafana server and configure 
prometheus as a single data source. 

Finally, you can import our example dashboard at `examples/prometheus/ZIOMetricsDashboard.json` and enjoy the results.

> These steps are not intended to replace the Prometheus or Grafana documentation. Please refer to their web sites 
> for guidelines towards a more sophisticated setup or an installation on different platforms. 

---

### Download and configure Prometheus 

In the steps below the project checkout directory will be referred to as `$DIR`.

1. [Download](https://github.com/prometheus/prometheus/releases/download/v2.23.0/prometheus-2.23.0.linux-amd64.tar.gz) prometheus
1. Extract the downloaded archive to a directory of your choice, this will be referred to as `$PROMDIR`. 
1. Within `$PROMDIR` execute 
   ```
   ./prometheus --config.file $DIR/examples/prometheus/promcfg.yml
   ```
   This will start the prometheus server which regularly polls the HTTP endpoint of the example above for its metrics.

### Download and configure Grafana

1. [Download](https://dl.grafana.com/oss/release/grafana-7.3.6.linux-amd64.tar.gz) grafana
1. Extract the downloaded archive to a directory of your choice, this will be referred to as `$GRAFANADIR`.
1. Within `$GRAFANADIR` execute 
   ```
   ./bin/grafana-server
   ```
   This will start a Grafana server.
1. Now you should be able to login to Grafana at `http://localhost:3000' with the default user `admin` with the default 
   password `admin`. 

   Upon the first login you will be asked to change the default password. 
1. Within the Grafana menu on the left hand side you will find `Manage Dashboards` within that page, select `Import`. 
1. You can now either install a dashboard from grafana.com or use a text field to paste JSON. 
1. Paste the content of `$DIR/examples/prometheus/ZIOMetricsDashboard.json` into the text field and select `Load`.

   This will import our dashboard example. 
1. Now, under `Manage Dashboards` the just imported ZIO dashboard should be visible. 
1. Navigate to the dashboard. 

### Grafana dashboard 

Here is a screenshot of the Grafana dashboard produced with the setup above. 

![A simple Grafana Dashboard](../img/ZIOMetrics-Grafana.png)
