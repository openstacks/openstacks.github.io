---
layout: post
title:  "Openstack RPC 服务原理解析"
categories: Openstack 
tags:  Openstack RPC Service Manager
---

* content
{:toc}

本文通过实例来阐述Openstack中RPC Service的实现原理和方法。





###  Openstack中的服务

编程模式从面向过程，面向对象，演变为现在的面向服务。openstack模块提供两种服务：一个是Rest服务，提供Rest API；另一个是RPC服务，提供RPC API。    

如下图所示，如果是跨项目的服务调用(如nova调用keystone)，使用rest api。如果是项目内不同模块之间的服务调用,则使用RPC API调用(如nova-conductor和nova-compute之间)。

  > ![](/assets/restrpc.png) 


#### Rest API    
     
 Rest API是openstack的标准对外接口。
 
openstack中的rest api实现有两个关键技术：wsgi和Paste Deployment。

- wsgi是pthon中的一个接口规范，定义web server和web application之间的接口。openstack中的rest API 实现都遵循了这个规范。[rest API规范](http://legacy.python.org/dev/peps/pep-0333/)

- Paste Deployment是一个针对wsgi开发的库，用来配置和加载wsgi application和server。openstack中配置都是通过api-paste.ini文件提供。通过这个文件就可以直接调用Paste Deployment代码来加载web server和上面的application。[Past Deployment规范](http://pythonpaste.org/deploy/)
 
   
 
#### RPC API
 
 RPC即Remote Procedure Call(远程方法调用)，是Openstack中一种用来实现跨进程(或者跨机器)的通信机制。    
 Openstack中同项目的各服务通过RPC实现彼此间通信。比如nova项目的nova-conductor和nova-compute之间就通过RPC进行通信。
  
  > ![](/assets/rabt.png) 
 
#### 如何实现一个PRC服务 

 下面以nova-compute为例，演示如何构建一个PRC service。
 
 - 在cmd目录下定义console启动脚本，其中的main函数如下：cmd/compute.py
  {% highlight python %}
  def main():
    config.parse_args(sys.argv)
    logging.setup(CONF, 'nova')
    
    server = service.Service.create(binary='nova-compute',
                                    topic=CONF.compute_topic,
                                    db_allowed=CONF.conductor.use_local)
    service.serve(server)
    service.wait()

  {% endhighlight %}
 
 - 在compute/manager.py的ComputeManager类里定义RPC方法
 
   比如我们定义一个startup VM的方法
   
  {% highlight python %}
  
def start_instance(self, context, instance):
        """Starting an instance on this host."""

  {% endhighlight %} 

- 在compute/rpcapi.py 定义RPC client,里面定义可以远程调用的方法

  每个服务会提供一个rpcapi.py文件，这个文件包含了调用本服务RPC api的client端代码。
  
   {% highlight python %}
def start_instance(self, ctxt, instance):
        version = '4.0'
        cctxt = self.router.by_instance(ctxt, instance).prepare(
                server=_compute_host(None, instance), version=version)
        cctxt.cast(ctxt, 'start_instance', instance=instance)
  {% endhighlight %}   

- 在setup.cfg中配置endpoint

{% highlight python %}
  console_scripts =
    nova-compute = nova.cmd.compute:main
 {% endhighlight %}   
 
####  如何远程调用PRC服务

  下面这个例子演示nova-conductor模块如何远程调用上面定义的方法：start_instance()    
  在conductor中即引用rpcapi client:conductor/manager.py
  
  {% highlight python %}
   from nova.compute import rpcapi as compute_rpcapi
   self.compute_rpcapi.start_instance(context, instance)
  {% endhighlight %}  
  
  