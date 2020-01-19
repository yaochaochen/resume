#                       个人简历

# 联系方式

- 手机:  **17600370872**

- Email: **chaochenyao@gmail.com**

- 微信号:**chen1015412**

  ------



# 个人信息

- 姚超臣/男/1994
- 大专/计算机技术管理
- 工作年限: **5.5年**
- GitHub: https://github.com/yaochaochen
- 期望职位: Java 高级开发工程师，开发组组长
- 期望薪资: 税前20K~25K
- 期望城市:北京/杭州 

------

# 工作经历



## 广州悦途网络科技股份有限公司（2017年5月至今）

2017年入职此公司的北京办公室，入职时研发部只有三个研发伙伴，在三年里发展到60余人。

正在公司的发展中自己得到很多成长，从入职小白成为能扛起 **SURM MASTER**（类似于组长） 职责。在此期间带领伙伴按时按量完成以下项目



### 悦途接待系统（Web/App）

我在此项目负责了 GateWay设计、服务单、购物、支付、退款、商品管理等模块

在这个项目中使用很多设计模型，抽离核心业务层保证代码的高解耦，比如在退款中，存在很多种类型对接商家的接口，在业务层使用策略设计模式，根据退款方式解耦负重代码实现，为此实现了低耦合，易扩展的退款业务。代码ULM如下：

![Package strategy](/image/Package strategy.png)

在 GateWay 层完成了接口 token 桶令牌方式进行API签名、API 接口鉴权和流量控制。比如 API 校验签名 代码如下:

```java
private void checkSign(ServerHttpRequest request, String body, String accountCode) {
    String url = request.getURI().getRawPath();
    List<String> notAuthSignUriList = Lists.newArrayList(properties.getAuthPermitList());
    notAuthSignUriList.addAll(properties.getWebSocketList());
    if (!handler.match(notAuthSignUriList, url)) {
        if (util.isBlank(accountCode)) {
            throw new YueTokenExpiredException("凭据已失效, 请登录!");
        }
        String method = Objects.requireNonNull(request.getMethod()).name();
        MediaType contentType = request.getHeaders().getContentType();
        String contentTypeStr = contentType != null ? contentType.toString() : "";
        YueUriWrapper uri = new YueUriWrapper(request.getURI());
        String path = uri.getRawPath();
        String auth = uri.getAuthority();
        Map<String, Object> params = Maps.newHashMap();
        request.getQueryParams().forEach((k, v) -> params.put(k, v.get(0)));
        StringBuilder queryParamsString = new StringBuilder();
        Map<String, Object> queryParamsMap = Maps.newHashMap();
        request.getQueryParams().forEach((k, v) -> queryParamsMap.put(k, v.get(0)));
        Set<String> signOrderSet = CommonUtil.sortMapByKey(queryParamsMap).keySet();
        signOrderSet.remove("sign");
        signOrderSet.forEach(k -> {
            // 排序后的key进行取值拼装签名串
            Object oo = queryParamsMap.get(k);
            queryParamsString.append(k).append("=").append(oo).append("&");
        });
        StringBuilder sbr = new StringBuilder();
        sbr.append(method).append("\n").append(auth).append("\n").append(path).append("\n");
        if (queryParamsString.length() > 0) {
            sbr.append(queryParamsString.substring(0, queryParamsString.length() - 1));
        }
        if ((MediaType.APPLICATION_JSON.includes(contentType) || MediaType.APPLICATION_FORM_URLENCODED.includes(contentType))
                && (util.isNotBlank(body) && body.length() > 0)) {
            sbr.append("\n").append(contentTypeStr).append("\n");
            sbr.append(Base64.getEncoder().encodeToString(body.getBytes()));
        } else {
            body = null;
        }
        String sign = (String) params.get("sign");
        if (util.isBlank(sign)) {
            Map<String, String> mes = Maps.newHashMap();
            mes.put("错误信息", "签名为空");
            throw new YueSignIllegalException(GateWayDict.ILLEGAL_SIGN.getDesc(), mes);
        }
        AccountRes account = handler.getAccountResRedis(accountCode, false);
        String str = CommonUtil.hmacSha256Hashing(sbr.toString(), account.getPublicKey());
        checkSign(str, sign, account, sbr, body);
    }
}
```

在这个项目中，我遇到最大困难是如何保证在春运及小长假期间保证每日10万+的接待量，在项目上线后的一段时间经历一次系统宕机事件，为此在找到接待服务挂掉，采取重构接待部分代码，前期使用 FutureTask 异步完成业务流程，减少同步链路，使得服务稳定。后期使用 I/O 模型之多路 I/O 复用的网络框架 以及 **Protobuf** 序列化方案引入项目中代替传统的 JSON 序列化。以此完成服务调用毫秒响应(有点夸大了)。

我在这个项目中最自豪的技术实现细节

1. 中间件方面:

   使用 Kafka 中在不增加 Partition 的同时提升 consumer 处理消息的并行度。方法如下:

   使用JDK1.5中 ThreadPoolExecutor 线程池。 它有两个重要的参数：coreThreadCount 和 maxThreadCount。把接收的消息的丢进线程池中，把原本串行的消费消息流程变成并行的消费。提升消息消费的吞吐量。

2. SQL 方面:

   在使用PgSQL10+ 版本的前提下使用 其中特性以及在了解索引原理同时编写许多毫秒响应的SQL

   

```sql
-- 避免 in 查询 使用 regexp_split_to_table 函数 效率比In快100倍
SELECT A.platform_check_record_code FROM ( SELECT regexp_split_to_table( ?1, ',' ) AS svc_no ) AS tt JOIN svc_order A ON tt.svc_no = A.svc_no
```



```sql
--使用 DISTINCT 关键字 避免多表关联
--- AND e.refundNo ~ 模糊查询 使得走 B-Tree 索引
SELECT DISTINCT e.refundStatus ,e.refundNo,e.opStaffName,d.audit_status AS auditStatus,e.createTime,e.contactName,e.contactPhone,e.distributorCode,e.opHallCode FROM ( SELECT DISTINCT e.contact_phone AS contactPhone,e.contact_name AS contactName,A .refund_no AS refundNo,to_char(A.create_time, 'YYYY-MM-DD HH24:MI') AS createTime, A.refund_status AS refundStatus,A .op_staff_name AS opStaffName,A.distributor_code AS distributorCode,A.op_hall_code AS opHallCode FROM order_info e JOIN refund A ON e.order_no = A .order_no ) e JOIN ( SELECT b.audit_status,b.refund_no,b.op_staff_name FROM refund_log b WHERE b.create_time = ( SELECT MAX (create_time) FROM refund_log C WHERE b.refund_no = C .refund_no )) d ON e.refundNo = d.refund_no
where  AND e.refundNo ~ 
```

3. 使用 APO 完成 参数校验 异常统一处理

   ```java
   private StringBuilder verifyYueApiParamAnnotation(StringBuilder paramSb,
                                                     Map<String, Map<YueApiParamAnnotation, Object>> paramAnnMap) {
       paramAnnMap.forEach((k, v) -> {
           String name = "Authorization".equals(k) || "Version".equals(k) ? k.toLowerCase() : k;
           v.forEach((k2, v2) -> {
               if (k2.getRequired() && (v2 == null || (v2 instanceof String && (util.isBlank((String) v2) || "undefined".equals(v2))))) {
                   paramSb.append("参数").append(name).append("不能为空!");
               } else {
                   patternMatches(paramSb, k2.getAccess(), v2, name);
               }
           });
       });
       return paramSb;
   }
   private void patternMatches(StringBuilder paramSb, String access, Object value, String name) {
           if (util.isNotBlank(access) && value != null && String.class.isAssignableFrom(value.getClass())) {
               // 正则校验
               boolean isMatch = Pattern.matches(access, value.toString());
               if (!isMatch) {
                   paramSb.append("参数").append(name).append("格式不正确!");
               }
           }
       }
   ```

 异常处理

```java
public class YueBusinessLogicException extends YueException {

    public YueBusinessLogicException(String msg) {
        super(msg);
    }

    public YueBusinessLogicException(String msg, String businessType) {
        super(msg, businessType);
    }

}
```

在项目上线之前 接待员使用手工记账方式比较消耗时间精力，运营人员不能准确统计数据，在上线后的不断完善，从前方到后方极大的提高接待能力和运营管理能力。

### ERP（基于用友 U8ERP 融合自己特色采购系统）





------



## 北京中海纪元数字技术有限公司（2014年12月2017年5月）







# 技术文章

## Dive系列-深入学习笔记

- [Java-Dive](https://github.com/yaochaochen/note/blob/master/spring-dive/)

- [Spring-Dive](https://github.com/yaochaochen/note/blob/master/spring-dive/)

- [Spring-Boot-Dive](https://github.com/yaochaochen/note/blob/master/spring-boot-dive)
- [每日SQL](https://github.com/yaochaochen/note/blob/master/sql/SQL%E6%AF%8F%E6%97%A5%E4%B8%80%E9%A2%98.md](https://github.com/yaochaochen/note/blob/master/sql/SQL每日一题.md)

------



## 国外书籍阅读

- [J2EE.Development.without.EJB](https://github.com/yaochaochen/note/tree/master/书籍)
- [Spring-Integration-for-EAI](https://github.com/yaochaochen/note/blob/master/国外面试题/Spring-Integration-for-EAI.pdf)
- [JSR规约](https://github.com/yaochaochen/jsr)

------

## 演讲和讲义

- Mock使用分享 
- [机器学习简单应用分享](https://github.com/yaochaochen/note/blob/master/Machine/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E7%AE%97%E6%B3%95%E5%9F%BA%E7%A1%80.md](https://github.com/yaochaochen/note/blob/master/Machine/机器学习算法基础.md)
- [Spring IoC讨论](https://github.com/yaochaochen/note/blob/master/spring-dive/Spring%20IOC%20%E5%AE%B9%E5%99%A8%E6%A6%82%E8%BF%B0.md]) 

## 技能清单
