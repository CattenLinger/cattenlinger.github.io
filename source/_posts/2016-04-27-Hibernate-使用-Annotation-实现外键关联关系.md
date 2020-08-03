---
title: Hibernate 使用 Annotation 实现外键关联关系
date: 2016-04-27 06:07:58
categories:
- '笔记'
- '技术'
tags:
- 'Java'
- 'Hibernate'
---
因为在使用Hibernate的Annotation时遇到坑，坑了一晚上时间，所以写一篇文章记一下经验

如果并没有对Hibernate入门，还请在课室或者在技研中心，接入学校的教学网络，进入服务器smb://10.15.231.233/Video （在技研的网络里叫OPENSUSE-SERVER）。在 数据库->ORM->Hibernate 目录内查看视频教程了解

我的项目使用的是Hibernate 4.3.11.Final，是一个Maven工程。所用到的辅助工具有fastjson，用来把类直接输出成json查看，还有junit4，测试用。

依赖列表(pom.xml)节选

	<dependencies>
	        <dependency>
	            <groupId>org.hibernate</groupId>
	            <artifactId>hibernate-core</artifactId>
	            <version>4.3.11.Final</version>
	        </dependency>
	        <dependency>
	            <groupId>org.slf4j</groupId>
	            <artifactId>slf4j-simple</artifactId>
	            <version>1.7.18</version>
	        </dependency>
	        <dependency>
	            <groupId>mysql</groupId>
	            <artifactId>mysql-connector-java</artifactId>
	            <version>5.1.38</version>
	        </dependency>
	        <dependency>
	            <groupId>junit</groupId>
	            <artifactId>junit</artifactId>
	            <version>4.8.2</version>
	            <scope>test</scope>
	        </dependency>
	        <dependency>
	            <groupId>com.alibaba</groupId>
	            <artifactId>fastjson</artifactId>
	            <version>1.2.8</version>
	        </dependency>
	</dependencies>

我的工程是一个有关于评分的系统。在里面有两个类，一个是评分模板MarkingTemplate，一个是评分项目MarkingItem。

MarkingTemplate在数据库中的字段如下

|字段名               |数据类型     | 备注                                     |
|:------------------:|:----------:|:---------------------------------------:|
|id                  |INT         |数据编号（自动增长）                        |
|title               |nvarchar(50)|模板标题                                  |
|commit              |nvarchar(50)|模板备注                                  |

实体类字段如下：


	private int id;
	private String title;
	private String commit;
	private int value;
	private MarkingTemplate markingTemplate;


MarkingItem在数据库里的字段如下


|字段名               |数据类型     | 备注                                     |
|:------------------:|:----------:|:---------------------------------------:|
|id                  |INT         |数据编号（自增长）                          |
|title               |nvarchar(50)|评分项目标题                               |
|commit              |nvarchar(50)|评分项目备注                               |
|value               |INT         |项目分值（只有整数）                        |
|belong\_to\_template|INT         |所属模板的编号（外键链接至contest\_templates）|

实体类字段如下：


	private int id;
	private String title;
	private String commit;
	private List<MarkingItem> markingItems;


很明显，我数据库里只有MarkingItem到MarkingTemplate的关联，没有明显的反过来的关联。看视频教程的话，都是用的实体类配置文件实现这些关系，但是我想尝试使用Annotation（注解）来实现这个关系，因为本身项目简单，而且注解很简洁很方便。

MarkingTemplate类源码和注解


	@Entity
	@Table(name = "marking_templates")
	public class MarkingTemplate {
	    private int id;
	    private String title;
	    private String commit;
	    private List<MarkingItem> markingItems;
	
	    @Id
	    @GeneratedValue
	    public int getId() {
	        return id;
	    }
	
	    public void setId(int id) {
	        this.id = id;
	    }
	
	    public String getTitle() {
	        return title;
	    }
	
	    public void setTitle(String title) {
	        this.title = title;
	    }
	
	    public String getCommit() {
	        return commit;
	    }
	
	    public void setCommit(String commit) {
	        this.commit = commit;
	    }
	
	    @OneToMany(mappedBy = "markingTemplate")
	    public List<MarkingItem> getMarkingItems() {
	        return markingItems;
	    }
	
	    public void setMarkingItems(List<MarkingItem> markingItems) {
	        this.markingItems = markingItems;
	    }
	}


MarkingItem源码和注解

	
	@Entity
	@Table(name = "marking_items")
	public class MarkingItem {
	    private int id;
	    private String title;
	    private String commit;
	    private int value;
	    private MarkingTemplate markingTemplate;
	
	    @Id
	    @GeneratedValue
	    public int getId() {
	        return id;
	    }
	
	    public void setId(int id) {
	        this.id = id;
	    }
	
	    public String getTitle() {
	        return title;
	    }
	
	    public void setTitle(String title) {
	        this.title = title;
	    }
	
	    public String getCommit() {
	        return commit;
	    }
	
	    public void setCommit(String commit) {
	        this.commit = commit;
	    }
	
	    public int getValue() {
	        return value;
	    }
	
	    public void setValue(int value) {
	        this.value = value;
	    }
	
	    @ManyToOne(targetEntity = MarkingTemplate.class)
	    @JoinColumn(name = "belong_to_template")
	    public MarkingTemplate getMarkingTemplate() {
	        return markingTemplate;
	    }
	
	    public void setMarkingTemplate(MarkingTemplate markingTemplate) {
	        this.markingTemplate = markingTemplate;
	    }
	}


如果要实现双向的关联，那么需要在两边的类关联的字段所对应的属性声明上添加注解。

先解释一下MarkingItem的`getMarkingTemplate()`


    @ManyToOne(targetEntity = MarkingTemplate.class)
    @JoinColumn(name = "belong_to_template")
    public MarkingTemplate getMarkingTemplate() {
        return markingTemplate;
    }


从数据库里提取belong_to_template字段的数值然后再利用其查询即可得出MarkingTemplate的数值。`@ManyToOne`表示了一个多对一的关系。**模板内项目**对**模板**是多对一关系。**targetEntity**标明这个字段是关联到哪个类，通过这个，Hibernate可以很方便地把这个查出来的号码把MarkingItems所属的MarkingTemplate实体化出来。`@JoinColumn`表明插入列的字段名称，这个可以省去，我添加这个的原因是因为类中的属性名和数据库内的字段名不相符。

再解释一下MarkingTemplate的`getMarkingItems()`


    @OneToMany(mappedBy = "markingTemplate")
    public List<MarkingItem> getMarkingItems() {
        return markingItems;
    }


`@OneToMany`注解声明了一个一对多关系。从数据库字段表可以看出，**模板**对**模板内项目**是一对多关系。一个模板拥有多个模板内项目，这些项目在类里被存放在一个List里。而在数据库里，是每个项目都有一个字段标明它们属于哪个模板。**mappedBy**参数表明这个被在Many一方的类的那个属性给映射了。

当查出实体数据之后，Hibernate默认是对关联属性的数据进行延迟加载，就是等到调用get方法的时候再去生成SQL语句查询这个类。

测试类里我利用fastjson直接把数据可视化了。

获取MarkingTemplate


    @Test
    public void TestGetMarkingTemplate(){
    	System.out.println(JSON.toJSONString(session.get(MarkingTemplate.class,1)));
    }

输出结果
	
	{
	    "commit": "通用模板", 
	    "id": 1, 
	    "markingItems": [
	        {
	            "commit": "结合竞赛主题，作品选题具有实用性、合理性、创新性", 
	            "id": 6, 
	            "markingTemplate": {
	                "$ref": "$"
	            }, 
	            "title": "选题", 
	            "value": 10
	        }, 
	        {
	            "commit": "界面友好，完成速度快，操作简洁明了", 
	            "id": 7, 
	            "markingTemplate": {
	                "$ref": "$"
	            }, 
	            "title": "界面", 
	            "value": 20
	        }, 
	        {
	            "commit": "采用的软件开发方法与技术是否先进及与项目相宜", 
	            "id": 8, 
	            "markingTemplate": {
	                "$ref": "$"
	            }, 
	            "title": "技术", 
	            "value": 10
	        }, 
	        {
	            "commit": "功能强大，具有实用性", 
	            "id": 9, 
	            "markingTemplate": {
	                "$ref": "$"
	            }, 
	            "title": "功能", 
	            "value": 40
	        }, 
	        {
	            "commit": "作品的市场潜力与价值", 
	            "id": 10, 
	            "markingTemplate": {
	                "$ref": "$"
	            }, 
	            "title": "市场", 
	            "value": 10
	        }, 
	        {
	            "commit": "对问题的实际理解程度", 
	            "id": 11, 
	            "markingTemplate": {
	                "$ref": "$"
	            }, 
	            "title": "答辩", 
	            "value": 10
	        }
	    ], 
	    "title": "模板1"
	}

因为fastjson在生成json string的时候调用了里面的get方法，所以这里没体现hibernate的延迟加载。

获取MarkingItem

    @Test
    public void TestGetMarkingItems(){
        List markingItems = session.createQuery("from MarkingItem").setFirstResult(0).setMaxResults(9).list();
        for(Object o: markingItems){
            System.out.println(JSON.toJSONString(o));
        }
    }

输出结果


	{"commit":"结合竞赛主题，作品选题具有实用性、合理性、创新性","id":6,"markingTemplate":{"commit":"通用模板","id":1,"markingItems":[{"$ref":"$"},{"commit":"界面友好，完成速度快，操作简洁明了","id":7,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"界面","value":20},{"commit":"采用的软件开发方法与技术是否先进及与项目相宜","id":8,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"技术","value":10},{"commit":"功能强大，具有实用性","id":9,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"功能","value":40},{"commit":"作品的市场潜力与价值","id":10,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"市场","value":10},{"commit":"对问题的实际理解程度","id":11,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"答辩","value":10}],"title":"模板1"},"title":"选题","value":10}
	{"commit":"界面友好，完成速度快，操作简洁明了","id":7,"markingTemplate":{"commit":"通用模板","id":1,"markingItems":[{"commit":"结合竞赛主题，作品选题具有实用性、合理性、创新性","id":6,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"选题","value":10},{"$ref":"$"},{"commit":"采用的软件开发方法与技术是否先进及与项目相宜","id":8,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"技术","value":10},{"commit":"功能强大，具有实用性","id":9,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"功能","value":40},{"commit":"作品的市场潜力与价值","id":10,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"市场","value":10},{"commit":"对问题的实际理解程度","id":11,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"答辩","value":10}],"title":"模板1"},"title":"界面","value":20}
	{"commit":"采用的软件开发方法与技术是否先进及与项目相宜","id":8,"markingTemplate":{"commit":"通用模板","id":1,"markingItems":[{"commit":"结合竞赛主题，作品选题具有实用性、合理性、创新性","id":6,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"选题","value":10},{"commit":"界面友好，完成速度快，操作简洁明了","id":7,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"界面","value":20},{"$ref":"$"},{"commit":"功能强大，具有实用性","id":9,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"功能","value":40},{"commit":"作品的市场潜力与价值","id":10,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"市场","value":10},{"commit":"对问题的实际理解程度","id":11,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"答辩","value":10}],"title":"模板1"},"title":"技术","value":10}
	{"commit":"功能强大，具有实用性","id":9,"markingTemplate":{"commit":"通用模板","id":1,"markingItems":[{"commit":"结合竞赛主题，作品选题具有实用性、合理性、创新性","id":6,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"选题","value":10},{"commit":"界面友好，完成速度快，操作简洁明了","id":7,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"界面","value":20},{"commit":"采用的软件开发方法与技术是否先进及与项目相宜","id":8,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"技术","value":10},{"$ref":"$"},{"commit":"作品的市场潜力与价值","id":10,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"市场","value":10},{"commit":"对问题的实际理解程度","id":11,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"答辩","value":10}],"title":"模板1"},"title":"功能","value":40}
	{"commit":"作品的市场潜力与价值","id":10,"markingTemplate":{"commit":"通用模板","id":1,"markingItems":[{"commit":"结合竞赛主题，作品选题具有实用性、合理性、创新性","id":6,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"选题","value":10},{"commit":"界面友好，完成速度快，操作简洁明了","id":7,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"界面","value":20},{"commit":"采用的软件开发方法与技术是否先进及与项目相宜","id":8,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"技术","value":10},{"commit":"功能强大，具有实用性","id":9,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"功能","value":40},{"$ref":"$"},{"commit":"对问题的实际理解程度","id":11,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"答辩","value":10}],"title":"模板1"},"title":"市场","value":10}
	{"commit":"对问题的实际理解程度","id":11,"markingTemplate":{"commit":"通用模板","id":1,"markingItems":[{"commit":"结合竞赛主题，作品选题具有实用性、合理性、创新性","id":6,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"选题","value":10},{"commit":"界面友好，完成速度快，操作简洁明了","id":7,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"界面","value":20},{"commit":"采用的软件开发方法与技术是否先进及与项目相宜","id":8,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"技术","value":10},{"commit":"功能强大，具有实用性","id":9,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"功能","value":40},{"commit":"作品的市场潜力与价值","id":10,"markingTemplate":{"$ref":"$.markingTemplate"},"title":"市场","value":10},{"$ref":"$"}],"title":"模板1"},"title":"答辩","value":10}

嗯。。。这里把模板也加载出来了。