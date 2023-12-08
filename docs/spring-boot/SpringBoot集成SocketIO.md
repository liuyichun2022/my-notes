## SpringBoot 集成SocketIO

SocketIO官网：https://socket.nodejs.cn/

博客的五花八门，Socket'IO官网写的还是比较的详细。

SpringBoot集成netty-socketio

1、安装依赖

```xml
		<!--SocketIO-->
        <dependency>
            <groupId>com.corundumstudio.socketio</groupId>
            <artifactId>netty-socketio</artifactId>
            <version>1.7.11</version>
        </dependency>
```



2、yml文件配置

```yml
socketio:
  allowCustomRequests: true
  bossCount: 1
  host: 0.0.0.0       # 注意：要配置成0.0.0.0本机和服务器都能访问，不然很容易造成本地测试没问题，发布到环境上不能用
  maxFramePayloadLength: 1048576
  maxHttpContentLength: 1048576
  pingInterval: 25000
  pingTimeout: 6000000
  port: 8099
  upgradeTimeout: 1000000
  workCount: 100
```

3、SocketIoConfig配置注册启动SocketIO服务端,  通过配置configuration.setAuthorizationListener来进行ws接口的鉴权操作。

```java
import com.corundumstudio.socketio.SocketConfig;
import com.corundumstudio.socketio.SocketIOServer;
import com.moemil.bodyhealth.home.core.iosocket.SocketIOHandler;
import io.netty.handler.codec.http.HttpHeaders;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import javax.annotation.Resource;

@Configuration
@ConfigurationProperties("socketio")
@Slf4j
public class SocketIoConfig implements InitializingBean {

	@Resource
	private SocketIOHandler socketIOHandler;

	@Value("${socketio.host}")
	private String host;

	@Value("${socketio.port}")
	private Integer port;

	@Value("${socketio.bossCount}")
	private int bossCount;

	@Value("${socketio.workCount}")
	private int workCount;

	@Value("${socketio.allowCustomRequests}")
	private boolean allowCustomRequests;

	@Value("${socketio.upgradeTimeout}")
	private int upgradeTimeout;

	@Value("${socketio.pingTimeout}")
	private int pingTimeout;

	@Value("${socketio.pingInterval}")
	private int pingInterval;

	@Override
	public void afterPropertiesSet() throws Exception {
		SocketConfig socketConfig = new SocketConfig();
		socketConfig.setReuseAddress(true);
		socketConfig.setTcpNoDelay(true);
		socketConfig.setSoLinger(0);
		com.corundumstudio.socketio.Configuration configuration = new com.corundumstudio.socketio.Configuration();
		configuration.setSocketConfig(socketConfig);
		// host在本地测试可以设置为localhost或者本机IP，在Linux服务器跑可换成服务器IP
		configuration.setHostname(host);
		configuration.setPort(port);
		// socket连接数大小（如只监听一个端口boss线程组为1即可）
		configuration.setBossThreads(bossCount);
		configuration.setWorkerThreads(workCount);
		configuration.setAllowCustomRequests(allowCustomRequests);
		// 协议升级超时时间（毫秒），默认10秒。HTTP握手升级为ws协议超时时间
		configuration.setUpgradeTimeout(upgradeTimeout);
		// Ping消息超时时间（毫秒），默认60秒，这个时间间隔内没有接收到心跳消息就会发送超时事件
		configuration.setPingTimeout(pingTimeout);
		// Ping消息间隔（毫秒），默认25秒。客户端向服务器发送一条心跳消息间隔
		configuration.setPingInterval(pingInterval);
		configuration.setAuthorizationListener(handshakeData -> {
			//获取socket链接发来的token参数
			/*HttpHeaders httpHeaders = handshakeData.getHttpHeaders();
			String auth = httpHeaders.get("momi-auth");
			if (StrUtil.isBlank(auth)) {
				return false;
			}
			String token = SecureUtil.getToken(auth);
			if (StrUtil.isBlank(token)) {
				return false;
			}
			Claims claims = SecureUtil.parseJWT(token);
			if (Func.isNull(claims)) {
				return false;
			}
			String userId = Func.toStr(claims.get("user_id"));
			System.out.println(userId);
			httpHeaders.set("user_id", userId);*/
			String userId = handshakeData.getSingleUrlParam("userId");
			HttpHeaders httpHeaders = handshakeData.getHttpHeaders();
			httpHeaders.set("user_id", userId);
			return true;
		});

		SocketIOServer socketIOServer = new SocketIOServer(configuration);
		//添加事件监听器
		socketIOServer.addListeners(socketIOHandler);
		//启动SocketIOServer
		socketIOServer.start();
		System.out.println("SocketIO启动完毕");
	}
```

4、SocketIOHandler 类配置@OnConnect、@OnDisconnect、@OnEvent 等事件处理流程，对于前后端的一下事件处理也可以定义通用的路由处理策略去处理。



```
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.corundumstudio.socketio.AckRequest;
import com.corundumstudio.socketio.SocketIOClient;
import com.corundumstudio.socketio.annotation.OnConnect;
import com.corundumstudio.socketio.annotation.OnDisconnect;
import com.corundumstudio.socketio.annotation.OnEvent;
import com.moemil.bodyhealth.home.common.constant.EventConstant;
import com.moemil.bodyhealth.home.common.enums.EventTypeEnum;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.Objects;
import java.util.UUID;

@Component
@Slf4j
public class SocketIOHandler {

	@Resource
	private MassageHandleFactory massageHandleFactory;

	@OnConnect
	public void onConnect(SocketIOClient client){
		String userId = client.getHandshakeData().getHttpHeaders().get("user_id");
		ClientCache.saveClient(userId, client);
		client.set("userId", userId);
		log.info("建立连接成功, userId:{}, sessionId:{}", userId, client.getSessionId());
	}

	@OnDisconnect
	public void onDisconnect(SocketIOClient client){
		String userId = client.getHandshakeData().getSingleUrlParam("userId");
		ClientCache.deleteSessionClientByUserId(userId);
		log.info("连接关闭成功, userId:{}, sessionId:{}", userId, client.getSessionId());
	}

	@OnEvent(EventConstant.DEVICE_ACTIVE)
	public void activeEvent(SocketIOClient client, AckRequest ackRequest, String message){
		UUID sessionId = client.getSessionId();
		JSONObject jsonObject = JSON.parseObject(message);
		if (Objects.nonNull(jsonObject)) {
			client.set("activeCode", jsonObject.getString("activeCode"));
		}
		// 注册激活事件的流程
		log.info("设备注册激活流程, userId: {}, activeCode:{}", client.get("userId"), client.get("activeCode"));
		massageHandleFactory.router(EventTypeEnum.DEVICE_ACTIVE_EVENT.getBeanName(), client.get("userId"), message);
		System.out.println(message);
	}

}
```

5、利用redisson的pubsub机制来实现分布式的消息处理，单机的废物主要是有多个节点的时候，发送消息给用户，不知道是那个节点和后端建立的长连接，利用redis的发布订阅机制来实现分布式的会话消息。

```java
import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.corundumstudio.socketio.SocketIOClient;
import com.moemil.bodyhealth.home.common.constant.SocketIoConstants;
import com.moemil.bodyhealth.home.core.iosocket.ClientCache;
import com.moemil.bodyhealth.home.core.vo.SocketIoMessageVO;
import lombok.AccessLevel;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

@Slf4j
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class SocketIoUtils {


    /**
     * 发送消息
     *
     * @param sessionKey session主键 一般为用户id
     * @param message    消息文本
     */
    public static void sendMessage(String sessionKey, String message) {
		SocketIOClient client = ClientCache.getUserClient(String.valueOf(sessionKey));
        sendMessage(client, message);
    }

    /**
     * 订阅消息
     *
     * @param consumer 自定义处理
     */
    public static void subscribeMessage(Consumer<SocketIoMessageVO> consumer) {
        RedissonUtils.subscribe(SocketIoConstants.WEB_SOCKET_TOPIC, SocketIoMessageVO.class, consumer);
    }

    /**
     * 发布订阅的消息
     *
     * @param socketIoMessage 消息对象
     */
    public static void publishMessage(SocketIoMessageVO socketIoMessage) {
        List<String> unsentSessionKeys = new ArrayList<>();
        // 当前服务内session,直接发送消息
        for (String sessionKey : socketIoMessage.getSessionKeys()) {
            if (ClientCache.existSession(sessionKey)) {
                SocketIoUtils.sendMessage(sessionKey, socketIoMessage.getMessage());
                continue;
            }
            unsentSessionKeys.add(sessionKey);
        }
        // 不在当前服务内session,发布订阅消息
        if (CollUtil.isNotEmpty(unsentSessionKeys)) {
			SocketIoMessageVO broadcastMessage = new SocketIoMessageVO();
            broadcastMessage.setMessage(socketIoMessage.getMessage());
            broadcastMessage.setSessionKeys(unsentSessionKeys);
			RedissonUtils.publish(SocketIoConstants.WEB_SOCKET_TOPIC, broadcastMessage, consumer -> {
                log.info(" WebSocket发送主题订阅消息topic:{} session keys:{} message:{}",
					SocketIoConstants.WEB_SOCKET_TOPIC, unsentSessionKeys, socketIoMessage.getMessage());
            });
        }
    }

    /**
     * 发布订阅的消息(群发)
     *
     * @param message 消息内容
     */
    public static void publishAll(String message) {
		SocketIoMessageVO broadcastMessage = new SocketIoMessageVO();
        broadcastMessage.setMessage(message);
        RedissonUtils.publish(SocketIoConstants.WEB_SOCKET_TOPIC, broadcastMessage, consumer -> {
            log.info("WebSocket发送主题订阅消息topic:{} message:{}", SocketIoConstants.WEB_SOCKET_TOPIC, message);
        });
    }


    public static void sendMessage(SocketIOClient session, String message) {

		if (session == null || !session.isChannelOpen()) {
			log.error("[send] session会话已经关闭");
		} else {
			try {
				JSONObject jsonObject = JSON.parseObject(message);
				String msgType = jsonObject.getString("msgType");
				if (StrUtil.isNotBlank(msgType)) {
					session.sendEvent(msgType, message);
				} else {
					session.sendEvent(message);
				}
			} catch (Exception e) {
				log.error("[send] session({}) 发送消息({}) 异常", session, message, e);
			}
		}
    }
}
```



6、Redisson配置

```java
import lombok.Data;
import lombok.NoArgsConstructor;
import org.redisson.config.ReadMode;
import org.redisson.config.SubscriptionMode;
import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * Redisson 配置属性
 * @author liuyichun
 */
@Data
@ConfigurationProperties(prefix = "redisson")
public class RedissonProperties {

	/**
	 * redis缓存key前缀
	 */
	private String keyPrefix;

	/**
	 * 线程池数量,默认值 = 当前处理核数量 * 2
	 */
	private int threads;

	/**
	 * Netty线程池数量,默认值 = 当前处理核数量 * 2
	 */
	private int nettyThreads;

	/**
	 * 单机服务配置
	 */
	private SingleServerConfig singleServerConfig;

	/**
	 * 集群服务配置
	 */
	private ClusterServersConfig clusterServersConfig;

	@Data
	@NoArgsConstructor
	public static class SingleServerConfig {

		/**
		 * 客户端名称
		 */
		private String clientName;

		/**
		 * 最小空闲连接数
		 */
		private int connectionMinimumIdleSize;

		/**
		 * 连接池大小
		 */
		private int connectionPoolSize;

		/**
		 * 连接空闲超时，单位：毫秒
		 */
		private int idleConnectionTimeout;

		/**
		 * 命令等待超时，单位：毫秒
		 */
		private int timeout;

		/**
		 * 发布和订阅连接池大小
		 */
		private int subscriptionConnectionPoolSize;

	}

	@Data
	@NoArgsConstructor
	public static class ClusterServersConfig {

		/**
		 * 客户端名称
		 */
		private String clientName;

		/**
		 * master最小空闲连接数
		 */
		private int masterConnectionMinimumIdleSize;

		/**
		 * master连接池大小
		 */
		private int masterConnectionPoolSize;

		/**
		 * slave最小空闲连接数
		 */
		private int slaveConnectionMinimumIdleSize;

		/**
		 * slave连接池大小
		 */
		private int slaveConnectionPoolSize;

		/**
		 * 连接空闲超时，单位：毫秒
		 */
		private int idleConnectionTimeout;

		/**
		 * 命令等待超时，单位：毫秒
		 */
		private int timeout;

		/**
		 * 发布和订阅连接池大小
		 */
		private int subscriptionConnectionPoolSize;

		/**
		 * 读取模式
		 */
		private ReadMode readMode;

		/**
		 * 订阅模式
		 */
		private SubscriptionMode subscriptionMode;

	}
}
```

7、Redisson 单机和集群实例化配置RedissonConfig

```java
import cn.hutool.core.util.ObjectUtil;
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.moemil.bodyhealth.home.common.handle.KeyPrefixHandler;
import lombok.extern.slf4j.Slf4j;
import org.redisson.client.codec.StringCodec;
import org.redisson.codec.CompositeCodec;
import org.redisson.codec.TypedJsonJacksonCodec;
import org.redisson.spring.starter.RedissonAutoConfigurationCustomizer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;

@Slf4j
@AutoConfiguration
@EnableCaching
@EnableConfigurationProperties(RedissonProperties.class)
public class RedissonConfig {

	@Autowired
	private RedissonProperties redissonProperties;

	@Autowired
	private ObjectMapper objectMapper;

	@Bean
	public RedissonAutoConfigurationCustomizer redissonCustomizer() {
		return config -> {
			ObjectMapper om = objectMapper.copy();
			om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
			// 指定序列化输入的类型，类必须是非final修饰的。序列化时将对象全类名一起保存下来
			om.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);
			TypedJsonJacksonCodec jsonCodec = new TypedJsonJacksonCodec(Object.class, om);
			// 组合序列化 key 使用 String 内容使用通用 json 格式
			CompositeCodec codec = new CompositeCodec(StringCodec.INSTANCE, jsonCodec, jsonCodec);
			config.setThreads(redissonProperties.getThreads())
				.setNettyThreads(redissonProperties.getNettyThreads())
				.setCodec(codec);
			RedissonProperties.SingleServerConfig singleServerConfig = redissonProperties.getSingleServerConfig();
			if (ObjectUtil.isNotNull(singleServerConfig)) {
				// 使用单机模式
				config.useSingleServer()
					//设置redis key前缀
					.setNameMapper(new KeyPrefixHandler(redissonProperties.getKeyPrefix()))
					.setTimeout(singleServerConfig.getTimeout())
					.setClientName(singleServerConfig.getClientName())
					.setIdleConnectionTimeout(singleServerConfig.getIdleConnectionTimeout())
					.setSubscriptionConnectionPoolSize(singleServerConfig.getSubscriptionConnectionPoolSize())
					.setConnectionMinimumIdleSize(singleServerConfig.getConnectionMinimumIdleSize())
					.setConnectionPoolSize(singleServerConfig.getConnectionPoolSize());
			}
			// 集群配置方式 参考下方注释
			RedissonProperties.ClusterServersConfig clusterServersConfig = redissonProperties.getClusterServersConfig();
			if (ObjectUtil.isNotNull(clusterServersConfig)) {
				config.useClusterServers()
					//设置redis key前缀
					.setNameMapper(new KeyPrefixHandler(redissonProperties.getKeyPrefix()))
					.setTimeout(clusterServersConfig.getTimeout())
					.setClientName(clusterServersConfig.getClientName())
					.setIdleConnectionTimeout(clusterServersConfig.getIdleConnectionTimeout())
					.setSubscriptionConnectionPoolSize(clusterServersConfig.getSubscriptionConnectionPoolSize())
					.setMasterConnectionMinimumIdleSize(clusterServersConfig.getMasterConnectionMinimumIdleSize())
					.setMasterConnectionPoolSize(clusterServersConfig.getMasterConnectionPoolSize())
					.setSlaveConnectionMinimumIdleSize(clusterServersConfig.getSlaveConnectionMinimumIdleSize())
					.setSlaveConnectionPoolSize(clusterServersConfig.getSlaveConnectionPoolSize())
					.setReadMode(clusterServersConfig.getReadMode())
					.setSubscriptionMode(clusterServersConfig.getSubscriptionMode());
			}
			log.info("初始化 redis 配置");
		};
	}
}
```

9. 核心工具类RedissonUtils

```java
import com.moemil.core.tool.utils.SpringUtil;
import lombok.AccessLevel;
import lombok.NoArgsConstructor;
import org.redisson.api.*;

import java.time.Duration;
import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Consumer;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * redis 工具类
 *
 * @author liuyichun
 */
@NoArgsConstructor(access = AccessLevel.PRIVATE)
@SuppressWarnings(value = {"unchecked", "rawtypes"})
public class RedissonUtils {

    private static final RedissonClient CLIENT = SpringUtil.getBean(RedissonClient.class);

    /**
     * 限流
     *
     * @param key          限流key
     * @param rateType     限流类型
     * @param rate         速率
     * @param rateInterval 速率间隔
     * @return -1 表示失败
     */
    public static long rateLimiter(String key, RateType rateType, int rate, int rateInterval) {
        RRateLimiter rateLimiter = CLIENT.getRateLimiter(key);
        rateLimiter.trySetRate(rateType, rate, rateInterval, RateIntervalUnit.SECONDS);
        if (rateLimiter.tryAcquire()) {
            return rateLimiter.availablePermits();
        } else {
            return -1L;
        }
    }

    /**
     * 获取客户端实例
     */
    public static RedissonClient getClient() {
        return CLIENT;
    }

    /**
     * 发布通道消息
     *
     * @param channelKey 通道key
     * @param msg        发送数据
     * @param consumer   自定义处理
     */
    public static <T> void publish(String channelKey, T msg, Consumer<T> consumer) {
        RTopic topic = CLIENT.getTopic(channelKey);
        topic.publish(msg);
        consumer.accept(msg);
    }

    public static <T> void publish(String channelKey, T msg) {
        RTopic topic = CLIENT.getTopic(channelKey);
        topic.publish(msg);
    }

    /**
     * 订阅通道接收消息
     *
     * @param channelKey 通道key
     * @param clazz      消息类型
     * @param consumer   自定义处理
     */
    public static <T> void subscribe(String channelKey, Class<T> clazz, Consumer<T> consumer) {
        RTopic topic = CLIENT.getTopic(channelKey);
        topic.addListener(clazz, (channel, msg) -> consumer.accept(msg));
    }

    /**
     * 缓存基本的对象，Integer、String、实体类等
     *
     * @param key   缓存的键值
     * @param value 缓存的值
     */
    public static <T> void setCacheObject(final String key, final T value) {
        setCacheObject(key, value, false);
    }

    /**
     * 缓存基本的对象，保留当前对象 TTL 有效期
     *
     * @param key       缓存的键值
     * @param value     缓存的值
     * @param isSaveTtl 是否保留TTL有效期(例如: set之前ttl剩余90 set之后还是为90)
     * @since Redis 6.X 以上使用 setAndKeepTTL 兼容 5.X 方案
     */
    public static <T> void setCacheObject(final String key, final T value, final boolean isSaveTtl) {
        RBucket<T> bucket = CLIENT.getBucket(key);
        if (isSaveTtl) {
            try {
                bucket.setAndKeepTTL(value);
            } catch (Exception e) {
                long timeToLive = bucket.remainTimeToLive();
                setCacheObject(key, value, Duration.ofMillis(timeToLive));
            }
        } else {
            bucket.set(value);
        }
    }

    /**
     * 缓存基本的对象，Integer、String、实体类等
     *
     * @param key      缓存的键值
     * @param value    缓存的值
     * @param duration 时间
     */
    public static <T> void setCacheObject(final String key, final T value, final Duration duration) {
        RBatch batch = CLIENT.createBatch();
        RBucketAsync<T> bucket = batch.getBucket(key);
        bucket.setAsync(value);
        bucket.expireAsync(duration);
        batch.execute();
    }

    /**
     * 如果不存在则设置 并返回 true 如果存在则返回 false
     *
     * @param key   缓存的键值
     * @param value 缓存的值
     * @return set成功或失败
     */
    public static <T> boolean setObjectIfAbsent(final String key, final T value, final Duration duration) {
        RBucket<T> bucket = CLIENT.getBucket(key);
        return bucket.setIfAbsent(value, duration);
    }

    /**
     * 注册对象监听器
     * <p>
     * key 监听器需开启 `notify-keyspace-events` 等 redis 相关配置
     *
     * @param key      缓存的键值
     * @param listener 监听器配置
     */
    public static <T> void addObjectListener(final String key, final ObjectListener listener) {
        RBucket<T> result = CLIENT.getBucket(key);
        result.addListener(listener);
    }

    /**
     * 设置有效时间
     *
     * @param key     Redis键
     * @param timeout 超时时间
     * @return true=设置成功；false=设置失败
     */
    public static boolean expire(final String key, final long timeout) {
        return expire(key, Duration.ofSeconds(timeout));
    }

    /**
     * 设置有效时间
     *
     * @param key      Redis键
     * @param duration 超时时间
     * @return true=设置成功；false=设置失败
     */
    public static boolean expire(final String key, final Duration duration) {
        RBucket rBucket = CLIENT.getBucket(key);
        return rBucket.expire(duration);
    }

    /**
     * 获得缓存的基本对象。
     *
     * @param key 缓存键值
     * @return 缓存键值对应的数据
     */
    public static <T> T getCacheObject(final String key) {
        RBucket<T> rBucket = CLIENT.getBucket(key);
        return rBucket.get();
    }

    /**
     * 获得key剩余存活时间
     *
     * @param key 缓存键值
     * @return 剩余存活时间
     */
    public static <T> long getTimeToLive(final String key) {
        RBucket<T> rBucket = CLIENT.getBucket(key);
        return rBucket.remainTimeToLive();
    }

    /**
     * 删除单个对象
     *
     * @param key 缓存的键值
     */
    public static boolean deleteObject(final String key) {
        return CLIENT.getBucket(key).delete();
    }

    /**
     * 删除集合对象
     *
     * @param collection 多个对象
     */
    public static void deleteObject(final Collection collection) {
        RBatch batch = CLIENT.createBatch();
        collection.forEach(t -> {
            batch.getBucket(t.toString()).deleteAsync();
        });
        batch.execute();
    }

    /**
     * 检查缓存对象是否存在
     *
     * @param key 缓存的键值
     */
    public static boolean isExistsObject(final String key) {
        return CLIENT.getBucket(key).isExists();
    }

    /**
     * 缓存List数据
     *
     * @param key      缓存的键值
     * @param dataList 待缓存的List数据
     * @return 缓存的对象
     */
    public static <T> boolean setCacheList(final String key, final List<T> dataList) {
        RList<T> rList = CLIENT.getList(key);
        return rList.addAll(dataList);
    }

    /**
     * 注册List监听器
     * <p>
     * key 监听器需开启 `notify-keyspace-events` 等 redis 相关配置
     *
     * @param key      缓存的键值
     * @param listener 监听器配置
     */
    public static <T> void addListListener(final String key, final ObjectListener listener) {
        RList<T> rList = CLIENT.getList(key);
        rList.addListener(listener);
    }

    /**
     * 获得缓存的list对象
     *
     * @param key 缓存的键值
     * @return 缓存键值对应的数据
     */
    public static <T> List<T> getCacheList(final String key) {
        RList<T> rList = CLIENT.getList(key);
        return rList.readAll();
    }

    /**
     * 缓存Set
     *
     * @param key     缓存键值
     * @param dataSet 缓存的数据
     * @return 缓存数据的对象
     */
    public static <T> boolean setCacheSet(final String key, final Set<T> dataSet) {
        RSet<T> rSet = CLIENT.getSet(key);
        return rSet.addAll(dataSet);
    }

    /**
     * 注册Set监听器
     * <p>
     * key 监听器需开启 `notify-keyspace-events` 等 redis 相关配置
     *
     * @param key      缓存的键值
     * @param listener 监听器配置
     */
    public static <T> void addSetListener(final String key, final ObjectListener listener) {
        RSet<T> rSet = CLIENT.getSet(key);
        rSet.addListener(listener);
    }

    /**
     * 获得缓存的set
     *
     * @param key 缓存的key
     * @return set对象
     */
    public static <T> Set<T> getCacheSet(final String key) {
        RSet<T> rSet = CLIENT.getSet(key);
        return rSet.readAll();
    }

    /**
     * 缓存Map
     *
     * @param key     缓存的键值
     * @param dataMap 缓存的数据
     */
    public static <T> void setCacheMap(final String key, final Map<String, T> dataMap) {
        if (dataMap != null) {
            RMap<String, T> rMap = CLIENT.getMap(key);
            rMap.putAll(dataMap);
        }
    }

    /**
     * 注册Map监听器
     * <p>
     * key 监听器需开启 `notify-keyspace-events` 等 redis 相关配置
     *
     * @param key      缓存的键值
     * @param listener 监听器配置
     */
    public static <T> void addMapListener(final String key, final ObjectListener listener) {
        RMap<String, T> rMap = CLIENT.getMap(key);
        rMap.addListener(listener);
    }

    /**
     * 获得缓存的Map
     *
     * @param key 缓存的键值
     * @return map对象
     */
    public static <T> Map<String, T> getCacheMap(final String key) {
        RMap<String, T> rMap = CLIENT.getMap(key);
        return rMap.getAll(rMap.keySet());
    }

    /**
     * 获得缓存Map的key列表
     *
     * @param key 缓存的键值
     * @return key列表
     */
    public static <T> Set<String> getCacheMapKeySet(final String key) {
        RMap<String, T> rMap = CLIENT.getMap(key);
        return rMap.keySet();
    }

    /**
     * 往Hash中存入数据
     *
     * @param key   Redis键
     * @param hKey  Hash键
     * @param value 值
     */
    public static <T> void setCacheMapValue(final String key, final String hKey, final T value) {
        RMap<String, T> rMap = CLIENT.getMap(key);
        rMap.put(hKey, value);
    }

    /**
     * 获取Hash中的数据
     *
     * @param key  Redis键
     * @param hKey Hash键
     * @return Hash中的对象
     */
    public static <T> T getCacheMapValue(final String key, final String hKey) {
        RMap<String, T> rMap = CLIENT.getMap(key);
        return rMap.get(hKey);
    }

    /**
     * 删除Hash中的数据
     *
     * @param key  Redis键
     * @param hKey Hash键
     * @return Hash中的对象
     */
    public static <T> T delCacheMapValue(final String key, final String hKey) {
        RMap<String, T> rMap = CLIENT.getMap(key);
        return rMap.remove(hKey);
    }

    /**
     * 删除Hash中的数据
     *
     * @param key   Redis键
     * @param hKeys Hash键
     */
    public static <T> void delMultiCacheMapValue(final String key, final Set<String> hKeys) {
        RBatch batch = CLIENT.createBatch();
        RMapAsync<String, T> rMap = batch.getMap(key);
        for (String hKey : hKeys) {
            rMap.removeAsync(hKey);
        }
        batch.execute();
    }

    /**
     * 获取多个Hash中的数据
     *
     * @param key   Redis键
     * @param hKeys Hash键集合
     * @return Hash对象集合
     */
    public static <K, V> Map<K, V> getMultiCacheMapValue(final String key, final Set<K> hKeys) {
        RMap<K, V> rMap = CLIENT.getMap(key);
        return rMap.getAll(hKeys);
    }

    /**
     * 设置原子值
     *
     * @param key   Redis键
     * @param value 值
     */
    public static void setAtomicValue(String key, long value) {
        RAtomicLong atomic = CLIENT.getAtomicLong(key);
        atomic.set(value);
    }

    /**
     * 获取原子值
     *
     * @param key Redis键
     * @return 当前值
     */
    public static long getAtomicValue(String key) {
        RAtomicLong atomic = CLIENT.getAtomicLong(key);
        return atomic.get();
    }

    /**
     * 递增原子值
     *
     * @param key Redis键
     * @return 当前值
     */
    public static long incrAtomicValue(String key) {
        RAtomicLong atomic = CLIENT.getAtomicLong(key);
        return atomic.incrementAndGet();
    }

    /**
     * 递减原子值
     *
     * @param key Redis键
     * @return 当前值
     */
    public static long decrAtomicValue(String key) {
        RAtomicLong atomic = CLIENT.getAtomicLong(key);
        return atomic.decrementAndGet();
    }

    /**
     * 获得缓存的基本对象列表
     *
     * @param pattern 字符串前缀
     * @return 对象列表
     */
    public static Collection<String> keys(final String pattern) {
        Stream<String> stream = CLIENT.getKeys().getKeysStreamByPattern(pattern);
        return stream.collect(Collectors.toList());
    }

    /**
     * 删除缓存的基本对象列表
     *
     * @param pattern 字符串前缀
     */
    public static void deleteKeys(final String pattern) {
        CLIENT.getKeys().deleteByPattern(pattern);
    }

    /**
     * 检查redis中是否存在key
     *
     * @param key 键
     */
    public static Boolean hasKey(String key) {
        RKeys rKeys = CLIENT.getKeys();
        return rKeys.countExists(key) > 0;
    }
}
```

