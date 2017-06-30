## Go Kit Service

####  业务逻辑

将业务逻辑封装成接口

```go
// StringService 提供字符串的操作方式
type StringService interface {
	Uppercase(string) (string, error)
	Count(string) int
}
```

实现上述接口，即实现具体的功能

```go
type stringService struct{}

func (stringService) Uppercase(s string) (string, error) {
	if s == "" {
		return "", ErrEmpty
	}
	return strings.ToUpper(s), nil
}

func (stringService) Count(s string) int {
	return len(s)
}

var ErrEmpty = errors.New("Empty string")
```

#### 请求与响应

go-kit 中基本的消息机制是RPC，因此，接口中的每一个方法都应该封装成一个远程访问模块，对于每一个方法需定义`request` 和 `response` 结构用于捕获各自输入输出的参数。

```go
type uppercaseRequest struct {
	S string `json:"s"`
}

type uppercaseResponse struct {
	V   string `json:"v"`
	Err string `json:"err,omitempty"` // error 无法使用json格式化，因此使用string
}

type countRequest struct {
	S string `json:"s"`
}

type countResponse struct {
	V int `json:"v"`
}
```

#### Endpoints （端点）
Endpoint 的结构

```go
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```

endpoint 相当于一个单一的RPC. 也就是服务接口中的一个函数. 可以通过写一个简单的适配器把每一个服务方法转化为endpoint. 每一个适配器适配一个服务，并返回一个endpoint 和对应的方法通信

```go
import (
	"golang.org/x/net/context"
	"github.com/go-kit/kit/endpoint"
)

func makeUppercaseEndpoint(svc StringService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(uppercaseRequest) // 传入请求结构
		v, err := svc.Uppercase(req.S) // 业务
		if err != nil { 
			return uppercaseResponse{v, err.Error()}, nil // 返回响应结构
		}
		return uppercaseResponse{v, ""}, nil // 返回响应结构
	}
}

func makeCountEndpoint(svc StringService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(countRequest)
		v := svc.Count(req.S)
		return countResponse{v}, nil
	}
}
```

#### transports（传输）

为了服务能被外界访问，需要将服务暴露出去

```go
import (
	httptransport "github.com/go-kit/kit/transport/http"
)

//通过以下过程生成 handler
uppercaseHandler := httptransport.NewServer(
		makeUppercaseEndpoint(svc),
		decodeUppercaseRequest,
		encodeResponse,
)

/**
 * func NewServer 的参数：
 *   endpoint.Endpoint：type Endpoint func(ctx context.Context, request interface{})
(response interface{}, err error)
 *   DecodeRequestFunc：type DecodeRequestFunc func(context.Context, *http.Request)
(request interface{}, err error)
 *   EncodeResponseFunc：type EncodeResponseFunc func(context.Context, http.ResponseWriter, interface{}) error
 * ...
 */
```

业务实现：

```go
import (
	"encoding/json"
	"log"
	"net/http"

	"golang.org/x/net/context"

	httptransport "github.com/go-kit/kit/transport/http"
)

func main() {
	svc := stringService{}

	uppercaseHandler := httptransport.NewServer(
		makeUppercaseEndpoint(svc),
		decodeUppercaseRequest,
		encodeResponse,
	)

	countHandler := httptransport.NewServer(
		makeCountEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)

	http.Handle("/uppercase", uppercaseHandler)
	http.Handle("/count", countHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func decodeUppercaseRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request uppercaseRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func decodeCountRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request countRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}
```

### 中间件

没有日志系统和监测系统的系统不是好服务

#### logging （日志）

任何组件都需要像对待依赖一样对待日志，如数据库连接。因此在`func main`中实例化logger组件，可以在stringService的实现中直接实现日志组件，但使用中间件是最好的选择，中间件就是一个处理endpoint并会返回一个endpoint的函数。

```go
type Middleware func(Endpoint) Endpoint
```

在中间件中能做任何事

```go
func loggingMiddleware(logger log.Logger) Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			logger.Log("msg", "calling endpoint")
			defer logger.Log("msg", "called endpoint")
			return next(ctx, request)
		}
	}
}
```

在所有handler中配置这个中间件

```go
logger := log.NewLogfmtLogger(os.Stderr)

svc := stringService{}

var uppercase endpoint.Endpoint
uppercase = makeUppercaseEndpoint(svc)
uppercase = loggingMiddleware(log.NewContext(logger).With("method", "uppercase"))(uppercase)

var count endpoint.Endpoint
count = makeCountEndpoint(svc)
count = loggingMiddleware(log.NewContext(logger).With("method", "count"))(count)

uppercaseHandler := httptransport.Server(
	// ...
	uppercase,
	// ...
)

countHandler := httptransport.Server(
	// ...
	count,
	// ...
)
```

要是需要在应用中记录日志，比如是否有参数传入。为服务定义一个中间件也可以得到同样良好的结果. 从StringService 被定义为一个接口开始就应该用一个新的接口将 StringService 包裹，再实现各自的日志规范

```go
type loggingMiddleware struct {
	logger log.Logger
	next   StringService
}

func (mw loggingMiddleware) Uppercase(s string) (output string, err error) {
	defer func(begin time.Time) {
		mw.logger.Log(
			"method", "uppercase",
			"input", s,
			"output", output,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())

	output, err = mw.next.Uppercase(s)
	return
}

func (mw loggingMiddleware) Count(s string) (n int) {
	defer func(begin time.Time) {
		mw.logger.Log(
			"method", "count",
			"input", s,
			"n", n,
			"took", time.Since(begin),
		)
	}(time.Now())

	n = mw.next.Count(s)
	return
}
```

```go
import (
	"os"

	"github.com/go-kit/kit/log"
	httptransport "github.com/go-kit/kit/transport/http"
)

func main() {
	logger := log.NewLogfmtLogger(os.Stderr)

	var svc StringService
	svc = stringsvc{}
	svc = loggingMiddleware{logger, svc}

	// ...

	uppercaseHandler := httptransport.NewServer(
		// ...
		makeUppercaseEndpoint(svc),
		// ...
	)

	countHandler := httptransport.NewServer(
		// ...
		makeCountEndpoint(svc),
		// ...
	)
}
```

* 在传输域中使用endpoint 中间件，用于监测断开、速率限制等
* 在业务域中使用服务中间件，如日志、监测等

#### instrumentation

instrumentation 意味着使用 `metrics` 包进行记录统计服务运行期间的行为。统计工作进程数量、记录请求时间、记录所有被认为需要记录的事件。

```go
type instrumentingMiddleware struct {
	requestCount   metrics.Counter
	requestLatency metrics.TimeHistogram
	countResult    metrics.Histogram
	next           StringService
}

func (mw instrumentingMiddleware) Uppercase(s string) (output string, err error) {
	defer func(begin time.Time) {
		methodField := metrics.Field{Key: "method", Value: "uppercase"}
		errorField := metrics.Field{Key: "error", Value: fmt.Sprintf("%v", err)}
		mw.requestCount.With(methodField).With(errorField).Add(1)
		mw.requestLatency.With(methodField).With(errorField).Observe(time.Since(begin))
	}(time.Now())

	output, err = mw.next.Uppercase(s)
	return
}

func (mw instrumentingMiddleware) Count(s string) (n int) {
	defer func(begin time.Time) {
		methodField := metrics.Field{Key: "method", Value: "count"}
		errorField := metrics.Field{Key: "error", Value: fmt.Sprintf("%v", error(nil))}
		mw.requestCount.With(methodField).With(errorField).Add(1)
		mw.requestLatency.With(methodField).With(errorField).Observe(time.Since(begin))
		mw.countResult.Observe(int64(n))
	}(time.Now())

	n = mw.next.Count(s)
	return
}
```

```go
import (
	stdprometheus "github.com/prometheus/client_golang/prometheus"
	kitprometheus "github.com/go-kit/kit/metrics/prometheus"
	"github.com/go-kit/kit/metrics"
)

func main() {
	logger := log.NewLogfmtLogger(os.Stderr)

	fieldKeys := []string{"method", "error"}
	requestCount := kitprometheus.NewCounter(stdprometheus.CounterOpts{
		// ...
	}, fieldKeys)
	requestLatency := metrics.NewTimeHistogram(time.Microsecond, kitprometheus.NewSummary(stdprometheus.SummaryOpts{
		// ...
	}, fieldKeys))
	countResult := kitprometheus.NewSummary(stdprometheus.SummaryOpts{
		// ...
	}, []string{}))

	var svc StringService
	svc = stringService{}
	svc = loggingMiddleware{logger, svc}
	svc = instrumentingMiddleware{requestCount, requestLatency, countResult, svc}

	// ...

	http.Handle("/metrics", stdprometheus.Handler())
}
```

### 访问其他服务

实际业务中往往需要调用其他已有的服务，go kit 提供了传输中间件来解决这个问题

```go
type proxymw struct {
	ctx       context.Context
	next      StringService    
	uppercase endpoint.Endpoint 
}
```

#### 客户端端点

但需要调用一个已知的端点，无论是服务或是请求时叫做 `client endpoint` 

```go
func (mw proxymw) Uppercase(s string) (string, error) {
	response, err := mw.uppercase(mw.Context, uppercaseRequest{S: s})
	if err != nil {
		return "", err
	}
	resp := response.(uppercaseResponse)
	if resp.Err != "" {
		return resp.V, errors.New(resp.Err)
	}
	return resp.V, nil
}
```

接下来实例化这个代理中间件，将url 字符串转化为endpoint，需要HTTP 协议请求json时可以使用 `transport/http` 包

```go
import (
	httptransport "github.com/go-kit/kit/transport/http"
)

func proxyingMiddleware(proxyURL string, ctx context.Context) ServiceMiddleware {
	return func(next StringService) StringService {
		return proxymw{ctx, next, makeUppercaseEndpoint(ctx, proxyURL)}
	}
}

func makeUppercaseEndpoint(ctx context.Context, proxyURL string) endpoint.Endpoint {
	return httptransport.NewClient(
		"GET",
		mustParseURL(proxyURL),
		encodeUppercaseRequest,
		decodeUppercaseResponse,
	).Endpoint()
}
```

#### 服务发现和负载均衡

假如只有一个单一的远程服务的话，以上处理方式是不会有问题的，但现实中往往有许多服务实例可供获取，因此需要通过一些服务发现机制去获取这些服务，并且把这些任务分担到所有这些服务上。如果这些实例中的任何一个表现不良，也需要在不影响我们服务的可靠性的情况下处理它。

go-kit 针对不同服务发现系统提供适配器去获取最新的实例集合，并作为单个端点导出，这种适配器叫做subscribers（订阅）

```go
type Subscriber interface {
  Endpoints()  ([]endpoint.EndPoint, error)
}
```

在内部，subscribers 使用一个已提供的工厂函数转换每一个已发现的实例（典型的：host:port）进入endpoint

```go
type Factory func(instance string) (endpoint.Endpoint, error)
```

到目前为止，工厂函数makeUppercaseEndpoint直接访问url，但是调用一些安全相关的中间件进入服务也是非常重要的，如服务断开、速率限制等

```go
var e endpointEndpoint
e = makeUppercaseProxy(ctx, instance)
e = circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{}))(e)
e = kitratelimit.NewTokenBucketLimiter(jujuratelimit.NewBucketWithRate(float64(maxQPS), int64(maxQPS)))(e)
```

现在已获取了endpoint的集合，接下来就要去选择，用Load balances(负载均衡) 包含subscribers并从众多endpoint中选择一个。Go kit 提供了一对基础的负载均衡器

```go
type Balancer interface {
  Endpoint() (endpoint.Endpoint, error)
}
```

现在已可以通过一些heuristic（启发式）选择endpoint，可以用它提供一个单一的、逻辑性强、健壮的endpoint给消费者。重试策略包含负载均衡器，并返回可用的endpoint。重试的策略会重试错误的请求直到极限的请求或超时时间已到

```go
func Retry(max int, timeout time.Duration, lb Balancer) endpoint.Endpoint
```

连接最终的代理中间件。为了简单起见，假设用户将用一个标志指定多个逗号分隔的实例端点。

```go
func proxyingMiddleware(instances string, ctx context.Context, logger log.Logger) ServiceMiddleware {
  //加入实例为空，则不代理
  if instances == "" {
    logger.Log("proxy_to", "none")
    return func(next StringService) StringService { return next }
  }
  
  //为client端设置参数
  var (
  	gps          =     100    // 
    maxAttempts  =     3      // 每个请求等待的最长时间
    maxTime      =     250 * time.Milliscond // wallclock time, before giving up
  )
  
  // 为列表中的每个实例实例化一个端点，并且将它添加到一组固定的端点。在现实的服务器上，不管是否手动做以上
  // 操作都应该在服务发现系统中使用sd包的支持
  var (
  	instanceList = split(instances)
    subscriber    sd.FixedSubscriber
  )
  logger.Log("proxy_to", fmt.Sprint(instanceList))
  for _, instance := range instanceList {
    var e endpoint.Endpoint
    e = makeUppercaseProxy(ctx, instance)
    e = circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{}))(e)
    e = kitratelimit.NewTokenBuckerLimiter(jujuratelimit.NewBucketWithRate(float64(qps), int64(qps)))(e)
    subscriber = append(subscriber, e)
  }
  
  // 在所有的单独的端点中创建一个单一的、可重试的、负载均衡的端点
  balancer := lb.NewRoundRobin(subscriber)
  retry := lb.Retry(maxAttempts, maxTime, balancer)
  
  // 最后，返回这个服务中间件，并通过proxymw实现
  return func(next StringService) StringService {
    return proxymw{ctx, next, retry}
  }
}
```
