---
title: MyBatis 对象映射 （一）
date: 2016-05-26 19:55:57
tags: [mybatis, association, collection, 对象关联, 集合嵌套, 嵌套查询]
---

Note: 尽量少用关联嵌套查询，因为关联嵌套查询会导致 N+1 查询问题，也就是说

* 如果你执行了一条SQL语句来获取结果列表(1)

* 对于返回的每一个查询语句，都会执行查询细节的SQL语句(N)

* 至于为什么还要使用，是因为写不出很优美的各种JOIN语句，又或是写出的JOIN语句出现了一个奇怪的问题 

<!-- more -->

## 对象关联，集合嵌套

### 建立对应的表

``` sql
DROP TABLE IF EXISTS `t_items`;
CREATE TABLE `t_items` (
  `id` int(8) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=100 DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS `t_item_status`;
CREATE TABLE `t_item_status` (
  `itemId` int(8) NOT NULL,
  `status` VARCHAR(100),
  `time` TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS `t_sub_items`;
CREATE TABLE `t_sub_items` (
  `itemId` int(8) NOT NULL,
  `name` varchar(100)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

### 建立对应的po对象

* Items.java

``` java
public class Items {

    private Integer id;

    private String name;

    private ItemStatus status;

    private List<SubItem> subItems;

    public ItemStatus getStatus() {
        return status;
    }

    public void setStatus(ItemStatus status) {
        this.status = status;
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public List<SubItem> getSubItems() {
        return subItems;
    }

    public void setSubItems(List<SubItem> subItems) {
        this.subItems = subItems;
    }
```

* ItemStatus.java

``` java
public class ItemStatus {

    private String status;

    private Date time;

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

    public Date getTime() {
        return time;
    }

    public void setTime(Date time) {
        this.time = time;
    }
}
```

* SubItem.java

``` java
public class SubItem {

    private Integer itemId;

    private String name;

    public Integer getItemId() {
        return itemId;
    }

    public void setItemId(Integer itemId) {
        this.itemId = itemId;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

### mapper 文件

``` xml
<resultMap id="itemMap" type="org.hpeng526.ssm.po.Items">
        <id property="id" column="id" />
        <result property="name" column="name" />
        <association property="status" column="id" select="selectStatus"/>
        <collection property="subItems" javaType="ArrayList" ofType="org.hpeng526.ssm.po.SubItem" column="id" select="selectSubItems" />
    </resultMap>

    <resultMap id="itemStatusMap" type="org.hpeng526.ssm.po.ItemStatus">
        <result property="status" column="status" />
        <result property="time" column="time" />
    </resultMap>

    <select id="selectSubItems" resultType="org.hpeng526.ssm.po.SubItem" >
        SELECT * FROM t_sub_items WHERE itemId = #{id}
    </select>

    <select id="selectStatus" resultMap="itemStatusMap" >
        SELECT * FROM t_item_status WHERE itemId = #{id} ORDER BY time desc limit 1
    </select>

    <select id="findAll" resultMap="itemMap">
        SELECT * FROM t_items
    </select>
```

### 执行结果

``` json
[{"id":101,"name":"item1","status":{"status":"new status","time":1464266199000},"subItems":[{"itemId":101,"name":"笔"},{"itemId":101,"name":"book"}]}]

```

### 特殊属性说明
| 属性          | 描述          |
| ------------ |:------------:|
| column       | 在 association, collection 标签里面的column属性指的是传递给下个查询的数据库列名(也可以是别名)|

* [源码](https://github.com/hpeng526/ssm/tree/mybatis)

* [官方文档](http://www.mybatis.org/mybatis-3/sqlmap-xml.html)

