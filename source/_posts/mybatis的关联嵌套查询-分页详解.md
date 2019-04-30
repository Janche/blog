---
title: mybatis的关联嵌套查询--分页详解
comments: true
date: 2019-04-26 15:05:31
tags:
    - mybatis
    - 嵌套分页查询
---

### 问题描述
**1. mybatis嵌套查询后，分页混乱：mybatis通过查询结果之后折叠结果集把数据放在了集合里,这就导致总条数的混乱.而第一种的方式是分两次查询，分页只针对第一次查询,就不会有分页的问题,所以解决方案就是把你的collection写成第一种的方式**
**2. 折叠结果集映射不上数据**

<!-- more -->
### 1. 数据库
```
-- 区域表：
CREATE TABLE `area`  (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `parent_id` bigint(11),
  `area_name` varchar(100),
  `area_code` varchar(32),
   PRIMARY KEY (`id`) USING BTREE
) ;

-- 设备表
CREATE TABLE `device`  (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `area_id` bigint(11),
  `device_name` varchar(64),
  `device_code` varchar(36),
  `device_type` varchar(36),
   PRIMARY KEY (`id`) USING BTREE
);
```
>**关联说明：区域表和设备表是一对多的关系**

### 2. 实体类

```
// 区域类的DTO
package com.device.dto;
import lombok.Data;
import java.util.List;
@Data  // 此注解提供了实体类get和set方法
public class AreaOutpDTO {
    private Long id;
    private Long parentId;
    private String areaName;
    private String areaCode;
    private List<DeviceTypeNumDTO> devices; // 设备数量DTO集合
}

// 设备数量的DTO，注意这里不是直接的设备类，而是对设备类型和数量的一个统计，若是设备类的话，则也没有那么复杂了
package com.device.dto;
import lombok.Data;
@Data
public class DeviceTypeNumDTO {
    private String deviceType; // 设备类型
    private Long deviceNum;	// 设备数量
}
```
### 3. Mapper 文件的编写

```
    <resultMap id="AreaOutpDTOMap" type="com.device.dto.AreaOutpDTO">
        <id column="id" jdbcType="BIGINT" property="id" />
         <result column="parent_id" jdbcType="VARCHAR" property="parentId" />
        <result column="area_name" jdbcType="VARCHAR" property="areaName" />
        <result column="area_code" jdbcType="VARCHAR" property="areaCode" />
        <!-- 设备 类型数量集合 -->
        <collection property="devices" ofType="com.device.dto.DeviceTypeNumDTO"
                    select="selectDeviceTypeNumDTO"
                    column="id">
                    <!-- 下面这两行可以不用写，但写了也不会通过它映射，只能通过下面的DeviceTypeNumDTOMap进行映射 -->
                     <!-- <result column="device_type" jdbcType="VARCHAR" property="deviceType" />
        	     		  	<result column="device_num" jdbcType="BIGINT" property="deviceNum" />-->
        </collection>
    </resultMap>
    
	<!-- 此Map必须写，否则数据无法映射到devices中 -->
    <resultMap id="DeviceTypeNumDTOMap" type="com.device.dto.DeviceTypeNumDTO">
        <result column="device_type" jdbcType="VARCHAR" property="deviceType" />
        <result column="device_num" jdbcType="BIGINT" property="deviceNum" />
    </resultMap>
    
     <select id="listSubArea" resultMap="AreaOutpDTOMap">
        select
        a.id,
        a.parent_id,
        a.area_name,
        a.area_code,
        from area a
        where a.parent_id = #{areaId}
    </select>
    <select id="selectDeviceTypeNumDTO" resultMap="DeviceTypeNumDTOMap">
      select
        d.device_type, count(*) device_num
      from device d
      group by d.device_type, d.area_id
      having d.area_id = #{id}
    </select>
```
>**注意事项：(大坑) 开始没有写DeviceTypeNumDTOMap，而是将映射的字段写在collection中，发现无论如何都不能在折叠结果集中取到数据，而SQL却又是被调用了的。**

**上述Mapper的处理过程:**
在调用mapper接口中的`listArea`方法，首先调用SQL`select a.id,a.parent_id, a.area_name,a.area_code,from area a where a.parent_id = #{areaId}`，从数据库中拿到返回数据之后，开始做返回值映射，此时的返回值映射对象是一个`resultMap`（id:AreaOutpDTOMap），该resultMap实际上是一个`AreaOutpDTO`的实例，因此在开始做映射的时候，id和  parent_id,area_name,area_code,因为属性名和数据库返回值一致完成映射，但是到了`devices`属性的时候，发现他是一个`collection`集合对象，里面存放的是DeviceTypeNumDTO实例，那怎么获取里面的数据？看collection标签的属性，他需要关联一个查询操作`select="selectDeviceTypeNumDTO"`获取其数据，即通过` select d.device_type, count(*) device_num from device d group by d.device_type, d.area_id having d.area_id = #{id}`获取collection中的数据，`select="selectPer"`操作需要传入数据，传入的数据`column="id"`即是第一次查询中返回的`id`，**最后一步，就是将第二次查询出来的数据映射到collection中,切记一定要写一个`ResultMap`来接受，不能直接在collection中添加字段来映射。**

### 4. 分页的处理
**经过上面的Mapper映射后，调用PageHelper插件已经是对折叠后的结果集进行分页了,成功解决分页总数混乱的问题**
```
 @Override
    public List<AreaOutpDTO> listSubArea(Integer page, Integer size, String orderBy, Long areaId) {
        PageHelper.startPage(page, size, orderBy);
        return areaMapper.listSubArea(areaId, categoryId);
    }
```

### 5. 如果不分页，则采用mybatis的第二种映射方式
**仅改动Mapper文件，其余相同**

```
 <resultMap id="AreaOutpDTOMap" type="com.device.dto.AreaOutpDTO">
        <id column="id" jdbcType="BIGINT" property="id" />  
        <result column="parent_id" jdbcType="BIGINT" property="parentId" />
        <result column="area_name" jdbcType="VARCHAR" property="areaName" />
        <result column="area_code" jdbcType="VARCHAR" property="areaCode" />
		<collection property="devices" ofType="com.device.dto.DeviceTypeNumDTO">
	        <result column="device_type" jdbcType="VARCHAR" property="deviceType" />
       		<result column="device_num" jdbcType="BIGINT" property="deviceNum" />
        </collection>
    </resultMap>
    
   <select id="listArea" resultMap="AreaOutpDTOMap">
      select
       a.id,
       a.parent_id,
       a.area_name,
       a.area_code,
       c.device_type,
       c.device_num
     from area a
     left join (
              select
                     d.area_id, d.device_type, count(*) device_num
              from device d
              group by d.device_type, d.area_id) c
             on a.id = c.area_id
     where a.parent_id = #{areaId}
    </select>

```

转载请注明出处：https://blog.csdn.net/qq_34997906/article/details/84498115