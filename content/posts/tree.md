---
title: java中通用树状菜单工具类
subtitle:
date: 2023-03-10T15:32:40+08:00
draft: true
author:
  name: yxx
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - tree
categories:
  - java
hiddenFromHomePage: false
hiddenFromSearch: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content/#front-matter
---
开发中经常遇到树状结构，每次都要写比较麻烦。这里写了一个工具类简化其中的步骤
<!--more-->
话不多说，上代码
****
</br>
菜单类

```java
import com.baomidou.mybatisplus.annotation.*;
import io.swagger.annotations.ApiModelProperty;
import lombok.Data;

import java.io.Serializable;
import java.util.List;

/**
 * @author yxx
 * @since 2023-03-07 14:46
 */
@Data
@TableName("ee_menu")
public class Menu implements Serializable {
    /**
     * 主键
     */
    @ApiModelProperty(value = "业务主键")
    @TableId(type = IdType.AUTO)
    private Integer id;

    /**
     * 父id
     */
    @ApiModelProperty(value = "父id")
    private Integer parentId;

    /**
     * 菜单标识
     */
    @ApiModelProperty(value = "菜单标识")
    private String menuCode;

    /**
     * 菜单名称
     */
    @ApiModelProperty(value = "菜单名称")
    private String menuName;

    /**
     * 菜单类型: 1- 管理系统; 2- 子系统
     */
    @ApiModelProperty(value = "菜单类型: 1- 管理系统; 2- 子系统")
    private Integer menuType;

    /**
     * 是否删除: 0- 否; 1- 是
     */
    @ApiModelProperty(value = "是否删除: 0- 否; 1- 是")
    @TableLogic
    private Integer isDelete;

    /**
     * 子菜单集合
     */
    @TableField(exist = false)
    private List<Menu> children;
}
```
****
工具类
```java
import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.text.CharSequenceUtil;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.experimental.Accessors;

import java.beans.PropertyDescriptor;
import java.lang.reflect.Method;
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * @author yxx
 * @apiNote 返回层级菜单
 * @date 2023-03-10 14:51
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
@Accessors(chain = true)
public class TreeMenuUtil<T> {
    /**
     * 顶层节点的值
     */
    private String rootValue;
    /**
     * 对应父节点的子属性
     */
    private String childKey;
    /**
     * 父节点的属性值
     */
    private String rootKey;
    /**
     * 子菜单放置的字段
     */
    private String childProperty;
    /**
     * 要过滤的集合
     */
    private List<T> list;

    /**
     * @return List
     * @apiNote 返回树状菜单
     */
    public List<T> rootMenu() {
        //判断rootValue是否为空，如果为空的话给rootValue赋值
        ifNullRootValueSetValue();
        //筛选出来顶级目录
        List<T> rootList = list.stream()
                .filter(item -> CharSequenceUtil.equals(rootValue, getValueByProperty(item, childKey)))
                .collect(Collectors.toList());
        //从数据集合中剔除顶层目录，减少后续遍历次数，加快速度
        list.removeAll(rootList);
        //如果list不为空的话则遍历赋值子菜单
        if (CollUtil.isNotEmpty(rootList)) {
            for (T t : rootList) {
                setChildren(t, list);
            }
            return rootList;
        } else {
            return list;
        }
    }

    /**
     * 判断rootKey是空值的话则赋值
     */
    private void ifNullRootValueSetValue() {
        if (CharSequenceUtil.isBlank(rootValue)) {
            Set<String> rootValueSet = list.stream()
                    .map(m -> getValueByProperty(m, rootKey))
                    .collect(Collectors.toSet());
            Set<String> childValueSet = list.stream()
                    .map(m -> getValueByProperty(m, childKey))
                    .collect(Collectors.toSet());
            //将childKey（parentId）中的数据全部赋值过来
            Set<String> resultList = new HashSet<>(childValueSet);
            //算出childValueSet和rootValueSet的交集
            childValueSet.retainAll(rootValueSet);
            // 算出childValueSet 与交集不同的部分
            resultList.removeAll(childValueSet);
            //如果有差集的话，把第一个数据返回到rootValue中
            if (CollUtil.isNotEmpty(resultList)) {
                rootValue = String.valueOf(resultList.toArray()[0]);
            }
        }
    }

    /**
     * 给父级菜单赋值子菜单
     *
     * @param t    父菜单对象
     * @param list 所有数据
     */
    private void setChildren(T t, List<T> list) {
        String childPropertyTypeName = Objects.requireNonNull(getPropertyDescriptor(t, childProperty)).getPropertyType().getName();
        Stream<T> childStream = list.stream()
                .filter(item -> isChild(t, item));
        Collection<T> childList;
        if (childPropertyTypeName.contains("Set")) {
            childList = childStream.collect(Collectors.toSet());
            setValueByProperty(t, childList);
        } else {
            childList = childStream.collect(Collectors.toList());
            setValueByProperty(t, childList);
        }
        list.removeAll(childList);
        if (CollUtil.isNotEmpty(childList)) {
            for (T item : childList) {
                setChildren(item, list);
            }
        }
    }

    /**
     * 判断当前对象是不是父级对象的子级
     *
     * @param t    父对象
     * @param item 子对象
     * @return true:是; false:不是
     */
    private boolean isChild(T t, T item) {
        rootValue = getValueByProperty(t, rootKey);
        String childParentValue = getValueByProperty(item, childKey);
        return rootValue.equals(childParentValue);
    }

    /**
     * 通过属性获取属性的值
     *
     * @param t   对象
     * @param key 属性
     * @return 属性的值
     */
    private String getValueByProperty(T t, String key) {
        PropertyDescriptor keyProperty = getPropertyDescriptor(t, key);

        try {
            assert keyProperty != null;
            Method keyMethod = keyProperty.getReadMethod();
            return String.valueOf(keyMethod.invoke(t));
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }

    /**
     * 给子级属性赋值
     *
     * @param t    对象
     * @param list 子菜单集合
     */
    private void setValueByProperty(T t, Collection<T> list) {
        PropertyDescriptor keyProperty = getPropertyDescriptor(t, childProperty);
        //获取getCurrentPage()方法
        try {
            assert keyProperty != null;
            Method keyMethod = keyProperty.getWriteMethod();
            keyMethod.invoke(t, list);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 通过反射获取中的属性
     *
     * @param t   对象
     * @param key 属性
     * @return PropertyDescriptor
     */
    private PropertyDescriptor getPropertyDescriptor(T t, String key) {
        Class<?> clazz = t.getClass();
        try {
            return new PropertyDescriptor(key, clazz);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

}
```
***

使用

```java
@Override
    public List<Menu> treeMenu() {
        List<Menu> menuList = list();
        /*
         * 使用构造方法给工具类赋值
         * @param rootValue     顶级菜单的父级值                 这里就是parentId的值
         * @param childKey      实体类中对应父级菜单的属性        这里是parentId
         * @param rootKey       实体类中父级的属性               这里是id
         * @param childProperty 实体类中对应的放置子菜单的属性     这里是children
         * @param list          所有的树状菜单数据
         */
        return new TreeMenuUtil<>(null,"parentId", "id", "children", menuList).rootMenu();
    }
```