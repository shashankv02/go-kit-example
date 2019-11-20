# go-kit example

## Get go-kit
go get github.com/go-kit/kit

## Implementing a service

1. Define the service interface

2. Implement the service interface

3. Create the request reponse structs for all service menthods

4. Create the endpoints for all service menthods. Create the endpoint creator function which
takes the service as input and returns an endpoin closure.

An endpoint is of type `func(ctx context.Context, request interface{}) (response interface{}, err error) {`

5. Create RequestDecoder and ResponseEncoder functions for all request and response structs tp convert
user domain objects to http Request and Response objects
```
type DecodeRequestFunc func(context.Context, *http.Request) (request interface{}, err error)
type EncodeResponseFunc func(context.Context, http.ResponseWriter, response interface{}) error
```

6. Create the HTTP handlers using "github.com/go-kit/kit/transport/http" package.

```
countHandler := httptransport.NewServer(
		makeCountEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)
```

7. Use any http package to start the NewServer

```
http.Handle("/uppercase", uppercaseHandler)
	http.Handle("/count", countHandler)
	http.ListenAndServe(":8080", nil)
```

## Middleware/Decorator

There are two types of middleware - Endpoint middleware or service middleware.

For transport domain concerns like circuit breaking and rate limiting, use endpoint middleware.
For bussiness domain concerns like logging, instrumenting, use service middlewares.

### Intrumenting

To record runtime statistics of the service's behaviour, use "github.com/go-kit/kit/metrics"
The middleware should implement the service API. The middleware will compose the
service implementation and pass the request handling to the actual service implementation.

```
import "github.com/go-kit/kit/metrics"

type instrumentingMiddleware struct {
	requestCount   metrics.Counter
	requestLatency metrics.Histogram
	countResult    metrics.Histogram
	next           StringService
}

func (mw instrumentingMiddleware) Uppercase(s string) (v string, err error) {
	defer func(begin time.Time) {
		lvs := []string{"method", "uppercase", "error", fmt.Sprint(err != nil)}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
	}(time.Now())
	v, err = mw.next.Uppercase(s)
	return
}
```

"github.com/go-kit/kit/metrics/prometheus" package provides a wrapper on standard prometheus
client to create various metrics.

```
fieldKeys := []string{"method", "error"}
requestCount := kitprometheus.NewCounterFrom(stdprometheus.CounterOpts{
		Namespace: "my_group",
		Subsystem: "string_service",
		Name:      "request_count",
		Help:      "Number of requests",
	}, fieldKeys)
```

Expose the /metrics endpoint using "github.com/prometheus/client_golang/prometheus/promhttp"
```
http.Handle("/metrics", promhttp.Handler())
```

### Start prometheus server:
#### config
```
$ cat /tmp/prom/config.yml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: prometheus
    static_configs:
     # - targets: ['localhost:8080']
       - targets: ['host.docker.internal:8080']  # for mac
```

```
$ docker run  -p 9090:9090 -v /tmp/prom/config.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

Go to prometheus server and see the metrics!


## TODO
# Rate limiting
# Distributed tracing
