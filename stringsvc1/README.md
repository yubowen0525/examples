- [1. 概述](#1-概述)
- [2. 业务层](#2-业务层)
  - [2.1. 业务层interface的入参与出参的思考](#21-业务层interface的入参与出参的思考)
- [3. 业务层 <-> Endpoint](#3-业务层---endpoint)
- [Endponit <-> 传输层](#endponit---传输层)
- [4. 传输层 <-> 路由层](#4-传输层---路由层)


# 1. 概述
路由层 -> 传输层 -> EndPoint -> 业务层

# 2. 业务层
业务层的上层，设计业务interface
```
type StringService interface {
	Uppercase(string) (string, error)
	Count(string) int
}

//实现
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
```

## 2.1. 业务层interface的入参与出参的思考

1. 入参可以定义一个struct，确定业务的数据结构，当然简单可以不聚合
2. 出参是业务处理完毕的struct，确定业务处理完毕的数据结构， 当然简单可以不聚合
3. 业务层的出参，一个是一个error interface。 此处可以定义自己的业务error。
   1. 可以分为业务error，和基础设施的error，分类型分error code与msg， 
   2. 此处建议携带堆栈信息github errors。


# 3. 业务层 <-> Endpoint
此处需要一个adapter，将业务层的接口适配到传输层

**需要注意Endpoint层的入参与出参**:
1. 入参: 基本就是http.request的处理，参数400 error 判断，送入svc层
2. 出参: 基本就是将svc层的struct 转入至 http.response



```
// Adapter将 svc 业务层接口转化为endpoints层接口
func makeUppercaseEndpoint(svc StringService) endpoint.Endpoint {
	return func(_ context.Context, request interface{}) (interface{}, error) {
		req := request.(uppercaseRequest)
		v, err := svc.Uppercase(req.S)
		if err != nil {
			return uppercaseResponse{v, err.Error()}, nil
		}
		return uppercaseResponse{v, ""}, nil
	}
}

func makeCountEndpoint(svc StringService) endpoint.Endpoint {
	return func(_ context.Context, request interface{}) (interface{}, error) {
		req := request.(countRequest)
		v := svc.Count(req.S)
		return countResponse{v}, nil
	}
}
```

# Endponit <-> 传输层

传输层的数据转换：
1. RPC Thrift
2. GRPC Protocol
3. HTTP -> JSON

```
// request body convert to svc struct
// rpc .... 
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

// svc response convert to response interface{} 
func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}
```


# 4. 传输层 <-> 路由层

```
// 生成一个endpoint
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

// endpoint 与 router 绑定
	http.Handle("/uppercase", uppercaseHandler)
	http.Handle("/count", countHandler)
```