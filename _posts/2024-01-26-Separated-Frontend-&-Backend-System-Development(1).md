---
layout: post
title:  "Separated Frontend & Backend System Development(1)"
date:   2024-01-26 09:04:02 +0800
categories: jekyll update
tags: Tech-Share
---
<html>
<style>
    .tag-style {
        background-color: olivedrab;
        border-radius: 5px;
        padding-left: 4px;
        padding-right: 4px;
        display: inline-block;
        color: white;
        cursor: pointer;
    }
</style>
<body>
<a href="/tags/">
    <div class="tag-style"><i>Tech-Share</i></div>
</a>
<div markdown="block" style="margin-top: 10px">
    
# 前後端分離系統開發介紹(1)

## 0. 前言
前端為一個項目，後端為一個項目，分別部署后前端調用後端提供的API，總體來説可以看作兩個獨立的項目。通過這樣可以使得雙方專注與自己的部分，并且修改其中一端并不會影響到另一端的運行情況
- 本次介紹案例都在本地執行，前端使用 **Vue+Nginx**；後端使用 **Springboot+Tomcat**
    ```
    springboot: 3.2.2
    vue: 2.6.14
    vuex: 3.6.2
    nginx: 1.25.1
    tomcat: 10.1.18
    ```
- Springboot是SpringMVC的便捷版，可讓開發人員專注在開發細節上而非繁雜的配置；
另外Springboot嵌入在Tomcat、Jetty、Undertow上所以不需要額外部署它
- 體量比較: SpringMVC < Spring < Springboot
- Spring使用很多特殊的注解(annotation)作爲輔助，需要查閲[官方文檔](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#data)學習
- Vue本次使用 **Vue v2 + Vuex v3**; Vuex是Vue的狀態管理模式庫，通俗來説就是管理流通在Vue中的各種數據。另外Vue項目對各類庫的版本兼容很狹隘，Vue2/Vue3對應的依賴版本相差很多，可以使用 `npm audit fix` 檢查全部依賴的關聯性；原生的Vue2 [官方文檔](https://v2.vuejs.org/v2/api) 

## 1. 後端
### 1.1 範例項目架構
```
-src
    -main
        -java
            -controller
            -bl(business logic)
            -blImpl(business logic Implementation)
            -vo
            -po(dto)
            -dao
            -utils
        -resources
            -application.yml
            -application-prod.yml
            -application-dev.yml
    -test
```
### 1.2 配置文件
項目裏的配置文件默認為`src/main/resources/application.yaml`或替代為`application.properties`，兩者語法不同需注意，下面是一個簡單的例子
```
spring:
  application:
    name: SpringbootDemo
  datasource:
    url: jdbc:mysql://localhost:3306/demo_table
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    platform: mysql
    schema: classpath:schema.sql
    data: classpath:data.sql
    initialization: always
mybatis:
  mapper-locations: classpath:mapper/*Mapper.xml
server:
  port: 8090
```
- 還可以開發環境配置跟產品上線配置不同，在`application.yaml`中如下:
```
spring:
  profiles:
    active: prod  #switch prod/dev
```
然後生成`application-prod.yaml`和`application-dev.yaml`就可以切換

### 1.3 依賴文件
使用Maven管理項目依賴，下面是一個簡單的例子
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.5.5</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>caseLibrary</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>caseLibrary</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springdoc</groupId>
			<artifactId>springdoc-openapi-ui</artifactId>
			<version>1.1.45</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>8.0.19</version>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>1.3.2</version>
		</dependency>
		<dependency>
			<groupId>jdom</groupId>
			<artifactId>jdom</artifactId>
			<version>1.1</version>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<version>1.18.20</version>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```
### 1.4 SpringBoot架構
Springboot開發還是遵循SpringMVC，只不過把其中的View部分轉換成獨立的一個項目而并非包含在Spring項目中，因此後端項目只負責到Controller層。詳細關係為Contorller中注入Service，Service中注入DAO，因此可以把它視爲三層結構。大致關係如下:
```
Frontend -> Controller -> Service -> Repository -> Model(ORM)
```
#### 1.4.0 Application Main Entrance
- 繼承 `SpringBootServletInitializer` 是要讓這個Springboot程序能以WAR包的形式部署到外部Servlet容器中
```
@SpringBootApplication
public class CaseLibraryApplication extends SpringBootServletInitializer{

	public static void main(String[] args) {
		SpringApplication.run(CaseLibraryApplication.class, args);
	}

    @Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder builder){
		return builder.sources(BackendDemoApplication.class);
	}
}
```
#### 1.4.1 Controller層
```
@RestController
@RequestMapping("/graph")
public class GraphController {
    @Autowired
    private LawService lawService;

    @Autowired
    private LinkService linkService;

    @PostMapping("/getGraphData")
    public ResponseVO getGraphData(@RequestBody SearchTargetVO searchTargetVO){
        return linkService.getLinks(searchTargetVO.getSearchTarget());
    }
    @PostMapping("/addLink")
    public ResponseVO addLink(@RequestBody LinkVO linkVO){
        return linkService.addLink(linkVO);
    }

    @PostMapping("/deleteLinks")
    public ResponseVO deleteLinks(@RequestBody SearchTargetVO searchTargetVO) {
        return linkService.deleteLink(searchTargetVO.getSearchTarget());
    }
}
```
- `@RestController`表示這個類是Controller層，并且創造的是REST API
- `@RequestMapping`表示HTTP Request的路徑
- `@Autowired`將Service層的類注入到這層也就是Controller層
- `@PostMapping("/addLink")`聲明HTTP Request的類別和路徑
- `@RequestBody`可以使Controller接收到HTTP Client傳送的JSON數據，並將數據映射為預先設計好的Java類，例如在上述例子中的`addLink()`中的`LinkVO`類便是設計好的Java類，在設計層面上來説這個`LinkVO`屬於VO(Value Object)，屬於Controller響應前端請求后返回的數據格式，下面是`LinkVO`的例子:
```
public class LinkVO {
    private String source;
    private String target;
    private String rela;
    public String getSource() {
        return source;
    }
    public void setSource(String source) {
        this.source = source;
    }
    public String getTarget() {
        return target;
    }
    public void setTarget(String target) {
        this.target = target;
    }
    public String getRela() {
        return rela;
    }
    public void setRela(String rela) {
        this.rela = rela;
    }
}
```

- 有個好用的庫可以簡化手動生成Get/Set方法`lombok`: 使用時在maven中添加依賴

```
<dependency>
	<groupId>org.projectlombok</groupId>
	<artifactId>lombok</artifactId>
	<version>1.18.20</version>
</dependency>
```

簡化項目中的DAO/PO/VO.etc，`LinkVO`便可簡化為:

```
@Data
public class LinkVO {
    private String source;
    private String target;
    private String rela;
}
```

- 值得一提的是，`ResponseVO`是默認返回給前端的類，使得前端處理後端返回的數據時不會因爲不同API而有不同的預處理方法，下面是`ResponseVO`的設計可供參考:

```
public class ResponseVO {
    // 表示這次請求是否成功
    private Boolean success;
    // 顯示的訊息不管成功或者失敗
    private String message;
    // 返回的數據
    private Object content;

    public static ResponseVO buildSuccess(){
        ResponseVO response=new ResponseVO();
        response.setSuccess(true);
        return response;
    }
    public static ResponseVO buildSuccess(Object content){
        ResponseVO response=new ResponseVO();
        response.setContent(content);
        response.setSuccess(true);
        return response;
    }
    public static ResponseVO buildFailure(String message){
        ResponseVO response=new ResponseVO();
        response.setSuccess(false);
        response.setMessage(message);
        logger.error(message);
        return response;
    }
}
```

#### 1.4.2 Service層
也就是Business Logic層，是主要處理數據邏輯的模塊。所有的處理都在這層實現，並不會延申到Controller層(也就是說Controller層只是負責處理Http Request)。設計思路有很多，但是大體可以分爲兩部分: Service Interface和Service Implementation。Interface主要聲明要完成的方法，白話說就是TODO List；Implementation是實現Interface的模塊，下面提供一個例子:
- LinkService(Interface)

```
public interface LinkService {
    ResponseVO getLinks(String lawTitle);
    ResponseVO enterFormData();
    ResponseVO getRuleBySourceTitle(String searchTarget);
    ResponseVO addLink(LinkVO linkVO);
    ResponseVO deleteLink(String source);
}
```

- LinkServiceImpl(Implementaiton)

```
@Service
public class LinkServiceImpl implements LinkService {
    @Autowired
    private LinkRepository linkRepository;

    @Override
    public ResponseVO getLinks(String lawTitle) {
        return ResponseVO.buildSuccess(linkRepository.findBySource(lawTitle));
    }
    ...
}
```

- `@Service`表示這個類屬於Service層
- `LinkRepository`類為DAO層，所以在Service層中注入並使用
#### 1.4.3 Model (PO)
在介紹Model和Repository之前，要先介紹Spring Data JPA與其餘ORM的關係。本次介紹使用Spring Data JPA并非傳統JPA Provider(如hibernate)，論層級來説它應該處於JPA之上JPA Provider之下，它默認使用hibernate來實現細節(可以在pom依賴下看到spring-data-jpa有子依賴hibernate-core)。它是獨立于Spring framework上的技術，并且不需要開發者實現細節，只需要根據規則套用注解即可。

```
@Entity
@Table(name = "link")
@Data
public class Link {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @NotNull
    @Column(name = "source",columnDefinition = "varchar(255)",nullable = false)
    private String source;

    @Column(name = "target",columnDefinition = "varchar(255)",nullable = false)
    private String target;

    @NotNull
    @Column(name = "relationship",columnDefinition = "varchar(255)",nullable = false)
    private String relation;
}
```

- `@Entity` 是Spring Data JPA中的注解，通常來説添加這個注解就默認生成一個表(這個注解下的類)為表名
- `@Table(name = "link")`如果要指定特定表名則可以使用此注解
- `@Id` `@GeneratedValue` 是每個表必要的Id列，`@GeneratedValue`則制定Id列生成規則
- `@Column`聲明列
#### 1.4.4 Repository (DAO)
Spring Data JPA需要我們設計Repository的Interface，在運行過程中它會自動生成這個Repository的實現，避免我們手動生成冗長的代碼如Connection等。因此我們只需專注在SQL語句即可。另外，它兼容原生的SQL語句

```
public interface LinkRepository extends JpaRepository<Link, Integer> {

    List<Link> findBySource(String source);

    @Transactional
    @Query(value = "select * from link where source like concat('%',:source,'%')", nativeQuery = true)
    List<Link> searchBySource(String source);

    @Transactional
    @Query(value = "select count(*) from link where " +
            "source like concat('%',:source,'%') and " +
            "target like concat('%',:target,'%') and " +
            "relationship like concat('%',:relation,'%')",
            nativeQuery = true)
    Integer findLinkBySTR(@Param("source") String source, @Param("target") String target, @Param("relation") String relation);


    @Transactional
    void deleteLinksBySource(String source);
}
```

- `LinkRepository`繼承`JpaRepository`，它提供很多默認方法如基礎的CRUD: 即使在`LinkRepository`中沒有聲明find方法，在繼承`JpaRepository`后仍可依照規則調用。簡單來説，Spring Data JPA是可以讓使用者使用Interface和特定的Function Design來完成ORM的使用。例如在上述例子中，`JpaRepository<Link, Integer>`使用`Link`表(1-3-3), ID為Integer類型。`Link`類中有id, source, target, relation四列；則LinkRepository默認就擁有下列方法:

```
findById(Integer id)
...
findAll()

deleteById(Integer id)
...
deleteAll()

save(Link link)
...
```

- 可以節省很多時間并且非常直觀，同時也接受Override改變其方法實現
- `@Transactional`表示將此方法視爲一個事務，當此方法失敗則不會影響到其他的事務，若是此方法中其中一個操作失敗則全部一起失敗並rollback數據(可以定義rollback rules)
- `@Query`表示詳細的SQL語句，有很多屬性和規則，可在[官方文檔](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)查詢
- `@Param`可以指定在SQL中使用`:name`來代替`?1`

</div>
</body>
</html>
