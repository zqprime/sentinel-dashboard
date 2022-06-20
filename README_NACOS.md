详细内容可看我的博客 [sentinel之dashboard改造](https://www.jianshu.com/p/7b4127e692b9)

### 环境版本相关说明

> 1. 自定义数据源：`nacos-1.4.1`
> 2. sentinel-dashboard版本：`sentinel-1.8.0`
> 3. 通过推模式持久化规则数据到nacos

### 修改点

- pom依赖更改

  ```xml
  <!-- for Nacos rule publisher sample -->
  <dependency>
     <groupId>com.alibaba.csp</groupId>
     <artifactId>sentinel-datasource-nacos</artifactId>
     <!--<scope>test</scope>-->
  </dependency>
  ```

  注释掉 `scope`

- 修改application.properties

  ```yaml
  server.port=8858
  # nacos setting group default SENTINREL_GROUP
  sentinel.datasource.nacos.server-addr=http://localhost:8848
  sentinel.datasource.nacos.namespace=dev
  ```

  我的`nacos`使用`namespace`做环境区分，此处需要指定

- rule模块下新增nacos包，相关文件如下：

* NacosConfig 说明 （参考官网test下的相关文件）
```java
@Configuration
public class NacosConfig {
        @Value("${sentinel.datasource.nacos.server-addr:localhost:8848}")
        private String serverAddr;
        @Value("${sentinel.datasource.nacos.namespace:public}")
        private String namespace;
        @Bean
        public ConfigService nacosConfigService() throws Exception {
            Properties properties = new Properties();
            properties.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
            properties.put(PropertyKeyConst.NAMESPACE, namespace);
            return ConfigFactory.createConfigService(properties);
        }
}
```
此处获取配置中的`nacos`配置，初始化nacos中config模块（ConfigService）操作的依赖bean，并注入当前容器

- NacosConfigUtil 说明（参考官网test下的相关文件）

增加如下规则配置项：

  ```java
  /**
   * add cc for `degrade，authority，system`
   */
  public static final String DEGRADE_DATA_ID_POSTFIX = "-degrade-rules";
  public static final String AUTHORITY_DATA_ID_POSTFIX = "-authority-rules";
  public static final String SYSTEM_DATA_ID_POSTFIX = "-system-rules";
  public static final String GETWAY_API_DATA_ID_POSTFIX = "-gateway-api-rules";
  public static final String GETWAY_FLOW_DATA_ID_POSTFIX = "-gateway-flow-rules";
  ```

- CustomDynamicRule 说明，定义 Nacos数据源 推送，拉取操作

  ```java
  package com.alibaba.csp.sentinel.dashboard.rule.nacos;
  
  import com.alibaba.csp.sentinel.dashboard.util.JSONUtils;
  import com.alibaba.csp.sentinel.util.AssertUtil;
  import com.alibaba.csp.sentinel.util.StringUtil;
  import com.alibaba.fastjson.JSON;
  import com.alibaba.nacos.api.config.ConfigService;
  import com.alibaba.nacos.api.exception.NacosException;
  import com.fasterxml.jackson.core.JsonProcessingException;
  import com.fasterxml.jackson.databind.ObjectMapper;
  import java.util.ArrayList;
  import java.util.List;
  
  /**
   * @Description: 定义 Nacos数据源 推送，拉取操作
   * @Author sisyphus
   * @Date 2021/8/25 15:11
   * @Version V-1.0
   */
  public interface CustomDynamicRule<T> {
  
      /**
      *@Author sisyphus
      *@Description 远程获取规则-nacos数据源
      *@Date 2021/8/25 17:24
      *@Param [configService, appName, postfix, clazz-反序列化类]
      *@return java.util.List<T>
      **/
      default List<T> fromNacosRuleEntity(ConfigService configService, String appName, String postfix, Class<T> clazz) throws NacosException {
          AssertUtil.notEmpty(appName, "app name cannot be empty");
          String rules = configService.getConfig(
                  genDataId(appName, postfix),
                  NacosConfigUtil.GROUP_ID,
                  3000
          );
          if (StringUtil.isEmpty(rules)) {
              return new ArrayList<>();
          }
          return JSONUtils.parseObject(clazz, rules);
      }
      
      /**
       * @title setNacosRuleEntityStr
       * @description  将规则序列化成JSON文本，存储到Nacos server中
       * @author sisyphus
       * @param: configService nacos config service
       * @param: appName       应用名称
       * @param: postfix       规则后缀 eg.NacosConfigUtil.FLOW_DATA_ID_POSTFIX
       * @param: rules         规则对象
       * @updateTime 2021/8/26 15:47 
       * @throws  NacosException 异常
       **/
      default void setNacosRuleEntityStr(ConfigService configService, String appName, String postfix, List<T> rules) throws NacosException{
          AssertUtil.notEmpty(appName, "app name cannot be empty");
          if (rules == null) {
              return;
          }
          String dataId = genDataId(appName, postfix);
  
          //存储，推送远程nacos服务配置中心
          boolean publishConfig = configService.publishConfig(
                  dataId,
                  NacosConfigUtil.GROUP_ID,
                  printPrettyJSON(rules)
          );
          if(!publishConfig){
              throw new RuntimeException("publish to nacos fail");
          }
      }
  
      /**
      *@Author sisyphus
      *@Description 组装nacos dateId
      *@Date 2021/8/25 16:34
      *@Param [appName, postfix]
      *@return java.lang.String
      **/
      default String genDataId(String appName, String postfix) {
          return appName + postfix;
      }
  
      /**
      *@Author sisyphus
      *@Description 规则对象转换为json字符串,并带有格式化
      *@Date 2021/8/25 17:19
      *@Param [obj]
      *@return java.lang.String
      **/
      default String printPrettyJSON(Object obj) {
          try {
              ObjectMapper mapper = new ObjectMapper();
              return mapper.writerWithDefaultPrettyPrinter().writeValueAsString(obj);
          } catch (JsonProcessingException e) {
              return JSON.toJSONString(obj);
          }
      }
  }
  
  ```

- FlowRuleNacosProvider 和 FlowRuleNacosPublisher 说明（参考官网test下的相关文件）

**FlowRuleNacosProvider 代码**

  ```java
  package com.alibaba.csp.sentinel.dashboard.rule.nacos.flow;
  
  import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.FlowRuleEntity;
  import com.alibaba.csp.sentinel.dashboard.rule.DynamicRuleProvider;
  import com.alibaba.csp.sentinel.dashboard.rule.nacos.CustomDynamicRule;
  import com.alibaba.csp.sentinel.dashboard.rule.nacos.NacosConfigUtil;
  import com.alibaba.csp.sentinel.util.AssertUtil;
  import com.alibaba.nacos.api.config.ConfigService;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.stereotype.Component;
  import java.util.List;
  
  /**
   * @author Eric Zhao
   * @since 1.4.0
   */
  @Component("flowRuleNacosProvider")
  public class FlowRuleNacosProvider implements DynamicRuleProvider<List<FlowRuleEntity>>, CustomDynamicRule<FlowRuleEntity> {
  
      @Autowired
      private ConfigService configService;
  
      @Override
      public List<FlowRuleEntity> getRules(String appName) throws Exception {
          AssertUtil.notEmpty(appName, "app name cannot be empty");
          return fromNacosRuleEntity(configService, appName, NacosConfigUtil.FLOW_DATA_ID_POSTFIX, FlowRuleEntity.class);
      }
  }
  ```

**FlowRuleNacosPublisher  代码**

  ```java
  package com.alibaba.csp.sentinel.dashboard.rule.nacos.flow;
  
  import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.FlowRuleEntity;
  import com.alibaba.csp.sentinel.dashboard.rule.DynamicRulePublisher;
  import com.alibaba.csp.sentinel.dashboard.rule.nacos.CustomDynamicRule;
  import com.alibaba.csp.sentinel.dashboard.rule.nacos.NacosConfigUtil;
  import com.alibaba.csp.sentinel.util.AssertUtil;
  import com.alibaba.nacos.api.config.ConfigService;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.stereotype.Component;
  import java.util.*;
  
  /**
   * @author Eric Zhao
   * @since 1.4.0
   */
  @Component("flowRuleNacosPublisher")
  public class FlowRuleNacosPublisher implements DynamicRulePublisher<List<FlowRuleEntity>>, CustomDynamicRule<FlowRuleEntity> {
  
      @Autowired
      private ConfigService configService;
  
      @Override
      public void publish(String app, List<FlowRuleEntity> rules) throws Exception {
          AssertUtil.notEmpty(app, "app name cannot be empty");
          if (rules == null) {
              return;
          }
          setNacosRuleEntityStr(configService, app, NacosConfigUtil.FLOW_DATA_ID_POSTFIX, rules);
      }
  }
  ```

**此处实现与官网案例存在不同，两个类都实现了自定义接口`CustomDynamicRule`,将数据源操作的动作抽象为默认接口，后续如果需要其他数据源的操作，可直接在接口中增加操作方法即可, 其余规则对应操作类与以上类似，仅仅泛型对应匹配的规则类型不同（对应`AuthorityRuleEntity` , `DegradeRuleEntity` ,  `ParamFlowRuleEntity` , `SystemRuleEntity` , `ApiDefinitionEntity` , `GatewayFlowRuleEntity` ）**

- controller 改造（**重点**）

我全部使用新建类，并未在原有类做更改，全部在包 `v2`下

**接下来还是以flow为例进行相关说明**

- 首先增加了抽象类 `InDataSourceRuleStore` ，先看看代码，如下：

    ```java
    public abstract class InDataSourceRuleStore<T extends RuleEntity> {
        private final Logger logger = LoggerFactory.getLogger(InDataSourceRuleStore.class);
    
        /**
        *@Author sisyphus
        *@Description 格式化从外部数据源获取到的数据（RuleEntity下的部分数据字段填充等）
        *@Date 2021/8/25 18:11
        *@Param [entity - 远程获取规则且匹配当前需查询规则的实体对象, app - 当前规则标识]
        *@return void
        **/
        protected abstract void format(T entity, String app);
        /**
        *@Author sisyphus
        *@Description 更新规则时部分数据的整合，字段维护
        *@Date 2021/8/26 14:20
        *@Param [entity, oldEntity]
        *@return void
        **/
        protected abstract void merge(T entity, T oldEntity);
    
        /**
        *@Author sisyphus
        *@Description 根据当前id获取远程匹配的规则实体
         *              此处只对普通流控做了转换，会经过format进行，其余规则直接返回远程规则对象，后面根据具体情况自行转换改造
        *@Date 2021/8/26 10:33
        *@Param [ruleProvider, app, id]
        *@return T
        **/
        protected T findById(DynamicRuleProvider<List<T>> ruleProvider, String app, Long id) {
            try {
                // 远程获取规则(当前种类下（如网关流控，普通流控，系统等）的所有规则数据)
                List<T> rules = ruleProvider.getRules(app);
                // 匹配符合当前查询的规则，格式化远端规则数据为sentinel服务端可使用格式（Entity形式）
                if (rules != null && !rules.isEmpty()) {
                    Optional<T> entity = rules.stream().filter(rule -> (id.equals(rule.getId()))).findFirst();
                    if (entity.isPresent()){
                        T t = entity.get();
                        this.format(t, app);
                        return t;
                    }
                }
            } catch (Exception e) {
                logger.error("服务[{}]规则[{}]匹配远端规则异常：{}", app, id, e.getMessage());
                e.printStackTrace();
            }
            return null;
        }
    
        /**
        *@Author sisyphus
        *@Description 获取对应模块下的所有规则，存在format的进行规则转换
        *@Date 2021/8/26 11:27
        *@Param [ruleProvider, app]
        *@return java.util.List<T>
        **/
        protected List<T> list(DynamicRuleProvider<List<T>> ruleProvider, String app) throws Exception {
            List<T> rules = ruleProvider.getRules(app);
            if (rules != null && !rules.isEmpty()) {
                for (T entity : rules) {
                    this.format(entity, app);
                }
                // 此处排序是为了确保保存（save）时方便获取id以生成nextId
                rules.sort((p1,p2) -> (int) (p1.getId() - p2.getId()));
            } else {
                rules = new ArrayList<>();
            }
            return rules;
        }
    
        /**
        *@Author sisyphus
        *@Description 添加规则至远程数据源
         *              添加前先获取远程数据源，再加入本次新增，一起推送到远程数据源（否则存在覆盖的可能）
         *            修改nextId生成规则，原nextId生成由InMemoryRuleRepositoryAdapter类下nextId()方法实现，内部维护了一个AtomicLong实现自增，每次重启则重新从0开始
        *@Date 2021/8/26 11:43
        *@Param [rulePublisher, ruleProvider, entity]
        *@return void
        **/
        protected void save(DynamicRulePublisher<List<T>> rulePublisher, DynamicRuleProvider<List<T>> ruleProvider, T entity) throws Exception {
            if (null == entity || StringUtils.isEmpty(entity.getApp())) {
                throw new InvalidParameterException("app is required");
            }
            if (null != entity.getId()) {
                throw new InvalidParameterException("id must be null");
            }
            // 获取远程规则数据
            List<T> rules = this.list(ruleProvider, entity.getApp());
            // 增规则添加至集合
            long nextId = 1;
            if (rules.size() > 0) {
                // 获取集合的最后一个元素，得到id，进行增1操作（集合在）list方法内进行过排序，以保证此处获取到的最后一个元素为当前集合内id最大的元素
                nextId = rules.get(rules.size() - 1).getId() + 1;
            }
            entity.setId(nextId);
            rules.add(entity);
            // 推送远程存储源
            rulePublisher.publish(entity.getApp(), rules);
        }
    
        /**
         * @title update
         * @description 获取远程所有模式下的规则，匹配id，进行替换
         * @author sisyphus
         * @param: rulePublisher
         * @param: ruleProvider
         * @param: entity
         * @updateTime 2021/8/30 9:35
         * @return: com.alibaba.csp.sentinel.dashboard.domain.Result<T>
         * @throws
         **/
        protected Result<T> update(DynamicRulePublisher<List<T>> rulePublisher, DynamicRuleProvider<List<T>> ruleProvider, T entity) throws Exception {
            if (null == entity || null == entity.getId() || StringUtils.isEmpty(entity.getApp())) {
                return Result.ofFail(-1, "id is required");
            }
            // 获取远程规则数据
            List<T> rules = this.list(ruleProvider, entity.getApp());
            if (null == rules || rules.isEmpty()) {
                return Result.ofFail(-1, "Failed to save authority rule, no matching authority rule");
            }
            // 远程规则集合与当前规则匹配项，当前规则填充旧的集合中对应规则数据
            for (int i = 0; i < rules.size(); i++) {
                T oldEntity = rules.get(i);
                if (oldEntity.getId().equals(entity.getId())) {
                    // 新旧值替换填充，字段检查
                    this.merge(entity, oldEntity);
                    // 写回规则集合
                    rules.set(i, entity);
                    break;
                }
            }
            // 推送远程存储源
            rulePublisher.publish(entity.getApp(), rules);
            return Result.ofSuccess(entity);
        }
    
        /**
         * @title delete
         * @description 获取远程所有模式下的规则,从集合中删除对应规则项
         * @author sisyphus 
         * @param: rulePublisher
         * @param: ruleProvider
         * @param: id
         * @param: app
         * @updateTime 2021/8/30 9:36 
         * @return: com.alibaba.csp.sentinel.dashboard.domain.Result<java.lang.Long>
         * @throws
         **/
        protected Result<Long> delete(DynamicRulePublisher<List<T>> rulePublisher, DynamicRuleProvider<List<T>> ruleProvider, long id, String app) throws Exception {
            List<T> rules = this.list(ruleProvider, app);
            if (null == rules || rules.isEmpty()) {
                return Result.ofSuccess(null);
            }
            // 匹配删除项，移除集合
            boolean removeIf = rules.removeIf(flowRuleEntity -> flowRuleEntity.getId().equals(id));
            if (!removeIf){
                return Result.ofSuccess(null);
            }
            // 推送远程存储源
            rulePublisher.publish(app, rules);
            return Result.ofSuccess(id);
        }
    }
    ```

**说明：该类的作用，摒弃掉所有内存存储操作**

* **controller类**

 ```java
  @RestController
  @RequestMapping(value = "/v2/flow")
  public class FlowControllerV2 extends InDataSourceRuleStore<FlowRuleEntity>{
  
      private final Logger logger = LoggerFactory.getLogger(FlowControllerV2.class);
  
      @Autowired
  //    @Qualifier("flowRuleDefaultProvider")
      @Qualifier("flowRuleNacosProvider")
      private DynamicRuleProvider<List<FlowRuleEntity>> ruleProvider;
      @Autowired
  //    @Qualifier("flowRuleDefaultPublisher")
      @Qualifier("flowRuleNacosPublisher")
      private DynamicRulePublisher<List<FlowRuleEntity>> rulePublisher;
  
      @GetMapping("/rules")
      @AuthAction(PrivilegeType.READ_RULE)
      public Result<List<FlowRuleEntity>> apiQueryMachineRules(@RequestParam String app) {
  
          if (StringUtil.isEmpty(app)) {
              return Result.ofFail(-1, "app can't be null or empty");
          }
          try {
              /*List<FlowRuleEntity> rules = ruleProvider.getRules(app);
              if (rules != null && !rules.isEmpty()) {
                  for (FlowRuleEntity entity : rules) {
                      entity.setApp(app);
                      if (entity.getClusterConfig() != null && entity.getClusterConfig().getFlowId() != null) {
                          entity.setId(entity.getClusterConfig().getFlowId());
                      }
                  }
              }
              rules = repository.saveAll(rules);*/
              List<FlowRuleEntity> rules = this.list(ruleProvider, app);
              return Result.ofSuccess(rules);
          } catch (Throwable throwable) {
              logger.error("Error when querying flow rules", throwable);
              return Result.ofThrowable(-1, throwable);
          }
      }
  
      private <R> Result<R> checkEntityInternal(FlowRuleEntity entity) {
          if (entity == null) {
              return Result.ofFail(-1, "invalid body");
          }
          if (StringUtil.isBlank(entity.getApp())) {
              return Result.ofFail(-1, "app can't be null or empty");
          }
          if (StringUtil.isBlank(entity.getLimitApp())) {
              return Result.ofFail(-1, "limitApp can't be null or empty");
          }
          if (StringUtil.isBlank(entity.getResource())) {
              return Result.ofFail(-1, "resource can't be null or empty");
          }
          if (entity.getGrade() == null) {
              return Result.ofFail(-1, "grade can't be null");
          }
          if (entity.getGrade() != 0 && entity.getGrade() != 1) {
              return Result.ofFail(-1, "grade must be 0 or 1, but " + entity.getGrade() + " got");
          }
          if (entity.getCount() == null || entity.getCount() < 0) {
              return Result.ofFail(-1, "count should be at lease zero");
          }
          if (entity.getStrategy() == null) {
              return Result.ofFail(-1, "strategy can't be null");
          }
          if (entity.getStrategy() != 0 && StringUtil.isBlank(entity.getRefResource())) {
              return Result.ofFail(-1, "refResource can't be null or empty when strategy!=0");
          }
          if (entity.getControlBehavior() == null) {
              return Result.ofFail(-1, "controlBehavior can't be null");
          }
          int controlBehavior = entity.getControlBehavior();
          if (controlBehavior == 1 && entity.getWarmUpPeriodSec() == null) {
              return Result.ofFail(-1, "warmUpPeriodSec can't be null when controlBehavior==1");
          }
          if (controlBehavior == 2 && entity.getMaxQueueingTimeMs() == null) {
              return Result.ofFail(-1, "maxQueueingTimeMs can't be null when controlBehavior==2");
          }
          if (entity.isClusterMode() && entity.getClusterConfig() == null) {
              return Result.ofFail(-1, "cluster config should be valid");
          }
          return null;
      }
  
      @PostMapping("/rule")
      @AuthAction(value = AuthService.PrivilegeType.WRITE_RULE)
      public Result<FlowRuleEntity> apiAddFlowRule(@RequestBody FlowRuleEntity entity) {
  
          Result<FlowRuleEntity> checkResult = checkEntityInternal(entity);
          if (checkResult != null) {
              return checkResult;
          }
          /*entity.setId(null);
          Date date = new Date();
          entity.setGmtCreate(date);
          entity.setGmtModified(date);
          entity.setLimitApp(entity.getLimitApp().trim());
          entity.setResource(entity.getResource().trim());*/
          try {
              /*entity = repository.save(entity);
              publishRules(entity.getApp());*/
              this.save(rulePublisher, ruleProvider, entity);
          } catch (Throwable throwable) {
              logger.error("Failed to add flow rule", throwable);
              return Result.ofThrowable(-1, throwable);
          }
          return Result.ofSuccess(entity);
      }
  
      @PutMapping("/rule/{id}")
      @AuthAction(AuthService.PrivilegeType.WRITE_RULE)
  
      public Result<FlowRuleEntity> apiUpdateFlowRule(@PathVariable("id") Long id,
                                                      @RequestBody FlowRuleEntity entity) {
          if (id == null || id <= 0) {
              return Result.ofFail(-1, "Invalid id");
          }
         /* FlowRuleEntity oldEntity = repository.findById(id);*/
          FlowRuleEntity oldEntity = this.findById(ruleProvider, entity.getApp(), id);
          if (oldEntity == null) {
              return Result.ofFail(-1, "id " + id + " does not exist");
          }
          if (entity == null) {
              return Result.ofFail(-1, "invalid body");
          }
  
          /*entity.setApp(oldEntity.getApp());
          entity.setIp(oldEntity.getIp());
          entity.setPort(oldEntity.getPort());
          */
          entity.setId(id);
          /*Date date = new Date();
          entity.setGmtCreate(oldEntity.getGmtCreate());
          entity.setGmtModified(date);*/
          Result<FlowRuleEntity> checkResult = checkEntityInternal(entity);
          if (checkResult != null) {
              return checkResult;
          }
          try {
              /*entity = repository.save(entity);
              if (entity == null) {
                  return Result.ofFail(-1, "save entity fail");
              }*/
              return this.update(rulePublisher, ruleProvider, entity);
          } catch (Throwable throwable) {
              logger.error("Failed to update flow rule", throwable);
              return Result.ofThrowable(-1, throwable);
          }
      }
  
      @DeleteMapping("/rule/{id}")
      @AuthAction(PrivilegeType.DELETE_RULE)
      public Result<Long> apiDeleteRule(@PathVariable("id") Long id, @RequestParam("app") String app) {
          if (id == null || id <= 0) {
              return Result.ofFail(-1, "Invalid id");
          }
          if (StringUtils.isEmpty(app)) {
              return Result.ofFail(-1, "Invalid app");
          }
          /*FlowRuleEntity oldEntity = repository.findById(id);
          if (oldEntity == null) {
              return Result.ofSuccess(null);
          }*/
  
          try {
              /*repository.delete(id);*/
              return this.delete(rulePublisher, ruleProvider, id, app);
          } catch (Exception e) {
              return Result.ofFail(-1, e.getMessage());
          }
      }
  
      @Override
      protected void format(FlowRuleEntity entity, String app) {
          entity.setApp(app);
          if (entity.getClusterConfig() != null && entity.getClusterConfig().getFlowId() != null) {
              entity.setId(entity.getClusterConfig().getFlowId());
          }
          Date date = new Date();
          entity.setGmtCreate(date);
          entity.setGmtModified(date);
          entity.setLimitApp(entity.getLimitApp().trim());
          entity.setResource(entity.getResource().trim());
      }
  
      @Override
      protected void merge(FlowRuleEntity entity, FlowRuleEntity oldEntity) {
          entity.setApp(oldEntity.getApp());
          entity.setIp(oldEntity.getIp());
          entity.setPort(oldEntity.getPort());
          Date date = new Date();
          entity.setGmtCreate(oldEntity.getGmtCreate());
          entity.setGmtModified(date);
      }
  }
  ```
**注释部分为原有代码**
* 页面部分更改
    1. `identity.js` 文件中 `app.controller` 下的 `FlowServiceV1` 改为 `FlowServiceV2`
    2. `sidebar.html`修改如下：
  ```html
  <ul class="nav nav-second-level" ng-show="entry.active">
          <li ui-sref-active="active">
            <a ui-sref="dashboard.metric({app: entry.app})">
              <i class="fa fa-bar-chart"></i>&nbsp;&nbsp;实时监控</a>
          </li>

          <li ui-sref-active="active" ng-if="!entry.isGateway">
            <a ui-sref="dashboard.identity({app: entry.app})">
              <i class="glyphicon glyphicon-list-alt"></i>&nbsp;&nbsp;簇点链路</a>
          </li>

          <li ui-sref-active="active" ng-if="entry.isGateway">
            <a ui-sref="dashboard.gatewayIdentity({app: entry.app})">
              <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;请求链路</a>
          </li>

          <!--<li ui-sref-active="active" ng-if="entry.appType==0">
            <a ui-sref="dashboard.flow({app: entry.app})">
              <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则-Nacos</a>
          </li>-->

          <li ui-sref-active="active" ng-if="entry.isGateway">
            <a ui-sref="dashboard.gatewayApi({app: entry.app})">
              <i class="glyphicon glyphicon-tags"></i>&nbsp;&nbsp;&nbsp;API 管理</a>
          </li>
          <li ui-sref-active="active" ng-if="entry.isGateway">
            <a ui-sref="dashboard.gatewayFlow({app: entry.app})">
              <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则</a>
          </li>

          <li ui-sref-active="active" ng-if="!entry.isGateway">
            <a ui-sref="dashboard.flow({app: entry.app})">
              <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则</a>
          </li>

          <!--<li ui-sref-active="active" ng-if="!entry.isGateway">
            <a ui-sref="dashboard.flowV1({app: entry.app})">
              <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则</a>
          </li>-->

          <li ui-sref-active="active">
            <a ui-sref="dashboard.degrade({app: entry.app})">
              <i class="glyphicon glyphicon-flash"></i>&nbsp;&nbsp;降级规则</a>
          </li>
          <li ui-sref-active="active" ng-if="!entry.isGateway">
            <a ui-sref="dashboard.paramFlow({app: entry.app})">
              <i class="glyphicon glyphicon-fire"></i>&nbsp;&nbsp;热点规则</a>
          </li>
          <li ui-sref-active="active">
            <a ui-sref="dashboard.system({app: entry.app})">
              <i class="glyphicon glyphicon-lock"></i>&nbsp;&nbsp;系统规则</a>
          </li>
          <li ui-sref-active="active" ng-if="!entry.isGateway">
            <a ui-sref="dashboard.authority({app: entry.app})">
              <i class="glyphicon glyphicon-check"></i>&nbsp;&nbsp;授权规则</a>
          </li>
          <li ui-sref-active="active" ng-if="!entry.isGateway">
            <a ui-sref="dashboard.clusterAppServerList({app: entry.app})">
              <i class="glyphicon glyphicon-cloud"></i>&nbsp;&nbsp;集群流控</a>
          </li>

          <li ui-sref-active="active">
            <a ui-sref="dashboard.machine({app: entry.app})">
              <i class="glyphicon glyphicon-th-list"></i>&nbsp;&nbsp;机器列表</a>
          </li>
        </ul>
  ```
    3. 接下来是包`webapp-resources-app-scripts-service`下对应控制规则的js做修改，修改部分主要为delete方
       法，因为我们对删除方法做过更改，删除逻辑变为从远程获取所有规则，删除当前匹配规则，推送远程,修改
       部分为参数增加参数`app`
  ```javascript
  this.deleteRule = function (rule) {
        var param = {
            app: rule.app
        };
        return $http({
            url: '/v2/flow/rule/' + rule.id,
            params: param,
            method: 'DELETE'
        });
    };
  ```
  依次修改`flow_service_v2.js`, `authority_service.js`, `degrade_service.js`, `param_flow_service.js`,
  `systemservice.js`, `api_service.js`, `flow_service.js` , 后两个js在gateway下
    4. 修改页面或者JS后不生效的话，可能需要重新编译angular
  > * 找到 `package.json` 文件，鼠标右键 `Run 'npm install'` ,待依赖安装完成，如果直接在`webapp/resource`下执行 `npm run build`，会报错缺失`gulp`
  > * 执行 `npm run build` ，完成编译，文件就会更新了
  
改造过程碰到问题颇多，感谢 [github-franciszhao](https://github.com/franciszhao/sentinel-dashboard-nacos/tree/3d8249ac85802e5fbea824e8506fa4234ebe5b20) 和官方文档 [在生产环境中使用 Sentinel](https://github.com/alibaba/Sentinel/wiki/%E5%9C%A8%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E4%B8%AD%E4%BD%BF%E7%94%A8-Sentinel) 提供相关帮助