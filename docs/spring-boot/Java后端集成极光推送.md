## java 后端集成极光推送

极光官网：https://www.jiguang.cn/


极光uniapp集成： https://blog.csdn.net/qq_44930306/article/details/129085708

GIT官网：https://github.com/jpush



1、注册开通极光服务

2、SpringBoot集成极光SDK

(1) 添加极光java sdk依赖

```xml
		<dependency>
            <groupId>cn.jpush.api</groupId>
            <artifactId>jpush-client</artifactId>
            <version>3.4.3</version>
        </dependency>

        <dependency>
            <groupId>cn.jpush.api</groupId>
            <artifactId>jiguang-common</artifactId>
            <version>1.1.7</version>
        </dependency>
```

（2）yml文件配置

```
jpush:
  appKey: xxxxxx
  secret: xxxxxx
```

（3）JPushConfiguration 配置类注册JPushClient Bean

```java
import cn.jpush.api.JPushClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;

@Configuration
public class JPushConfiguration {

	@Value("${jpush.appKey}")
	private String appKey;

	@Value("${jpush.secret}")
	private String secret;

	private JPushClient jPushClient;

	@PostConstruct
	public void initJPushClient() {
		jPushClient = new JPushClient(secret, appKey);
	}

	public JPushClient getJPushClient() {
		return jPushClient;
	}
}
```

(4)  PushVO 消息对象定义

```java
import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;
import lombok.Data;
import lombok.EqualsAndHashCode;
import lombok.experimental.Accessors;

import java.util.Map;

@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@ApiModel(value="PushVO", description="推送消息")
public class PushVO {

	@ApiModelProperty(value = "必填, 通知内容, 内容可以为空字符串，则表示不展示到通知栏")
	private String alert;

	@ApiModelProperty(value = "可选, 附加信息, 供业务使用。")
	private Map<String, String> extras;

	@ApiModelProperty(value = "android专用 可选, 通知标题如果指定了，则通知里原来展示 App名称的地方，将展示成这个字段。")
	private String title;

}
```

（5） 极光推送API接口

```java
import cn.jiguang.common.resp.APIConnectionException;
import cn.jiguang.common.resp.APIRequestException;
import cn.jpush.api.push.PushResult;
import cn.jpush.api.push.model.Platform;
import cn.jpush.api.push.model.PushPayload;
import cn.jpush.api.push.model.audience.Audience;
import cn.jpush.api.push.model.notification.Notification;
import com.moemil.bodyhealth.home.common.config.JPushConfiguration;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

/**
 * 极光推送sdk接口
 * @author liuyichun
 */
@Service
@Slf4j
public class JPushService {

	//一次推送最大数量 (极光限制1000)
	private static final int max_size = 800;

	@Autowired
	private JPushConfiguration jPushConfig;

	/**
	 * 广播 (所有平台，所有设备, 不支持附加信息)
	 * @param pushBean PushVO
	 * @return boolean
	 */
	public boolean pushAll(PushVO pushBean) {
		return sendPush(PushPayload.newBuilder()
			.setPlatform(Platform.all())
			.setAudience(Audience.all())
			.setNotification(Notification.alert(pushBean.getAlert()))
			.build());
	}

	/**
	 * 推送全部ios ios广播
	 * @param pushBean PushVO
	 * @return boolean
	 */
	public boolean pushIos(PushVO pushBean) {
		return sendPush(PushPayload.newBuilder()
			.setPlatform(Platform.ios())
			.setAudience(Audience.all())
			.setNotification(Notification.ios(pushBean.getAlert(), pushBean.getExtras()))
			.build());
	}


	/**
	 * 推送ios 指定id
	 * @param pushBean PushVO
	 * @param registrationIds String
	 * @return boolean
	 */
	public boolean pushIos(PushVO pushBean, String... registrationIds) {
		// 剔除无效 registid
		registrationIds = checkRegistrationIds(registrationIds);
		// 每次推送max_size个
		while (registrationIds.length > max_size) {
			sendPush(PushPayload.newBuilder()
				.setPlatform(Platform.ios())
				.setAudience(Audience.registrationId(Arrays.copyOfRange(registrationIds, 0, max_size)))
				.setNotification(Notification.ios(pushBean.getAlert(), pushBean.getExtras()))
				.build());
			registrationIds = Arrays.copyOfRange(registrationIds, max_size, registrationIds.length);
		}
		return sendPush(PushPayload.newBuilder()
			.setPlatform(Platform.ios())
			.setAudience(Audience.registrationId(Arrays.copyOfRange(registrationIds, 0, max_size)))
			.setNotification(Notification.ios(pushBean.getAlert(), pushBean.getExtras()))
			.build());
	}

	/**
	 * 推送到全部的android客户端
	 * @param pushBean PushVO
	 * @return boolean
	 */
	public boolean pushAndroid(PushVO pushBean) {
		return sendPush(PushPayload.newBuilder()
			.setPlatform(Platform.android())
			.setAudience(Audience.all())
			.setNotification(Notification.android(pushBean.getAlert(), pushBean.getTitle(), pushBean.getExtras()))
			.build());
	}


	/**
	 * 推送到Android指定用户
	 * @param pushBean PushVO
	 * @param registrationIds String
	 * @return boolean
	 */
	public boolean pushAndroid(PushVO pushBean, String... registrationIds) {
		// 剔除无效 registrationId
		registrationIds = checkRegistrationIds(registrationIds);
		while (registrationIds.length > max_size) { // 每次推送max_size个
			sendPush(PushPayload.newBuilder()
				.setPlatform(Platform.android())
				.setAudience(Audience.registrationId(Arrays.copyOfRange(registrationIds, 0, max_size)))
				.setNotification(Notification.android(pushBean.getAlert(), pushBean.getTitle(), pushBean.getExtras()))
				.build());
			registrationIds = Arrays.copyOfRange(registrationIds, max_size, registrationIds.length);
		}
		return sendPush(PushPayload.newBuilder()
			.setPlatform(Platform.android())
			.setAudience(Audience.registrationId(registrationIds))
			.setNotification(Notification.android(pushBean.getAlert(), pushBean.getTitle(), pushBean.getExtras()))
			.build());
	}

	/**
	 * 删除无效的用户
	 * @param registrationIds String[]
	 * @return String[]
	 */
	public String[] checkRegistrationIds(String[] registrationIds) {
		List<String> regList = new ArrayList<>(registrationIds.length);
		for (String registrationId : registrationIds) {
			if (registrationId != null && !"".equals(registrationId.trim())) {
				regList.add(registrationId);
			}
		}
		return regList.toArray(new String[0]);
	}


	/**
	 * 消息推送核心api
	 * @param pushPayload PushPayload
	 * @return boolean
	 */
	public boolean sendPush(PushPayload pushPayload) {
		PushResult result = null;
		try {
			result = jPushConfig.getJPushClient().sendPush(pushPayload);
		} catch (APIConnectionException e) {
			log.error("极光推送连接异常: ", e);
		} catch (APIRequestException e) {
			log.error("极光推送请求异常: ", e);
		}
		if (result != null && result.isResultOK()) {
			log.info("极光推送请求成功: {}", result);
			return true;
		} else {
			log.info("极光推送请求失败: {}", result);
			return false;
		}
	}
}
```

