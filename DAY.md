## 2019-01-07

* MyBatis xml中注意如果实体中某些字段为空，则插入时将无视数据库默认值而插入空值

* MySQL添加/删除索引(非主键索引可以自命名，规范：uniq_字段名/idx_字段名)：

  ```mysql
  ALTER TABLE aaa ADD INDEX idx_a_bb_c(a,bb,c);
  ALTER TABLE aaa ADD UNIQUE uniq_stuid(stu_id);
  ALTER TABLE aaa ADD PRIMARY KEY(a);
  ALTER TABLE plat_stu_course_attach_info DROP INDEX idx_stuid;
  ```

## 2019-01-10

* 树形结构json数据拼装
* CONCAT(01,00) => 10

## 2019-01-11

* MyBatis xml <where>标签会自动将其后第一个条件的and或者是or给忽略掉

## 2019-01-14

* mysql新增记录，日期字段可使用字符串表示-'1994-08-17'、'19940817'

## 2019-01-22

* Controller接收页面List数据方法：
  1. 前台传输：data:json_data
       后台接收：batchAdjustUserCourse(@RequestBody List<TimeTableAdjustVo> timeTableAdjustVos)
       备注：只适用于POST方法提交-get方式中参数中的双引号会被编码导致传到后台不再是json串格式，导致解析出
  2. 前台传输：data:{"datas":data},//或者data:{"datas[]":data}
       后台接收：test(@RequestParam("datas[]") ArrayList[] ids)
  3. -----------后续和杨帆验证
* myBatis xml判断字符串是否相等时不能使用`<if test="flag=='2'">`,因为2会被解析为字符char类型。可使用`'2'.toString()`或者单双引号交换位置来达到效果

## 2019-01-24

* 拼装返回JSON时，往result中塞入Map时需要先将Map转为JSON再塞，否则使用JSON.toJSONString传到前台将无法识别为JSON
  `result.put("data", JSONObject.toJSON(conflictResult));`

* ImageIcon类有神奇的缓存，使用时需要清理缓存后再次获取对象：

  ```java
  ImageIcon imgIcon = new ImageIcon(smallImgPath); imgIcon.getImage().flush();//flush后重新获取，否则一直有缓存数据
  imgIcon = new ImageIcon(smallImgPath);
  ```

## 2019-03-11

※好久不见。。你是真的辣鸡

* 项目成功启动，但一查数据库就显示无法创建连接：errorCode 0, state 08001——导致的原因有多种可能，当时是连接自己笔记本的数据库，一样的参数配置，CRM可以，ACT不行，最后发现是MySQL驱动jar包版本与数据库版本不兼容导致

## 2019-03-20

* MySQL中tinyint是用于表示Boolean类型的数据格式，MyBatis查询出来并使用JSONObject接收的话，0会变成false；1会变成true——可通过select 字段*1 来解决

* sum函数在数据库是number类型的，你的代码中可以使用任何装的下数字型都可以接收。如：sum的值小于java中int的最大值，你就可以用int接收；如果大于Int的最大值而小于double的最大值，你就可以用double。一般在程序设计时，如果不确定它的值范围，可以用long型接收。

* NULL值在sql中不能进行任何操作：

  - NUULL参与算术运算，值都为NULL

  - NULL参与比较运算，结果都为false

  - NULL参与聚集运算（count除外），结果都为NULL

  - NOT IN中含有NULL值时不会返回任何数据:

    ```sql
    SELECT
    	*
    FROM
    	dbo.TableA AS a
    WHERE
    	a.id NOT IN ( 2, NULL );
    -- 等同于:
    SELECT
    	*
    FROM
    	Table_A AS a
    WHERE
    	a.id <> 2
    	AND a.ID <> NULL
    -- 由于NULL值不能参与比较运算符，导致条件不成立，查询不出来数据
    ```

## 2019-03-31

* 备份表结构（包含相关键，B预先不存在）
  `CREATE TABLE B_Table LIKE A_Table;`
  备份表结据（把表A数据备份到表B中，B表结构与A结构一样）
  `INSERT INTO B_Table  SELECT * FROM  A_Table;`

## 2019-04-07

* <? extends T>
  上界通配符，set方法失效——不能确定里面存放的到底是T的子类中哪个具体类，只能取不能存
  <? super T>
  下界通配符，get方法失效——除非用Object类型接收，只能存不方便取

* Java泛型PECS原则
  Producer Extends Consumer Super，参数化类型如果表示一个生产者，向外提供数据则使用extends；如果表示一个消费者，使用消费外部数据则使用super

* JDK 8 Collections.copy()源码：

  ```java
  public static <T> void copy(List<? super T> dest, List<? extends T> src) {
      int srcSize = src.size();
      if (srcSize > dest.size())
          throw new IndexOutOfBoundsException("Source does not fit in dest");
      
      if (srcSize < COPY_THRESHOLD || (src instanceof RandomAccess && dest instanceof RandomAccess)) {
          for (int i=0; i<srcSize; i++)
              dest.set(i, src.get(i));
      } else {
          ListIterator<? super T> di=dest.listIterator();
          ListIterator<? extends T> si=src.listIterator();
          for (int i=0; i<srcSize; i++) {
              di.next();
              di.set(si.next());
          }
      }
  }
  ```

## 2019-05-24

* 使用string的split(String regex)方法时需注意，参数regex实际是指一个正则表达式，故使用./*^|等在正则表达式中有特殊含义的符号作为分隔符时需要使用\\加以转义

* nginx代理下获取ip：

  1. 需要在nginx的配置文件中增加 

     `proxy_set_header X-real-ip           $remote_addr;`
     位置要注意, 可以放在 location / 中, 如果你配置了.do, .action的转发, 那就要放在 .do,.action的配置中

  2. 取ip的方法,先取 X-real-ip的值：

     ```java
     private String getIpAddr(HttpServletRequest request) {
     	String ip = request.getHeader("X-real-ip");//先从nginx自定义配置获取
     	if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
     		ip = request.getHeader("x-forwarded-for");
     	}
     	if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
     		ip = request.getHeader("Proxy-Client-IP");
     	}
     	if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
     		ip = request.getHeader("WL-Proxy-Client-IP");
     	}
     	if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
     		ip = request.getRemoteAddr();
     	}
     	return ip;
     }
     ```

## 2019-06-06

* IN (x); x中包含null时会查不到数据
* Mybatis判断bug：
  1. Integer类型参数值为0时，i==''结果是true
  2. BigDecimal类型参数值为0时，i==null结果是true
* `int p = 4396; double h = p/100;// h = 43.0`

## 2019-07-04

* mybatis通过resultMap构造复杂对象（结果类中包含类型为List的属性），不能直接使用limit做分页
* 默认情况下，mybatis 的 update 操作返回值是记录的 matched 的条数，并不是影响的记录条数——update 更新与原字段相同的值，数据库不会进行更新操作——通过 JDBC URL 显式的指定 useAffectedRows 选项，可以得到受影响的记录的条数

## 2019-09-12

* MySQL参数innodb_lock_wait_timeout设置了锁等待的超时时间（单位：秒，默认50），而参数innodb_rollback_on_timeout则设置了超时后的回滚策略：0表示仅放弃最后一条语句（即等待锁造成超时的语句）-事务不回滚，仍可继续执行并提交；1表示回滚整个事务，5.6版本回滚后将开启一个新的事务，5.7则不再自动开启新事务——这是SQL级层面，即使innodb_lock_wait_timeout为OFF，你也可以在应用程序层面捕获锁等待超时然后回滚整个事务

* MyBatis在传入单个参数判断空值时需要使用内置对象`_parameter`

  ```xml
  <if test="_parameter != null">
  	id = #{id}
  </if>
  ```

## 2019-09-19

* myBatis foreach嵌套

  ```xml
  <insert id="batchInsertOrgMenu" parameterType="java.util.List">
  	INSERT INTO
  		plat_bk_org_menu
  		(
  			org_id,
  			menu_id,
  			create_time
  		)
  	VALUES
  	<foreach collection="list" item="om" separator=",">
  		<foreach collection="om.orgIds" item="orgId" separator=",">
  		(
  			#{orgId},
  			#{om.menuId},
  			NOW()
  		)
  		</foreach>
  	</foreach>
  </insert>
  ```

## 2019-10-08

* finally:

  1. 无论何种情况，finally中的语句一定执行

  2. finally是在函数返回前——return后跟的表达式执行后执行的

  3. 执行finally前，要返回的值被保存起来，无论finally中代码如何执行都不影响此返回值（除非返回值是Map等）——函数返回值是在finally执行前确定的

     ```java
     // 输入参数1，得到结果为1，finally中打印i为4
     static int testFinally(int i){
         try {
     		return i++;
     	} catch (Exception e) {
     		
     	}finally {
     		System.out.println("#");
     		i = i * 2;
     		System.out.println("finally:" + i);
     	}
         return -1;
     }
     ```

## 2019-10-24

* MyBatis参数为数组时，parameterType可以不写，或者写 parameterType="java.util.List" ，参数是整数数组还可以写成 parameterType="Integer[]" 

# 2019-11-05

* ```sql
  -- a为唯一索引时：
  INSERT INTO tab(a,b) VALUES (1,2),(4,5) ON DUPLICATE KEY UPDATE a=a+1;--(2,2) (5,5)
  INSERT INTO tab(a,b) VALUES (1,2),(4,5) ON DUPLICATE KEY UPDATE a=VALUES(a)+VALUES(b);-- (3,1) (9,5)
  REPLACE INTO tab(a,b) VALUES (1,2),(4,5);-- 先DELETE后INSERT
  ```

*  Mybatis查询无结果时返回值：
  * List、Map类型会先执行new语句再赋值，因此返回的结果不会为null
  * 普通类不会执行new语句直接赋值，返回的结果就为null 

# 2019-12-3

* json请求，含有set等集合数据时直接放入，在整个转jsonString

  ```
  TreeMap<String, Object> reqMap = new TreeMap<>();
  reqMap.put("nickName", "Robert");
  Set<Integer> set = new HashSet<>();
  set.add(1);
  reqMap.put("langs", set);
  String reqJson = JSON.toJSONString(reqMap);
  ```

* 注意collection数据在转string时每个元素前会自动附加空格——AbstractCollection.toString()

* mybatis数组、集合参数的非空判断：

  ```xml
  <if test="arr != null and arr.length > 0">sql</if>
  <if test="col != null and col.size() > 0">sql</if>
  ```

# 2019-12-17

* ```xml
  <dependency>
  	<groupId>org.apache.shardingsphere</groupId>
  	<artifactId>sharding-jdbc-spring-boot-starter</artifactId>
  	<version>4.0.0-RC3</version>
  </dependency>
  <dependency>
  	<groupId>org.apache.shardingsphere</groupId>
  	<artifactId>sharding-jdbc-core</artifactId>
  	<version>4.0.0-RC3</version>
  </dependency>
  ```

  使用非springboot的方式（在application.properties/application.yml中配置）使用shardingsphere时必须使用下面 sharding-jdbc-core jar包，使用 sharding-jdbc-spring-boot-starter jar包时，项目会要求必须在application.properties/application.yml中配置，否则报错

* nginx默认配置有一项：

  ```
  server { 
      ... 
      location / { 
          try_files $uri $uri/ /index.html; 
      } 
      #try_files:按顺序检查文件是否存在，返回第一个找到的文件或文件夹(结尾加斜线表示为文件夹)，如果所有的文件或文件夹都找不到，会进行一个内部重定向到最后一个参数
      ... 
  } 
  ```

  当nginx收到不存在的资源请求时，将试图以/index.html来解析文件。但若/index.html亦不存在，则会再次进入此location造成死循环，错误日志：rewrite or internal redirection cycle while internally redirecting to...，前台500 Internal Server Error

  为避免这种情况可将配置改为：`try_files $uri $uri/=404;`这样如果访问不存在的资源将直接返回404

## 2020-01-7

* MySQL有很多关键字，其中保留关键字用户如果使用则必须使用转义符`引用（eg:SELECT），非保留关键字用户可以自由使用而无需引用

  **BUT**，在ShardingSphere最新的SQL解析引擎（Antlr4）中，部分包含非保留关键字的SQL会导致解析错误造成异常，eg：

  ```mysql
  -- 此sql的第11行res.source的source会被识别为关键字造成解析失败
  SELECT 
    sou.channel_code channelCode,
    COUNT(res.id) reservationAmount,
    SUM(IF(res.is_contact = 0, 1, 0)) vaildReservationAmount,
    pos.school_name schoolName,
    pos.id schoolId 
  FROM
    plat_reservation res 
    INNER JOIN plat_source_channel sou 
      ON sou.id = res.source 
    INNER JOIN plat_org_school pos 
      ON pos.id = res.school_id 
  WHERE sou.channel_code IS NOT NULL 
    AND sou.school_id = ? 
    AND DATE(res.create_time) >= DATE(?) 
    AND DATE(res.create_time) <= DATE(?) 
    AND res.school_id = ? 
    AND res.source IS NOT NULL 
  GROUP BY sou.channel_code ;
  ```

  ps:不是所有包含非保留关键字的sql都会异常-应该与解析引擎的解析规则有关；平常使用还是应该避开关键字

* 静态方法调用Spring容器的bean-bean的实例必须也为静态：

  ```java
  private static TestService tService;
  
  // 方法一
  @Autowired
  private TestService testServiceImpl;
  @PostConstruct
  public void initialize(){
      tService = testServiceImpl;
  }
  
  //方法二
  // 类上使用@Component注解——@Component会调用构造函数
  public void setTService(TestService t){
      tService = t;
  } 
  ```

##  2020-4-1

* MySQL触发器&事件&存储过程

  1. 触发器：

     * 监视某种情况，并触发某种操作

     * 示例：

       ```mysql
       DROP TRIGGER IF EXISTS `user_score_optimize`;
       -- 示例1
       CREATE TRIGGER user_score_optimize AFTER UPDATE ON adt_user_score FOR EACH ROW
       BEGIN
       		UPDATE adt_user_score us
       		LEFT JOIN (
       		SELECT
       			SUM( optimize_line ) tLines,
       			SUM( optimize_steps ) tSteps,
       			user_id,
                   `mode`
       		FROM
       			adt_game_info 
       		WHERE
       			game_id = 4 
       		GROUP BY
       			user_id,
       		MODE 
       		) t ON t.user_id = us.user_id 
       		AND us.`mode` = t.`mode` 
       		SET us.optimize_line = t.tLines,
       		us.optimize_steps = t.tSteps 
       	WHERE
       		us.optimize_line != t.tLines 
       		OR us.optimize_steps != t.tSteps;
       END;
       -- 示例2 注意‘new’的使用-相应的，还能用‘old’—— 都是用以引用触发器中发生变化的记录内容
       CREATE TRIGGER user_score_optimize BEFORE UPDATE ON adt_user_score FOR EACH ROW SET new.optimize_line = (SELECT SUM(optimize_line) FROM adt_game_info WHERE game_id = 4 AND user_id = new.user_id AND `mode` = new.`mode`);
       ```

     * 上面示例1会造成`Can't update table 'adt_user_score' in stored function/trigger because it is already used by statement which invoked this stored function/trigger.`，因为在触发器里对刚刚插入的数据进行了insert/update，造成了循环调用。此时应该将`update`改为`set`进行操作-如示例2

  2. 事件
     * 定时任务

     * 示例：

       ```mysql
       DROP EVENT IF EXISTS user_score_optimize;
       CREATE DEFINER = `leap_ad_hbc`@`%` EVENT `leap_adventure`.`user_score_optimize`
       ON SCHEDULE
       EVERY '20' SECOND STARTS '2020-04-01 10:42:45'
       ON COMPLETION PRESERVE
       DO BEGIN
       		UPDATE adt_user_score us
       		LEFT JOIN (
       		SELECT
       			SUM( optimize_line ) tLines,
       			SUM( optimize_steps ) tSteps,
       			user_id,
       		MODE 
       		FROM
       			adt_game_info 
       		WHERE
       			game_id = 4 
       		GROUP BY
       			user_id,
       		MODE 
       		) t ON t.user_id = us.user_id 
       		AND us.`mode` = t.`mode` 
       		SET us.optimize_line = t.tLines,
       		us.optimize_steps = t.tSteps 
       	WHERE
       		us.optimize_line != t.tLines 
       	OR us.optimize_steps != t.tSteps;
       END;
       ```

  3. 存储过程

     * 定时任务

     * 示例：

       ```mysql
       CREATE DEFINER=`leap_ad_hbc`@`%` PROCEDURE `optimalCodeView`(total int)
       BEGIN
       	DECLARE i int DEFAULT 1; -- 必须在前面
       
       	-- 临时表
       	CREATE TEMPORARY TABLE IF NOT EXISTS optimal_code_view(
       		level_id INT,
               optimize_line INT,
               optimize_steps INT,
               optimal_code_txt MEDIUMTEXT,
               id BIGINT
           ) DEFAULT CHARSET = utf8;
       	TRUNCATE TABLE optimal_code_view;
       
           -- 循环执行
           WHILE i <= total DO
       	INSERT INTO optimal_code_view
       	SELECT 
               gi.level_id,
               gi.optimize_line,
               gi.optimize_steps,
               gi.optimal_code_txt,
               gi.id
       	FROM adt_game_info gi
       	WHERE gi.game_id = 4 AND gi.is_clear = 1 AND optimize_line > 0 AND `mode` = 1 AND gi.level_id = i
       	ORDER BY
       		optimize_line DESC,
       		optimize_steps DESC
       	LIMIT 5;
  	
       	set i = i + 1;
       	END WHILE;
       
       -- 返回结果集
       SELECT * FROM optimal_code_view;
       END
       
       -- 调用
       CALL optimalCodeView(100);
       ```
       
     * Navicat 修改存储过程，保存时可能会提示 `PROCEDURE _Navicat_Temp_Stored_Proc already exists` ,应该是由于实时保存引起的，执行一下 `DROP PROCEDURE _Navicat_Temp_Stored_Proc` 就好了
     

# 2020-6-4

1. yaml配置文件：

   ```yaml
   person:
   	# 字符
       name: hbc
       # 数字
       age: 18
       # 布尔
       isRich: false
       # 日期 ISO 8601格式
       birth: 1994-08-17T13:00:00+08:00
       # map 注意key与value之间要有空格，可以换行
       maps: {k1: v1, k2: 12}
       # 数组
       lists:
         - lisi
         - zhaoliu
   ```

   * 使用 `~` 表示null

   * SpringBoot中使用 `@Value()` 只能给普通变量注入值，不能直接给静态变量赋值;若要给静态变量赋值，可以构造set()方法，并需要在类上加入 `@Component` 注解

   * yaml中的数组、map、list 只能用对象去接收，不能用 `@Value` 直接注入，需要通过配置类去接收（或者用下面的方式）

   * `@Value` 有两种：

     ```
     @Value("${game.chinaNationId}")
     private Integer chinaNationId;
     // #{SpEL表达式}
     @Value("#{1+2}")
     private Integer chinaNationId2;// China在国家表中的Id
     // #与$结合使用可以实现使用@Value接收map、list
     @Value("#{'${game.testSchoolId}'.split(',')}")
     private List<Integer> skipSchoolList;
     @Value("#{${map}}")
     private Map<String, String> map;
     ```

2. MySQL会认为NULL值比其他类型的数据小，在order by 正序排序的话，NULL值是在最前面的。通过在order by 排序字段前加上负号后使用DESC，可以达到正序排序并且NULL值在后面的效果
3. SpringBoot外部配置文件属性名称应该使用kebab-case（短横线式）而非camelCased（驼峰式），使用驼峰式有时会引发启动异常（有些又不会，具体原因不明）

# 2020-6-17

* 如果想要从Spring Boot jar包中的classpath中加载文件，需要使用文件流的形式来获取，例如：

  ```java
  ClassPathResource resource = new ClassPathResource("application.yml");
  resource.getInputStream();
  // 或者
  InputStream is = this.getClass().getResourceAsStream("application.yml"); 
  
  // 下面这样获取会出现异常-java.io.FileNotFoundException，因为这种方式下Spring是尝试通过文件系统路径获得文件，但它无法访问jar包中的路径-未解压
  resource.getFile();
  ```

* maven

  * 查看jar包依赖树：`mvn dependency:tree`
  * 分析jar包：`mvn dependency:analyze`



# 2020-6-22

* 类中定义 final 成员变量而不初始化会编译报错：`The blank final field userService may not have been initialized`。可以定义一个静态代码块/非静态代码块/构造函数
* 在 JVM 中，判断一个对象是否是某个类型时，如果该对象的实际类型与待比较的类型的类加载器不同，那么会返回 false。

* PUT 和 POST 都有更改指定URI的语义，但 PUT 被定义为 idempotent（幂等） 的方法，POST则不是。故：
  1. PUT请求：如果两个请求相同，后一个请求会把第一个请求覆盖掉——所以PUT用来改资源
  2. Post请求：后一个请求不会把第一个请求覆盖掉——所以Post用来增资源

* MySQL
  * 对于 InnoDB 来说，它的聚集索引和行记录是存储在一起的——聚集索引的叶子节点存储行记录（主键索引树）
  * 如果没有主动设置主键，就会选一个不包含NULL的第一个唯一索引列作为主键列，并把它用作一个聚集索引。如果没有这样的索引就会使用行号生成一个聚集索引，把它当做主键，这个行号6bytes，自增。可以用select _rowid from table来查询。
  * InnoDB普通索引的叶子节点存储主键值
  * 覆盖索引就是从辅助索引中就能直接得到查询结果，而不需要回表到聚簇索引中进行再次查询，所以可以减少搜索次数（不需要从辅助索引树回表到聚簇索引树），或者说减少IO操作（通过辅助索引树可以一次性从磁盘载入更多节点），从而提升性能
  * 按照MySQL最左匹配原则，建立 (a,b) 联合索引时，只按照 b 查询按理来说应该用不了索引。但实际上当查询字段是 a、b 中的字段（也可以包含id）时是能用索引的——此时 explain 语句会发现 possible_keys 为空，key 里却是 ab 联合索引，key_len 为 (a,b) 索引总长，type 为 index——符合最左匹配原则的话 type 应该是 ref。解释：此时查询字段符合覆盖索引，MySQL 还是会使用 (a,b) 索引树，不过是对整个索引树进行扫描（所以 type 为 index），找到符合条件的记录时，因为符合覆盖索引所以不用回表；而当不符合覆盖索引时，则无法避免回表——MySQL 此时直接选择进行全表扫描，而不使用索引树。
  * MySQL 5.6引入了索引下推优化，可以在索引遍历过程中，对索引中包含的字段先做判断，过滤掉不符合条件的记录，减少回表次数——like
  * 索引失效：
    * 在索引字段上使用不等操作——!=或<>（实测不影响唯一索引的使用，而且似乎在数据量不多的情况下普通索引也会生效——猜测应该和优化器有关）
    * 联合索引不符合最左前缀原则（加一句，且不符合覆盖索引）
    * 使用 OR 组合多个查询条件
    * 在索引字段上进行计算、函数、类型转换等操作——类型转换包含：查询条件字段参数类型与字段类型不符，比如 phone 字段为字符串时使用 where phone = 18720989734——隐式自动类型转换
    * 在索引字段上使用前导模糊查询——like '%9734'
    * null列是可以用到索引的，不管是单列索引还是联合索引，但 is null，is not null 是不走索引的
    * MySQL 优化器认会在全表扫描、使用各种索引等各个方案中选择他认为成本最低的那个，可能会放弃部分索引甚至选择全表扫描（比如索引）

# 2020-7-22

* 从设计理念上看确实有一定的相似性。主要流程都是 SQL 解析 -> SQL 改写 -> SQL 路由 -> SQL 执行 -> 结果归并。但架构设计上是不同的。Mycat 是基于 Proxy，它复写了 MySQL 协议，将 Mycat Server 伪装成一个 MySQL 数据库，而 Sharding-JDBC 是基于 JDBC 接口的扩展，是以 jar 包的形式提供轻量级服务的。