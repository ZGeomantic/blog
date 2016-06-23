---
layout: post
title:  "docker源码分析——v1.2与v1.11对比"
date:   2016-06-20 15:32:28 +0800
categories: my article
---
分析一下docker/api/server/router包下面的逻辑调整。

可以肯定的一点是，与v1.2相比，v1.11已经取消了根据cmd参数，通过反射来调用相应函数的方式。现在统一的用Backend来统一接口，直接注册到路由中，不同再创建、运行job，可能连Engine都不需要了。

在api/server/router下的每个子目录中，都有一样的代码组织结构：


	-- foo 	
	  --| backend.go 	
	  --| foo.go 		
	  --| foo_routes.go	
  

上面的foo代表router目录下的任一个子目录，对外提供的docker命令行操作api，比如[build][build],container等，foo目录下的具体文件虽然有不同，但是一定会有上面列举出来的三个文件。其中

- 每个backend.go中都定义了一个名为Backend接口(interface)，其中的函数对应的所有功能的接口列表。
- 与foo包名相同的foo.go文件，定义了一个名为fooRouter的对象，主要是注册一些http的handler
- foo_routes.go文件，可以看做是foo.go文件的扩展，为fooRouter对象绑定了那些注册router时要用到的的httputils.APIFunc,基本与backend.go中的函数一一对应。

--------



### 下面以最常用的docker pull命令为例，具体分析api/server/router包的代码结构：

在/docker/api/server/router/image/backend.go:L41，有这样的一个接口定义:
    
{% highlight go %}
type registryBackend interface {
    PullImage(ctx context.Context, ref reference.Named, metaHeaders map[string][]string, authConfig *types.AuthConfig, outStream io.Writer) error
   
}
{% endhighlight %}

其中的PullImage就是docker pull命令对应的将要注册到router中的函数，具体实现在哪儿呢？这个待会儿解释，我们先看这个函数在哪里被调用,见（/docker/api/server/router/image/image_routes.go:L75-130）

    func (s *imageRouter) postImagesCreate(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
    ...
        err = s.backend.PullImage(ctx, ref, metaHeaders, authConfig, output)
    ...
    }

这里的backend是在创建imageRouter这个结构体(/docker/api/server/router/image/image.go:L6）的时候，从外界传入的。其实上面提到backend.go文件中定义的接口，只是提供了一套规范，并没有在当前的package下实现，而是在外部被调用的时候才传入具体的实现（比如daemon包下的Daemon对象绑定了很多函数，都是这些backend的实现，见源码/docker/daemon/daemon.go:L1038即是PullImage的具体实现）。这样的设计非常赞，让server/router的代码功能划分非常清晰。

接下来，我们看一下postImagesCreate这个函数是在哪里被注册到router中的，见/docker/api/server/router/image/image.go:L37：
	
	func (r *imageRouter) initRoutes() {
	...
		router.NewPostRoute("/images/create", r.postImagesCreate),
	...
	}

现在，整个结构的思路应该清晰了：

1. router目录下的每个包都在backend.go声明了一套Backend接口，让外界提供实现，
2. router目录下的每个包都有一个fooRouter结构体，负责路由的注册（foo.go）和注册函数的实现（foo_routes.go），其中注册函数的实现中，调用了Backend接口。

其实api.server.router包下的逻辑基本就这么简单。如果有比较认真的同学，可能还会问Backend的实现究竟是什么？怎么样传入的呢？怎么证明是daemon.Daemon这个结构体呢？可以看看/docker-bump_v1.11.1/docker/daemon.go:L408-421这段代码：


{% highlight go %}
func initRouter(s *apiserver.Server, d *daemon.Daemon) {
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

	s.InitRouter(utils.IsDebugEnabled(), routers...)
}
{% endhighlight %}

这是在daemon初始化的时候执行的代码，其中的foo.NewRouter所有的创建代码都是指向刚刚我们分析的逻辑。至此，router包分析完毕。


[build]: http://jekyllrb.com/docs/home