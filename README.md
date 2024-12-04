
网关提供外部访问内部微服务的统一入口，基于分布式和服务治理等功能特点，外部不能绕过网关调用内部微服务（框架本身提供外部可以直接访问内部微服务的功能，这里不作详细说明），外部通过 http 协议请求网关暴露的接口，网关再用基于 TCP/IP 协议的 RPC 方式调用内部被发现的微服务。



## 1 创建网关



创建一个基于 .Net Core 6\.0 的 Asp.Net Core Web （项目模版选 WebApi）的项目，命名为 ApiGateway



### 1\.1 添加 Jimu.Client 引用



网关相对于微服务而言属于客户端，所以引用 Jimu.Client，因为需要支持 Consul， 所以还要因为 Jimu 对 consul 的扩展库 Jimu.Common.Discovery.ConsulIntegration





```
Install-Package  Jimu.Client
Install-Package  Jimu.Common.Discovery.ConsulIntegration
```




### 1\.2 然后在 Startup 里添加 Jimu 的启动代码






```
using System.Linq;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.IdentityModel.Tokens;
using Autofac;
using Jimu;
using Jimu.Client;
using Jimu.Client.ApiGateway;

namespace ApiGateway
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddCors(); // 支持跨域
            services.UseJimu();  // 支持 Jimu
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {

            app.UseStaticFiles();
            app.UseAuthentication();
            app.UseStatusCodePages();

            // jimu client, 启动 jimu 
            var host = new ServiceHostClientBuilder(new ContainerBuilder())
                .UseLog4netLogger(new LogOptions
                {
                    EnableConsoleLog = true,  // 启用控制台日志
                    EnableFileLog = true, // 启用文件日志
                    FileLogLevel = LogLevel.Info | LogLevel.Error, // 文件日志只记录 Info 和 Error
                })
                .UseConsulForDiscovery("127.0.0.1", 8500, "JimuService") // 配置 consul 
                .UseDotNettyForTransfer() // 使用 dotnetty 做 RPC 的通信库
                .UseHttpForTransfer() // 同时支持使用  http 作为 RPC 的通信库
                .UsePollingAddressSelector() // 使用轮询算法实现负载均衡
                .UseServerHealthCheck(1) // 服务健康监控，时间间隔为 1 分钟
                .SetRemoteCallerRetryTimes(3) // 调用微服务失败重试次数，设为 3次
                .SetDiscoveryAutoUpdateJobInterval(1) // 服务发现，时间间隔 1 分钟
                .UseToken(() => { var headers = JimuHttpContext.Current.Request.Headers["Authorization"]; return headers.Any() ? headers[0] : null; }) // 获取请求所携带的 token
                .Build();
            app.UseJimu(host); // 使用 jimu
            host.Run(); // 启动 jimu
        }
    }
}
```



最基本的网关完成了，已经支持接收请求。 下面要加入一些可视化的服务治理，如微服务器列表，服务器健康状态，服务列表



### 1\.3 服务治理


:[楚门加速器](https://shexiangshi.org)
服务治理还未完善，只简单做了服务器列表和健康状态，以及开放的服务列表。


创建 2 个控制器：



#### 1\.3\.1 Server 展示服务器列表






```
using Autofac;
using Jimu;
using Jimu.Client;
using Jimu.Client.ApiGateway;
using Microsoft.AspNetCore.Mvc;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace ApiGateway.Controllers
{
    public class ServerController : Controller
    {
        public IActionResult Index()
        {
            return View();
        }
        //[HttpGet(Name ="addresses")]
        // 获取服务器列表
        public async Task> GetAddresses()
        {
            var serviceDiscovery = JimuClient.Host.Container.Resolve();
            var addresses = await serviceDiscovery.GetAddressAsync();
            return addresses;

        }

    }
}
```



注意：控制器和方法上面都不需添加任何属性



#### 1\.3\.2 Services 展示服务列表






```
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Autofac;
using Jimu;
using Jimu.Client;
using Jimu.Client.ApiGateway;
using Microsoft.AspNetCore.Mvc;

namespace ApiGateway.Controllers
{
    //[Produces("application/json")]
    public class ServicesController : Controller
    {
        public IActionResult Index()
        {
            return View();
        }
        //[HttpGet(Name ="services")]
        // 获取服务列表
        public async Task> GetServices(string server)
        {
            var serviceDiscovery = JimuClient.Host.Container.Resolve();
            var routes = await serviceDiscovery.GetRoutesAsync();
            if (routes != null && routes.Any() && !string.IsNullOrEmpty(server))
            {
                return (from route in routes
                        where route.Address.Any(x => x.Code == server)
                        select route.ServiceDescriptor).ToList();
            }
            return (from route in routes select route.ServiceDescriptor).ToList();
        }
    }
}
```


 



