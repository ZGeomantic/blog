---
layout: post
title:  "Docker笔记——api/server包分析"
date:   2016-07-01 21:30:15 +0800
categories: my article
---

## api/server包
最重要的是server.go文件，其中定义了一个Server对象：

```
type Server struct {
	cfg           *Config
	servers       []*HTTPServer
	routers       []router.Router
	authZPlugins  []authorization.Plugin
	routerSwapper *routerSwapper
}
```

简单解释下这些成员的含义：


成员变量|作用
---|---
cfg|记录APIserver的一些配置信息
servers|HTTPServer是http server和listener实例。在serveAPI()函数中其实启动的是这里的每个server
routers|server/router包下定义的Router对象数组，这里实际上是一个二维数组，即[][]router.Route，两个变量的名称只差一个字母，要小心区分：Route是真正包含着**httputils.APIFunc**处理逻辑的对象，Router是一组Route。<br>这个设计和命名与golang官方http包的设计是一致的。这个变量是为了暂存外部传入的变量，最终赋给routerSwapper。详见**InitRouter**和**createMux**函数。
authZPlugins | 自定义的安全验证插件？？
routerSwapper| 负责router（类型为***mux.Router**）的赋值，它本身以及它的成员router都是标准包中http.HttpHandler接口的实现，这在下面serveAPI()和



成员的初始化顺序大概是这样：
routers -> routerSwapper -> servers。

下面看几个**Server**上所绑定的比较重要的函数：


#### 1. 初始化s.routers、s.routerSwapper
主要做一件事：初始化**主路由**并赋给s.routerSwapper的router变量。

代码入如下，分析在注释中。

```go
// InitRouter initializes the list of routers for the server.
// This method also enables the Go profiler if enableProfiler is true.
func (s *Server) InitRouter(enableProfiler bool, routers ...router.Router) {

	// 首先，把传入的router.Route[]类型的变量赋给s.routers，这些是外部注册好的路由对象。
	for _, r := range routers {
		s.routers = append(s.routers, r)
	}

	// 由s.routers构造出*mux.Router对象，目的是赋给s.routerSwapper。
	m := s.createMux()
	if enableProfiler {
		profilerSetup(m)
	}
	s.routerSwapper = &routerSwapper{
		router: m,
	}
}

// createMux initializes the main router the server uses.
func (s *Server) createMux() *mux.Router {
	m := mux.NewRouter()

	logrus.Debugf("Registering routers")
	for _, apiRouter := range s.routers {
		for _, r := range apiRouter.Routes() {
		
			// 这里的r.Hanlder()获得的实际上是httputils.APIFunc类型，需要转换成http.Handler
			f := s.makeHTTPHandler(r.Handler())

			// 全部注册到返回值*mux.Router中
			logrus.Debugf("Registering %s, %s", r.Method(), r.Path())
			
			// github.com/gorilla/mux包的Handler注册函数只接受实现了http.Handler接口的函数
			m.Path(versionMatcher + r.Path()).Methods(r.Method()).Handler(f)
			m.Path(r.Path()).Methods(r.Method()).Handler(f)
		}
	}

	return m
}
```



为什么上面的createMux()中需要一层转换，因为s.routers对象是[]router.Router类型（在api/server/router中）。
尤其是Route接口中的Handler()函数的返回类型httputils.APIFunc，其本身是为了方便docker内部使用而定义，任何实现了httputils.APIFunc的接口，都可以注册成为docker的API endpoint。源码定义如下：

```go
// Router defines an interface to specify a group of routes to add to the docker server.
type Router interface {
	// Routes returns the list of routes to add to the docker server.
	Routes() []Route
}

// Route defines an individual API route in the docker server.
type Route interface {
	// Handler returns the raw function to create the http handler.
	Handler() httputils.APIFunc
	// Method returns the http method that the route responds to.
	Method() string
	// Path returns the subpath where the route responds to.
	Path() string
}
```
可以看到其中的Hanlder返回的并不是标准的http.Handler接口，而是docker自己定义的接口httputils.APIFunc，但这个是请求处理业务的代码部分，所以需要转换后，才能注册到gorilla/mux包下的*mux.Router对象中。



#### 2 初始化s.servers
初始化**s.servers**，创建所有的server。
根据地址和监听对象，依次创建**HTTPServer**对象，赋给对象的servers成员。

```go
// Accept sets a listener the server accepts connections into.
func (s *Server) Accept(addr string, listeners ...net.Listener) {
	for _, listener := range listeners {
		httpServer := &HTTPServer{
			srv: &http.Server{
				Addr: addr,
			},
			l: listener,
		}
		s.servers = append(s.servers, httpServer)
	}
}
```

#### 3 启动所有的server
依次启动**s.servers**中的每个HTTPServer，把**s.routerSwapper**赋给这些执行的server。并监控是否每个服务都正常的运行、关闭。

>注意：之所以可以把 **s.routerSwapper** 赋给 **s.servers**的Hanlder ，是因为它实现了
	http.HttpHandler接口.

```go
// serveAPI loops through all initialized servers and spawns goroutine
// with Server method for each. It sets createMux() as Handler also.
func (s *Server) serveAPI() error {
	var chErrors = make(chan error, len(s.servers))
	for _, srv := range s.servers {
		srv.srv.Handler = s.routerSwapper
		go func(srv *HTTPServer) {
			var err error
			logrus.Infof("API listen on %s", srv.l.Addr())
			// 这里很有意思，对于用到了关闭的网络连接的地方，是不算作错误信息的
			if err = srv.Serve(); err != nil && strings.Contains(err.Error(), "use of closed network connection") {
				err = nil
			}
			chErrors <- err
		}(srv)
	}
	
	// 一直收集管道中传来的信息，如果有错误，就立刻中断程序流程
	for i := 0; i < len(s.servers); i++ {
		err := <-chErrors
		if err != nil {
			return err
		}
	}

	return nil
}
```

上面的逻辑是启动sever，但它不是一个公有的方法，所以真正的启动函数肯定在其他地方,实际上是：

```go
func (s *Server) Wait(waitChan chan error) {
	if err := s.serveAPI(); err != nil {
		logrus.Errorf("ServeAPI error: %v", err)
		waitChan <- err
		return
	}
	waitChan <- nil
}
```

## 与api/server/router的交互

#### DaemonCli
在docker/deamon.go中，是初始化、启动Server对象的地方，主要逻辑有，

- 根据命令参数的配置，[初始化httpserver的监听地址和listener](https://github.com/docker/docker/blob/v1.11.1/docker/daemon.go#L240)：
- [初始化路由](https://github.com/docker/docker/blob/v1.11.1/docker/daemon.go#L292)，具体的逻辑在下面会解释。
- [启动sever](https://github.com/docker/docker/blob/v1.11.1/docker/daemon.go#L319)，[中断信号捕捉](https://github.com/docker/docker/blob/v1.11.1/docker/daemon.go#L321)，[调试模式开启](https://github.com/docker/docker/blob/v1.11.1/docker/daemon.go#L302)。

初始化路由的这个逻辑，是**api/server**包与**api/server/router**重要的交汇，两个包的初始化逻辑在这里串联起来，[代码如下](https://github.com/docker/docker/blob/v1.11.1/docker/daemon.go#L408)。

```go
func initRouter(s *apiserver.Server, d *daemon.Daemon) {
	
	// router包下的每个包的初始化，依赖的参数是从这里传入的*daemon.Daemon，
	// 具体的url已经在router包里注册好了
	routers := []router.Router{
		container.NewRouter(d),
		image.NewRouter(d),
		systemrouter.NewRouter(d),
		volume.NewRouter(d),
		build.NewRouter(dockerfile.NewBuildManager(d)),
	}
	if d.NetworkControllerEnabled() {
		routers = append(routers, network.NewRouter(d))
	}
	
	//  *apiserver.Server把Router包中的注册信息转换成主路由，并保存到routerSwapper中作为主路由
	s.InitRouter(utils.IsDebugEnabled(), routers...)
}
``` 




