# Nacos

## 单机模式启动server

官方文档很坑所以记录一下

### 下载

https://github.com/alibaba/nacos/releases/tag/1.2.1

### 修改配置

修改`nacos/conf/application.properties`

```properties
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://localhost:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=root
```

这些原本是注释掉的，不修改配置直接启动的话，会默认使用内嵌的derby，路径为：`nacos/data/derby-data`。我启动的时候报错了所以改成mysql（只支持mysql）

### 执行sql

执行`nacos/conf/nacos-mysql.sql`

创建所需的表，用于保存用户、配置、集群等信息

### 启动

以单机模式运行：`nacos/bin/startup.sh -m standalone`

### 控制台

http://localhost:8848/nacos/index.html

账号密码都是nacos

## 集群模式启动server

至少三台server，因为要投票

### 配置

修改文件`nacos/conf/cluster.conf`，内容为三台server的ip端口。192.168.31.69是我的本地ip，**注意：不能用localhost**！单机模式可以用localhost但是集群模式不可以。

```conf
192.168.31.69:8848

192.168.31.69:8849

192.168.31.69:8850
```

### 启动

启动脚本默认就是集群模式运行：`nacos/bin/startup.sh`

在web控制台集群管理下可以看到集群节点的列表

## 作为配置中心（单机）

首先新建两个项目，因为接下来要测试服务间的openfeign调用，所以至少需要两个项目。分别叫demoa和demob。

这里测试配置中心只需要用到demoa

###添加配置

```yml
spring:
  application:
    name: demoa
  profiles:
    active: dev
  cloud:
    nacos:
      config:
        # 配置中心的地址
        server-addr: localhost:8848
        file-extension: properties
```

配置文件的dataId为`{spring.application.name}-{spring.profiles.active}.{spring.cloud.nacos.config.file-extension}`，即项目启动时会去获取配置中心的demoa-dev.properties这个文件。

配置文件的完整id为：dataId+group+namespaceId，group不配置的话默认是DEFAULT_GROUP，namespace不配置的话就为空

#### 注

1. 如果使用yml，file-extension中要配yaml，且server端配置文件后缀也是.yaml
2. 配置文件里不能有注释！配置文件里不能有注释！配置文件里不能有注释！
3. 配置文件里不能有值为空的配置，如：`job.executor.ip=`这样的就不能有
4. properties格式配置文件的`=`左右，如果有空格，不影响发布但是读取配置的时候不会trim

### 使用配置

```java
@RestController
@RequestMapping("/config")
@RefreshScope
public class ConfigController {

    @Value("${useLocalCache:false}")
    private boolean useLocalCache;

    @RequestMapping("/get")
    public boolean get() {
        return useLocalCache;
    }

}
```

使用@Value即可获取配置中心的配置。加@RefreshScope实现热更新。

#### 注

1. 本来以为项目启动时本地的配置会自动上传到server端，事实证明我想多了。配置文件需要通过api或者在web控制台手动创建，或者在web控制台直接上传文件
2. 如果本地和server端配置了同一个参数，会优先使用server上的
3. 使用@ConfigurationProperties的配置类，不需要加@RefreshScope
4. 配置热更新主要是针对一些用户自定义的配置。有些外部依赖里的配置，比如说database，只在服务启动时加载一次，即使配置可以热更新，也要重启服务才会重新加载数据源，放在server端意义不大。唯一的好处就是可以不用重新打包，web端修改配置以后服务直接重启一下就行。另外经测试，log级别，oss的配置都是可以实时生效的，其他配置懒得测了。
5. 启动日志中，可以看到会自动订阅3个配置文件：demoa（没有后缀名也行？），demoa.properties，以及根据spring.profiles.active中设置的环境来订阅，比如这里是订阅的是demoa-dev.properties。

```log
[fixed-localhost_8848] [subscribe] demoa-dev.properties+DEFAULT_GROUP
[fixed-localhost_8848] [add-listener] ok, tenant=, dataId=demoa-dev.properties, group=DEFAULT_GROUP, cnt=1
[fixed-localhost_8848] [subscribe] demoa+DEFAULT_GROUP
[fixed-localhost_8848] [add-listener] ok, tenant=, dataId=demoa, group=DEFAULT_GROUP, cnt=1
[fixed-localhost_8848] [subscribe] demoa.properties+DEFAULT_GROUP
[fixed-localhost_8848] [add-listener] ok, tenant=, dataId=demoa.properties, group=DEFAULT_GROUP, cnt=1
```

4. 在web控制台添加或修改配置文件后，可以看到demoa的日志中，收到了通知

```log
[fixed-localhost_8848] [notify-ok] dataId=demoa-dev.properties, group=DEFAULT_GROUP, md5=25a822367cd73c79acc56da3c73fa07e, listener=com.alibaba.cloud.nacos.refresh.NacosContextRefresher$1@358a5ed3 
[fixed-localhost_8848] [notify-listener] time cost=2132ms in ClientWorker, dataId=demoa-dev.properties, group=DEFAULT_GROUP, md5=25a822367cd73c79acc56da3c73fa07e, listener=com.alibaba.cloud.nacos.refresh.NacosContextRefresher$1@358a5ed3 
```

5. 这个nacos-config跟redis一样坑，只要导入了spring-cloud-starter-alibaba-nacos-config依赖，即使不做任何配置，启动时也会去加载配置（默认是localhost:8848）。所以如果想不使用配置中心，必须设置nacos.config.enable=false，或者直接把依赖删了

## 作为注册中心（单机）

这个没啥好说的，跟eureka差不多，看官方文档就行

## 作为配置中心和注册中心（集群）

```yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.31.69:8848,192.168.31.69:8849,192.168.31.69:8850
      config:
        server-addr: 192.168.31.69:8848,192.168.31.69:8849,192.168.31.69:8850
```

其实只要配置集群中任何一个节点就行了，注册信息会自动同步到其他节点。但是万一那个节点刚好挂了呢，所以最好全部节点都配上。

####注

还看到有一种部署方式，是在client的server-addr中配置nginx的地址，然后nginx再将注册请求负载均衡到nacos集群的各个节点。

如：

```yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: nacos.ticdata.cn
      config:
        server-addr: nacos.ticdata.cn
```

```conf
  # nacos集群的三个节点
  upstream nacos-cluster {
      server 127.0.0.1:8848;
      server 127.0.0.1:8849;
      server 127.0.0.1:8850;
  }

  server{
      listen 80;
      server_name nacos.ticdata.cn;
      location /{
         proxy_pass http://nacos-cluster;
      }
  }
```

## 随便写写

### namespace

命名空间用于隔离，比如不同的租户可以共用一个nacos server，为租户分配不同的命名空间的权限（因为config_info表里的tenant_id字段，存的就是命名空间id）。**服务和配置都可以设置所属的命名空间**，默认的命名空间是public。在web控制台可以新建命名空间，名称不能重复但是！但是！但是！**在服务中配置命名空间时，必须配置id而不是名称**

```yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
        namespace: 51f2c2e0-97e7-4959-afc6-8d4eeb8651d8
      config:
        server-addr: localhost:8848
        namespace: 51f2c2e0-97e7-4959-afc6-8d4eeb8651d8
```

####坑

config相关配置必须配置在bootstrap中，discovery可以配置在application中

### group

在一个namespace中还可以用group再进行更小粒度的隔离

### 用户权限

可以看到permissions表里resource字段中保存的是`e99e2d98-ee60-4d26-b8bd-0946c4d7f742:*:*`，我猜测这个结构的意思是namespace:group:file，以后版本应该会将权限细分到文件

### server节点动态扩容

有个server节点更新线程（ServerListUpdater）每5s读取一次cluster.conf，更新server节点列表，并遍历ServerChangeListener通知。因此**server节点支持动态扩容**。

##配置中心深入理解

### 配置文件本地缓存

client会在本地缓存配置文件，路径为`用户目录/nacos/config`。启动或重启时会先从本地加载配置缓存。

### 配置文件更新过程

####client端轮询

client每30s，将本地缓存的groupKey以及md5发送到server，server会计算本地配置文件的md5并对比，返回md5不一致的配置文件的key。然后client再挨个key发送请求去下载最新配置文件。

这个定时30s不是在client端控制的，而是在server端。client端会循环发送请求到server端对比md5，server端每次收到请求后先对比一次md5，不一致就立即返回不一致的配置文件的key，一致的话就挂起29.5s以后再对比一次，以此达到一种类似定时任务的效果。

#####不多bb直接上源码：

```java
// 客户端轮询更新配置的线程
class LongPollingRunnable implements Runnable {
  	// 任务的批次，每3000个配置文件为一个批次。对没有看错就是3000
    private int taskId;

    @Override
    public void run() {
        List<CacheData> cacheDatas = new ArrayList<CacheData>();
        List<String> inInitializingCacheList = new ArrayList<String>();
        try {
            // 获取本地缓存的配置文件
            for (CacheData cacheData : cacheMap.get().values()) {
                ...
            }

            // 发送请求到server端对比md5，返回修改过的配置文件的key
            List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);
            LOGGER.info("get changedGroupKeys:" + changedGroupKeys);

          	// 发送请求下载最新配置文件
            for (String groupKey : changedGroupKeys) {
                ...
                try {
                    // 发送下载请求
                    String[] ct = getServerConfig(dataId, group, tenant, 3000L);
                    CacheData cache = cacheMap.get().get(GroupKey.getKeyTenant(dataId, group, tenant));
                  	// 更新本地缓存内容
                    cache.setContent(ct[0]);
                    ...
                } catch (NacosException ioe) {
                    ...
                }
            }
            ...
            // 循环执行下去
            executorService.execute(this);
        } catch (Throwable e) {
            ...
        }
    }
}
// 第17行跳到这里
List<String> checkUpdateConfigStr(String probeUpdateString, boolean isInitializingCacheList) throws IOException {
  	...
    try {
      // 设置超时时间。这里timeout是30。readTimeoutMs设置为45s，因为server端会延迟29.5s再处理请求，防止请求处理时间>500ms导致超时
      long readTimeoutMs = timeout + (long) Math.round(timeout >> 1);
      // 发送请求到server对比md5
      HttpResult result = agent.httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params,agent.getEncode(), readTimeoutMs);
      ...
    } catch (IOException e) {
      ...
    }
  ...
}
```

```java
// server端处理请求，最终跳到这个线程
class ClientLongPolling implements Runnable {

    @Override
    public void run() {
        // 就是这里！万恶之源！这里延迟了29.5s后执行。为什么是29.5呢，因为client端原本设置的是30s超时，所以要留500ms用来处理请求，后来发现不够，所以client端改成45s了（我猜的）
        asyncTimeoutFuture = scheduler.schedule(() -> {
            try {
                // 删除订阅关系
                allSubs.remove(ClientLongPolling.this);
                if (isFixedPolling()) {
                    ...
                    // 对比md5，返回md5不一样的配置的key
                    List<String> changedGroups = MD5Util.compareMd5(
                        (HttpServletRequest)asyncContext.getRequest(),
                        (HttpServletResponse)asyncContext.getResponse(), clientMd5Map);
                    if (changedGroups.size() > 0) {
                        sendResponse(changedGroups);
                    } else {
                        sendResponse(null);
                    }
                } else {
                    ...
                }
            } catch (Throwable t) {
                ...
            }

        }, timeoutTime, TimeUnit.MILLISECONDS);

        // 订阅。web控制台修改文件后就是遍历这个订阅列表来通知的
        allSubs.add(this);
    }
  
  	// 响应请求
    void sendResponse(List<String> changedGroups) {
        // 取消任务，防止提前响应后再次响应
        if (null != asyncTimeoutFuture) {
            asyncTimeoutFuture.cancel(false);
        }
        generateResponse(changedGroups);
    }
    ...
}
```

#### web控制台修改文件

web控制台更新配置以后，发送请求到后台，遍历订阅列表并返回修改的文件的key。client收到响应并处理完以后，会再次发送请求来订阅。

client发送对比md5的请求，server挂起请求之前会先将这个请求添加到订阅列表里面，处理请求之前会移除订阅。所以不是一直订阅的，而是server端挂起请求的时间段内才会订阅。**从server端开始处理请求到client发送下一次请求的这个很短的间隔内，在web控制台修改配置文件是不会有推送的**。不过这个几乎没有影响，因为client下一次再请求就会发现md5不一致然后更新了。

##### 上源码：

```java
// 提交配置文件修改的请求
@PostMapping
public Boolean publishConfig(...)
    throws NacosException {
    ...
    if (StringUtils.isBlank(betaIps)) {
        if (StringUtils.isBlank(tag)) {
            // 将修改后的配置保存到config_info表中
            persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, false);
          	// 发布ConfigDataChangeEvent事件
            EventDispatcher.fireEvent(new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));
        } else {
            ...
        }
    } else {
      ...
    }
    ...
    return true;
}

// LongPollingService监听器。这里捕获的是LocalDataChangeEvent而不是ConfigDataChangeEvent，具体见下一节
public void onEvent(Event event) {
  if (isFixedPolling()) {
  } else {
    if (event instanceof LocalDataChangeEvent) {
      LocalDataChangeEvent evt = (LocalDataChangeEvent)event;
      // 执行通知线程
      scheduler.execute(new DataChangeTask(evt.groupKey, evt.isBeta, evt.betaIps));
    }
  }
}

// 通知线程
class DataChangeTask implements Runnable {
  @Override
  public void run() {
    try {
      	...
        // 遍历所有订阅者，allSubs是订阅列表
        for (Iterator<ClientLongPolling> iter = allSubs.iterator(); iter.hasNext(); ) {
          ClientLongPolling clientSub = iter.next();
          if (clientSub.clientMd5Map.containsKey(groupKey)) {
            	...
            	// 删除订阅关系，一上来就删除，在处理请求（对比md5）之前
            	iter.remove();
            	...
              // 这里会提前中断那个延迟29.5s的定时线程，并立即返回修改的文件的key
              clientSub.sendResponse(Arrays.asList(groupKey));
          }
        }
    } catch (Throwable t) {
      	...
    }
  }
  ...
}
```

#### 配置轮询时间

配置更新轮询时间主要由两个参数控制：timeout和delayTime。延迟时间为：`timeout-delayTime`。

client发送那个对比md5的请求时，会把timeout放在headers里，所以虽然scheduleThreadPool是在server端的，但是client可以控制延迟的timeout参数，通过如下配置：

```yml
spring:
  cloud:
    nacos:
      config:
        config-long-poll-timeout: 30000
```

默认的timeout是30000ms，默认的delayTime是500ms，因此延迟时间为29.5s。

看源码发现，server会自动读取一个**系统开关配置文件**（先这么叫吧），dataId=com.alibaba.nacos.meta.switch，group=DEFAULT_GROUP，这个文件里面可以配置delayTIme等参数。上源码：

```java
public class SwitchService {
  	// 开关配置文件的dataId
    public static final String SWITCH_META_DATAID = "com.alibaba.nacos.meta.switch";
		// 是否在server端固定延迟时间，为true则无视client端配置的timeout参数
    public static final String FIXED_POLLING = "isFixedPolling";
  	// 固定的延迟时间
    public static final String FIXED_POLLING_INTERVAL = "fixedPollingInertval";
		// delayTime参数，这个参数client无法控制
    public static final String FIXED_DELAY_TIME = "fixedDelayTime";
  	// 配置读取后保存到这里面
    private static volatile Map<String, String> switches = new HashMap<>();
  	...
}
```

```java
// 这边从开关配置文件读取fixedDelayTime，没有配置的话默认500
int delayTime = SwitchService.getSwitchInteger(SwitchService.FIXED_DELAY_TIME, 500);
// str是从client的请求头里读取的
long timeout = Math.max(10000, Long.parseLong(str) - delayTime);
// 开关配置里的isFixedPolling为true，使用固定延迟时间
if (isFixedPolling()) {
    timeout = Math.max(10000, getFixedPollingInterval());
} else {
  	// 这里会立即对比一次md5，如果不一致就立即响应，一致的话就挂起请求
    List<String> changedGroups = MD5Util.compareMd5(req, rsp, clientMd5Map);
    if (changedGroups.size() > 0) {
        generateResponse(req, rsp, changedGroups);
        return;
    } else if (noHangUpFlag != null && noHangUpFlag.equalsIgnoreCase(TRUE_STR)) {
      	// client的请求头里有 是否挂起 的参数，刚启动时的那次请求为true，会立即返回
        return;
    }
}
```

### server集群同步配置文件

在web控制台上修改配置文件以后，请求发送到任意一个server节点，节点会保存配置到数据库中的config_info表中，然后发送请求通知所有server（**包括自己**，因为web上修改配置的请求只修改了数据库，没有更新本地缓存），将数据库中的最新配置信息，更新到本地的缓存文件中。

####被通知方

TaskManager中while轮询任务列表，server节点收到通知后就往任务列表中添加任务就行了。server节点更新本地缓存时，会发布LocalDataChangeEvent事件，这个事件会**被上一节中的LongPollingService捕获**。

所以流程应该是：**web端修改配置文件 -> 任意一个节点收到请求 -> 保存到数据库 -> 通知所有server节点更新缓存（从数据库读取最新的配置） -> 将web端更新的文件的key推送给client（提前返回挂起的请求） -> client来下载最新的配置**

```java
// 接收通知的请求
@GetMapping("/dataChange")
public Boolean notifyConfigInfo(...) {
    ...
    if (StringUtils.isNotBlank(isBetaStr) && trueStr.equals(isBetaStr)) {
        ...
    } else {
        // 调用dumpService处理
        dumpService.dump(dataId, group, tenant, tag, lastModifiedTs, handleIp);
    }
    return true;
}
```

```java
// DumpService.java
private TaskManager dumpTaskMgr;

public void dump(...) {
    ...
    // 添加任务
    dumpTaskMgr.addTask(groupKey, new DumpTask(groupKey, tag, lastModified, handleIp, isBeta));
}

@PostConstruct
public void init() {
  	...
    // init中设置默认处理器为DumpProcessor
    DumpProcessor processor = new DumpProcessor(this);
  	...
    dumpTaskMgr = new TaskManager("com.alibaba.nacos.server.DumpTaskManager");
  	dumpTaskMgr.setDefaultTaskProcessor(processor);
  	...
}

// DumpProcessor.process()。就是在这里更新本地配置缓存
public boolean process(String taskType, AbstractTask task) {
        DumpTask dumpTask = (DumpTask)task;
        ...
        // 从数据库查询最新配置信息
        ConfigInfo cf = dumpService.persistService.findConfigInfo(dataId, group, tenant);
  			...
    		boolean result;
  			if (null != cf) {
          // 更新本地的配置缓存。并发布LocalDataChangeEvent事件
          result = ConfigService.dump(dataId, group, tenant, cf.getContent(), lastModified, cf.getType());
   			...
  			} else {
          // 如果数据库里没有说明被删除了（都是物理删除），那就删除本地的配置缓存
    			result = ConfigService.remove(dataId, group, tenant);
    			...
    		}
  ...
}
```

```java
// TaskManager.java
// 构造方法中启动ProcessRunnable线程，这个线程会轮询tasks
public TaskManager(String name) {
    this.name = name;
    if (null != name && name.length() > 0) {
        this.processingThread = new Thread(new ProcessRunnable(), name);
    } else {
        this.processingThread = new Thread(new ProcessRunnable());
    }
		...
    this.processingThread.start();
}

class ProcessRunnable implements Runnable {
    @Override
    public void run() {
      while (!TaskManager.this.closed.get()) {
        try {
          // 每100ms执行一次
          Thread.sleep(100);
          TaskManager.this.process();
        } catch (Throwable e) {
        }
      }
    }
}

// 这个process()调用的是DumpService中的process()
protected void process() {
  for (Map.Entry<String, AbstractTask> entry : this.tasks.entrySet()) {
      AbstractTask task = null;
    	...
      // 获取任务
      task = entry.getValue();
      if (null != task) {
        ...
        // 先将任务从tasks中删除
        this.tasks.remove(entry.getKey());
        ...
      }
    	...
    if (null != task) {
      // 获取指定的任务处理器，目前看来set方法没有被调用过，都是用的默认处理器
      TaskProcessor processor = this.taskProcessors.get(entry.getKey());
      if (null == processor) {
        // 使用默认处理器
        processor = this.getDefaultTaskProcessor();
      }
      if (null != processor) {
        boolean result = false;
          // 处理任务，这里调用的就是上面那个process()
          result = processor.process(entry.getKey(), task);
        	...
      }
    }
  }
	...
}

// 任务添加到tasks中，在DumpService中调用
public void addTask(String type, AbstractTask task) {
    ...
    AbstractTask oldTask = tasks.put(type, task);
    ...
}
private final ConcurrentHashMap<String, AbstractTask> tasks = new ConcurrentHashMap<>();
```

#### 通知方

看代码，原本发送通知也是用的TaskManager轮询任务列表的方式（我猜的），但是后来改成了事件监听的方式。

在web控制台修改配置文件后，会发布ConfigDataChangeEvent事件，这个事件会被AsyncNotifyService捕获后通知其他节点同步。

```java
// AsyncNotifyService.java
// 设置要监听的事件为ConfigDataChangeEvent
public List<Class<? extends Event>> interest() {
  List<Class<? extends Event>> types = new ArrayList<Class<? extends Event>>();
  types.add(ConfigDataChangeEvent.class);
  return types;
}

// 事件处理
public void onEvent(Event event) {
  if (event instanceof ConfigDataChangeEvent) {
    ConfigDataChangeEvent evt = (ConfigDataChangeEvent) event;
    ...
    // 获取所有server节点，通知
    List<?> ipList = serverListService.getServerList();
    Queue<NotifySingleTask> queue = new LinkedList<NotifySingleTask>();
    for (int i = 0; i < ipList.size(); i++) {
      // 发送请求到/dataChange
      queue.add(new NotifySingleTask(dataId, group, tenant, tag, dumpTs, (String) ipList.get(i), evt.isBeta));
    }
    EXECUTOR.execute(new AsyncTask(httpclient, queue));
  }	
}
```

```java
// 这个就是配置文件更新流程里面，web端修改配置文件的提交请求
@PostMapping
public Boolean publishConfig(...)
    throws NacosException {
    ...
    if (StringUtils.isBlank(betaIps)) {
        if (StringUtils.isBlank(tag)) {
            // 将修改后的配置保存到config_info表中
            persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, false);
          	// 发布更新事件，在这里发布ConfigDataChangeEvent事件
            EventDispatcher.fireEvent(new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));
        } else {
            ...
        }
    } else {
      ...
    }
    ...
    return true;
}
```

### 共享配置与扩展配置

```yml
spring:
  cloud:
    nacos:
      config:
        server-addr: nacos.ticdata.cn
        group: mon-platform
        shared-configs:
          - dataId: database.properties
            group: mon-platform
            refresh: true
          - dataId: redis.properties
            group: mon-platform
            refresh: true
        extension-configs:
          - dataId: database2.properties
            refresh: true
```

####shared-configs

同一个命名空间下，可以共享配置文件。refresh默认是false，说明共享配置文件默认是**不会热更新**的。

#### extension-configs

感觉跟共享配置区别不大

#### 优先级

demoa-dev.properties > demoa.properties > extension > shared

## 注册中心深入理解

### 临时实例

web端服务详情里可以看到，默认注册上去的服务是**临时实例**。临时实例和持久化实例，两者有什么区别？如何注册持久化实例？

目前看来client端无法控制注册方式，自动注册只能注册为临时实例。猜测注册持久化实例可能需要手动发送注册请求。

### 健康状态

web端服务详情里可以看到每个服务的健康状态，根据心跳超时时间判断。默认每5s发送一次心跳，15s未收到心跳则将状态置为不健康，30s未收到心跳则从注册表中移除。心跳间隔和超时时间可通过如下配置自定义：

```yml
spring:
  cloud:
    nacos:
      discovery:
        heart-beat-interval: 5
        heart-beat-timeout: 15
        ip-delete-timeout: 30
```

+ 如果server端收到心跳请求后，发现这个实例已经不在注册表中了，则会**重新注册**这个实例。因为可能是网络故障导致server一直收不到心跳，然后server以为这个节点挂了就移除了，然后网络恢复后又能够收到心跳了，所以要重新注册。
+ 每个service都有一个定时线程（ClientBeatCheckTask.java），每隔5s扫描一次这个service下的所有instance，然后根据心跳时间更新实例的健康状态，以及移除死亡（三次心跳没收到）实例。移除死亡实例其实就是往本地的删除实例的接口发个请求。

### client请求随机发送

client往server发送的请求（如注册，心跳等等），都是随机选取server节点的。如果发送失败，会选择其他server重发，直到把所有server节点都发一遍。

serverList就是从配置的server-addr读取的，所以如果配nginx的话在client看来算只有一个server。但是不等于单机模式，是否是单机是在服务端记录的，客户端不管。

```java
/**
 * client发送请求都是调用这个接口。NamingProxy.java
 * @param servers 所有server节点的列表
 */
public String reqAPI(String api, Map<String, String> params, String body, List<String> servers, String method) throws NacosException {
		...
    if (servers != null && !servers.isEmpty()) {
        // 从serverList中随机选取一个节点
        Random random = new Random(System.currentTimeMillis());
        int index = random.nextInt(servers.size());
        for (int i = 0; i < servers.size(); i++) {
            String server = servers.get(index);
            try {
                // 发送请求
                return callServer(api, params, body, server, method);
            } catch (NacosException e) {
                ...
            }
            // 如果发送失败，再往下一个节点重发，直到所有节点都发一遍
            index = (index + 1) % servers.size();
        }
    }

		// 当serverList.size()==1时，nacosDomain=serverList。这边再重发一次，因为size=1时上面不会循环
    if (StringUtils.isNotBlank(nacosDomain)) {
        for (int i = 0; i < UtilAndComs.REQUEST_DOMAIN_RETRY_COUNT; i++) {
            try {
                return callServer(api, params, body, nacosDomain, method);
            } catch (NacosException e) {
                ...
            }
        }
    }
    ...
}
```

### @CanDistro

server端有些controller接口上有@CanDistro注解，比如服务注册请求，心跳请求等等。这个注解会被DistroFilter过滤器处理，过滤器中会判断一下这个请求是否由自己负责，如果加了注解但是不是由自己负责，就转发给负责该服务的server节点来处理。对于每个服务，负责处理该服务的server节点是固定的（为什么这么设计？？？）

```java
public class DistroFilter implements Filter {

    private static final int PROXY_CONNECT_TIMEOUT = 2000;
    private static final int PROXY_READ_TIMEOUT = 2000;

    @Autowired
    private DistroMapper distroMapper;
  	// 缓存了所有@RequestMapping注解的方法
    @Autowired
    private ControllerMethodsCache controllerMethodsCache;
  
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) servletRequest;
        HttpServletResponse resp = (HttpServletResponse) servletResponse;
        ...
        try {
            String path = new URI(req.getRequestURI()).getPath();
            ...
            Method method = controllerMethodsCache.getMethod(req.getMethod(), path);
            ...

            // 加了@CanDistro，但是不负责，就转发给其他节点
            if (method.isAnnotationPresent(CanDistro.class) && !distroMapper.responsible(groupedServiceName)) {
                String userAgent = req.getHeader(HttpHeaderConsts.USER_AGENT_HEADER);
                ...
                // 转发请求到其他健康的server节点
                HttpClient.HttpResult result = HttpClient.request("http://" + distroMapper.mapSrv(groupedServiceName) + req.getRequestURI(), headerList, HttpClient.translateParameterMap(req.getParameterMap()), body, PROXY_CONNECT_TIMEOUT, PROXY_READ_TIMEOUT, Charsets.UTF_8.name(), req.getMethod());
								...
                return;
            }
          
            // 自己负责处理请求
            ...
            filterChain.doFilter(requestWrapper, resp);
        } catch (AccessControlException e) {
            ...
        }
    }

    @Override
    public void destroy() {
    }
}
```

```java
public class DistroMapper implements ServerChangeListener {

    private List<String> healthyList = new ArrayList<>();

    @Autowired
    private SwitchDomain switchDomain;
    @Autowired
    private ServerListManager serverListManager;

    @PostConstruct
    public void init() {
      	// 把自己添加到server监听列表中
        serverListManager.listen(this);
    }
  	...
		// 是否负责指定的服务（没搞懂）
    public boolean responsible(String serviceName) {
        // 没开启distro，或者是单机模式，那就负责
        if (!switchDomain.isDistroEnabled() || SystemUtils.STANDALONE_MODE) {
            return true;
        }
				// 没有健康的server节点，不负责
        if (CollectionUtils.isEmpty(healthyList)) {
            return false;
        }

        // 这边为什么要前后都找？healthyList的元素是不重复的，index和lastIndex应该是一样的
        int index = healthyList.indexOf(NetUtils.localServer());
        int lastIndex = healthyList.lastIndexOf(NetUtils.localServer());
      	// 本地节点不在健康节点中，为什么要负责？？？？？？？
        if (lastIndex < 0 || index < 0) {
            return true;
        }
				// 判断自己是不是负责这个服务的server
        int target = distroHash(serviceName) % healthyList.size();
        return target >= index && target <= lastIndex;
    }

  	// 获取负责该服务的server节点
    public String mapSrv(String serviceName) {
        ...
        try {
            // 计算hashCode，然后模健康节点的size取出目标server。healthyList是排序过的，保证所有的server节点取出的同一个服务的目标server一致
            return healthyList.get(distroHash(serviceName) % healthyList.size());
        } catch (Exception e) {
            ...
            return NetUtils.localServer();
        }
    }

    public int distroHash(String serviceName) {
        return Math.abs(serviceName.hashCode() % Integer.MAX_VALUE);
    }

    @Override
    public void onChangeHealthyServerList(List<Server> latestReachableMembers) {
        List<String> newHealthyList = new ArrayList<>();
        for (Server server : latestReachableMembers) {
            newHealthyList.add(server.getKey());
        }
        healthyList = newHealthyList;
    }
}
```

### 服务注册过程

client启动时发送注册请求到server的/instance接口，收到请求的server节点首先判断这个服务是否由自己负责，如果是自己负责的话就先更新本地的注册信息，然后后台异步地将数据同步到其他的server节点。

```java
// ServiceManager.java
public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    // 服务不存在就新建服务
    createEmptyService(namespaceId, serviceName, instance.isEphemeral());
  	// 获取服务对象 
    Service service = getService(namespaceId, serviceName);
    ...
    // 向服务对象中添加实例
    addInstance(namespaceId, serviceName, instance.isEphemeral(), instance);
}

/**
 * Map<namespace, Map<group::serviceName, Service>>
 */
private Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();

// 如果服务不存在，新建服务的方法
private void putServiceAndInit(Service service) throws NacosException {
    // 放到serviceMap里
    putService(service);
    // 初始化心跳监控定时任务，这个定时任务负责更新实例的健康状态，以及移除过期实例
    service.init();
    // 注册两个监听器，service对象本身就是监听器，key分别如下：
    // com.alibaba.nacos.naming.iplist.ephemeral.{namespaceId}##{serviceName}
  consistencyService.listen(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), true), service);
    // com.alibaba.nacos.naming.iplist.{namespaceId}##{serviceName}
  consistencyService.listen(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), false), service);
}

// 第9行跳到这。添加实例
public void addInstance(String namespaceId, String serviceName, boolean ephemeral, Instance... ips) throws NacosException {
    String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral);
    Service service = getService(namespaceId, serviceName);
    synchronized (service) {
      // 向服务的实例列表中添加新的实例
      List<Instance> instanceList = addIpAddresses(service, ephemeral, ips);
      Instances instances = new Instances();
      instances.setInstanceList(instanceList);
      // 调用一致性服务，同步数据
      consistencyService.put(key, instances);
    }
}

// 接上面的put
public void put(String key, Record value) throws NacosException {
    // 更新本地的实例信息。有一个Notifier线程不停轮询tasks队列，这个方法就往队列中添加了一个task，取出后会调用service的onchange方法，更新instances，并重新计算md5（目前不知道有啥用）
    onPut(key, value);
    // 同步到其他server节点
    taskDispatcher.addTask(key);
}

// 这个线程不止一个，cpu有几核就会创建几个
public class TaskScheduler implements Runnable {
    private int index;
  	// 累计的任务数量
    private int dataSize = 0;
  	// 上次执行的时间
    private long lastDispatchTime = 0L;
  	// 任务队列
    private BlockingQueue<String> queue = new LinkedBlockingQueue<>(128 * 1024);
    ...
    @Override
    public void run() {
      List<String> keys = new ArrayList<>();
      while (true) {
        try {
          String key = queue.poll(partitionConfig.getTaskDispatchPeriod(),TimeUnit.MILLISECONDS);
          ...
					// 累计任务数量
          keys.add(key);
          dataSize++;
          // 累计1000个任务，或者距离上次执行时间大于2s就开始执行
          if (dataSize == partitionConfig.getBatchSyncKeyCount() ||
              (System.currentTimeMillis() - lastDispatchTime) > partitionConfig.getTaskDispatchPeriod()) {
            // 同步到其他server节点
            for (Server member : dataSyncer.getServers()) {
              // 跳过本地节点
              if (NetUtils.localServer().equals(member.getKey())) {
                continue;
              }
              // 同步数据。这里调用的是NamingProxy中的syncData方法，发送请求到server端的/distro/datum接口，请求中带有最新的实例信息
              SyncTask syncTask = new SyncTask();
              syncTask.setKeys(keys);
              syncTask.setTargetServer(member.getKey());
              dataSyncer.submit(syncTask, 0);
            }
            lastDispatchTime = System.currentTimeMillis();
            dataSize = 0;
          }
        } catch (Exception e) {
          ...
        }
      }
    }
}
```

### server健康检查

server节点之间也会有类似于心跳的机制，用于同步节点间的状态信息，每个server节点中会缓存一个健康的server节点列表，用于distro协议转发请求等等。

```java
// 服务状态同步线程
private class ServerStatusReporter implements Runnable {
    @Override
    public void run() {
        try {
          	...
            // 更新健康节点列表
            checkDistroHeartbeat();

            int weight = Runtime.getRuntime().availableProcessors() / 2;
            if (weight <= 0) {
                weight = 1;
            }
            long curTime = System.currentTimeMillis();
            // 发送给其他server节点的数据（状态信息），包含自己的ip、发送时间、权重，LOCALHOST_SITE=unknown，写死了，目前看来没啥用，可能以后版本会用
            String status = LOCALHOST_SITE + "#" + NetUtils.localServer() + "#" + curTime + "#" + weight + "\r\n";

            // 自己发送到自己
            onReceiveServerStatus(status);

            List<Server> allServers = getServers();
            ...
            if (allServers.size() > 0 && !NetUtils.localServer().contains(UtilsAndCommons.LOCAL_HOST_IP)) {
              	// 向其他节点发送自己的状态信息
                for (com.alibaba.nacos.naming.cluster.servers.Server server : allServers) {
                    if (server.getKey().equals(NetUtils.localServer())) {
                        continue;
                    }
                    Message msg = new Message();
                    msg.setData(status);
                    // 发送请求，收到这个请求的server最终调用的就是上面的onReceiveServerStatus方法，所以上面直接调用，就相当于发了个请求给自己
                    synchronizer.send(server.getKey(), msg);
                }
            }
        } catch (Exception e) {
            ...
        } finally {
            // 2s执行一次
            GlobalExecutor.registerServerStatusReporter(this, switchDomain.getServerStatusSynchronizationPeriodMillis());
        }
    }
}

// 接收其他server节点（或者自己）发来的状态信息
public synchronized void onReceiveServerStatus(String configInfo) {
    String[] configs = configInfo.split("\r\n");
    ...
    List<Server> tmpServerList = new ArrayList<>();
    for (String config : configs) {
      tmpServerList.clear();
      // site#ip:port#lastReportTime#weight
      String[] params = config.split("#");
			...
      Server server = new Server();
      // 目前看来site都是unknown，可能后续版本这个参数会有用
      server.setSite(params[0]);
      server.setIp(params[1].split(UtilsAndCommons.IP_PORT_SPLITER)[0]);
      server.setServePort(Integer.parseInt(params[1].split(UtilsAndCommons.IP_PORT_SPLITER)[1]));
      server.setLastRefTime(Long.parseLong(params[2]));

      // 本地的server节点列表缓存中不包含接收到的server节点就报错。猜测是为了防止本地的cluster.conf与其他server的不一致
      if (!contains(server.getKey())) {
        throw new IllegalArgumentException("server: " + server.getKey() + " is not in serverlist");
      }

      // 上一次收到这个server的状态信息的时间
      Long lastBeat = distroBeats.get(server.getKey());
      long now = System.currentTimeMillis();
      if (null != lastBeat) {
        // 判断心跳是否超时，默认10s（这边有问题，都收到这个心跳了肯定是存活的啊，直接设置true不就行了）
        server.setAlive(now - lastBeat < switchDomain.getDistroServerExpiredMillis());
      }
      // 更新这个server的心跳时间
      distroBeats.put(server.getKey(), now);
			...
      // 更新当前site下的server节点列表
      List<Server> list = distroConfig.get(server.getSite());
      if (list == null || list.size() <= 0) {
        list = new ArrayList<>();
        list.add(server);
        distroConfig.put(server.getSite(), list);
      }

      for (Server s : list) {
        String serverId = s.getKey() + "_" + s.getSite();
        String newServerId = server.getKey() + "_" + server.getSite();
        // 从节点列表中找到这个server节点（不是第一次心跳），存入这个server最新的状态信息
        if (serverId.equals(newServerId)) {
          tmpServerList.add(server);
          continue;
        }
        tmpServerList.add(s);
      }
      // 第一次收到这个server的心跳，比如刚改完cluster.conf后。则直接添加
      if (!tmpServerList.contains(server)) {
        tmpServerList.add(server);
      }
      distroConfig.put(server.getSite(), tmpServerList);
    }
    liveSites.addAll(distroConfig.keySet());
}

// 检查心跳，更新健康节点列表
private void checkDistroHeartbeat() {
  	// 可以看出来，健康节点列表只包含本地site下的server节点，即distro只会转发请求到同一个site下的其他节点
    List<Server> servers = distroConfig.get(LOCALHOST_SITE);

    List<Server> newHealthyList = new ArrayList<>(servers.size());
    long now = System.currentTimeMillis();
  	// 更新所有节点健康状态
    for (Server s : servers) {
        Long lastBeat = distroBeats.get(s.getKey());
        if (null == lastBeat) {
            continue;
        }
        s.setAlive(now - lastBeat < switchDomain.getDistroServerExpiredMillis());
    }

    List<String> allLocalSiteSrvs = new ArrayList<>();
    for (Server server : servers) {
				...
        server.setAdWeight(switchDomain.getAdWeight(server.getKey()) == null ? 0 : switchDomain.getAdWeight(server.getKey()));
        // 循环内没有用到i，为什么要i循环？？？？
        for (int i = 0; i < server.getWeight() + server.getAdWeight(); i++) {
          	// allLocalSiteSrvs和newHealthyList都不重复
            if (!allLocalSiteSrvs.contains(server.getKey())) {
                allLocalSiteSrvs.add(server.getKey());
            }
            if (server.isAlive() && !newHealthyList.contains(server)) {
                newHealthyList.add(server);
            }
        }
    }
		// 排序，保证所有的节点中缓存的健康节点列表顺序一致。因为distro转发时是根据hash%size来取的节点，顺序不一致会导致，同一个服务会取到不同的负责节点
    Collections.sort(newHealthyList);
  	// 存活率
    float curRatio = (float) newHealthyList.size() / allLocalSiteSrvs.size();
    // 存活率 > 0.7，且距离关闭检查超过60s。为什么存活率>0.7才会开启检查？？？？
    if (autoDisabledHealthCheck
        && curRatio > switchDomain.getDistroThreshold()
        && System.currentTimeMillis() - lastHealthServerMillis > STABLE_PERIOD) {
        // 开启健康检查
        switchDomain.setHealthCheckEnabled(true);
        autoDisabledHealthCheck = false;
    }

    // 如果当前健康列表和最新健康列表不一样，就更新
    if (!CollectionUtils.isEqualCollection(healthyServers, newHealthyList)) {
        // 是否打开自动健康检查
        if (switchDomain.isHealthCheckEnabled() && switchDomain.isAutoChangeHealthCheckEnabled()) {
            // 暂时关闭健康检查，60s后上面的方法中再开启。为什么要关闭了再开？？？
            switchDomain.setHealthCheckEnabled(false);
            autoDisabledHealthCheck = true;
            // 记录关闭检查的时间
            lastHealthServerMillis = System.currentTimeMillis();
        }
        // 更新健康节点列表并通知
        healthyServers = newHealthyList;
        notifyListeners();
    }
}
```

