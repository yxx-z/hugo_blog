# java中通用树状菜单工具类

开发中经常遇到树状结构，每次都要写比较麻烦。这里写了一个工具类简化其中的步骤
<!--more-->
这里用了java8函数式编程特性
话不多说，上代码
****
</br>

### 菜单类
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

### 工具类
```java
package com.yunjia.eduction.utils.tree;

import java.util.*;
import java.util.function.BiConsumer;
import java.util.function.Function;
import java.util.stream.Collectors;

/**
 * @author yxx
 * @apiNote 通用树形工具类
 * @since 2023-03-11 14:22
 */
public class TreeUtil {


    /***
     * 构建树
     *
     * @param listData 需要构建的结果集
     * @param parentKeyFunction 父节点id
     * @param keyFunction 主键
     * @param setChildrenFunction 子集
     * @param rootParentValue 父节点的值 0就填0 null就填null
     * @return java.util.List<T>
     */
    public static <T, U extends Comparable> List<T> buildTree(List<T> listData, Function<T, U> parentKeyFunction,
                                                              Function<T, ? extends Comparable> keyFunction, BiConsumer<T, List<T>> setChildrenFunction, U rootParentValue) {
        return buildTree(listData, parentKeyFunction, keyFunction, setChildrenFunction, rootParentValue, null, null);
    }

    /***
     * 构建树，并且对结果集做升序处理
     *
     * @param listData 需要构建的结构集
     * @param parentKeyFunction 父节点
     * @param keyFunction 主键
     * @param setChildrenFunction 子集
     * @param rootParentValue 父节点的值
     * @param sortFunction 排序字段
     * @return java.util.List<T>
     */
    public static <T, U extends Comparable> List<T> buildAscTree(List<T> listData, Function<T, U> parentKeyFunction,
                                                                 Function<T, ? extends Comparable> keyFunction, BiConsumer<T, List<T>> setChildrenFunction, U rootParentValue,
                                                                 Function<T, ? extends Comparable> sortFunction) {
        List<T> resultList =
                buildTree(listData, parentKeyFunction, keyFunction, setChildrenFunction, rootParentValue, sortFunction, 0);

        return sortList(resultList, sortFunction, 0);
    }

    /***
     * 构建树，并且对结果集做降序处理
     *
     * @param listData 需要构建的结构集
     * @param parentKeyFunction 父节点
     * @param keyFunction 主键
     * @param setChildrenFunction 子集
     * @param rootParentValue 父节点的值
     * @param sortFunction 排序字段
     * @return java.util.List<T>
     */
    public static <T, U extends Comparable> List<T> buildDescTree(List<T> listData, Function<T, U> parentKeyFunction,
                                                                  Function<T, ? extends Comparable> keyFunction, BiConsumer<T, List<T>> setChildrenFunction, U rootParentValue,
                                                                  Function<T, ? extends Comparable> sortFunction) {
        List<T> resultList =
                buildTree(listData, parentKeyFunction, keyFunction, setChildrenFunction, rootParentValue, sortFunction, 1);

        return sortList(resultList, sortFunction, 1);
    }

    private static <T, U extends Comparable> List<T> buildTree(List<T> listData, Function<T, U> parentKeyFunction,
                                                               Function<T, ? extends Comparable> keyFunction, BiConsumer<T, List<T>> setChildrenFunction, U rootParentValue,
                                                               Function<T, ? extends Comparable> sortFunction, Integer sortedFlag) {
        // 筛选出根节点
        Map<Comparable, T> rootKeyMap = new HashMap<>();
        // 所有的节点
        Map<Comparable, T> allKeyMap = new HashMap<>();
        // 存id:List<ChildrenId>
        Map<Comparable, List<Comparable>> keyParentKeyMap = new HashMap<>();

        for (T t : listData) {
            Comparable key = keyFunction.apply(t);
            Comparable parentKey = parentKeyFunction.apply(t);
            // 如果根节点标识为null，值也为null，表示为根节点
            if (rootParentValue == null && parentKeyFunction.apply(t) == null) {
                rootKeyMap.put(key, t);
            }
            // 根节点标识有值，值相同表示为根节点
            if (rootParentValue != null && parentKeyFunction.apply(t).compareTo(rootParentValue) == 0) {
                rootKeyMap.put(key, t);
            }
            allKeyMap.put(key, t);
            if (parentKey != null) {
                List<Comparable> children = keyParentKeyMap.getOrDefault(parentKey, new ArrayList<>());
                children.add(key);
                keyParentKeyMap.put(parentKey, children);
            }
        }
        List<T> returnList = new ArrayList<>();
        // 封装根节点数据
        for (Comparable comparable : rootKeyMap.keySet()) {
            setChildren(comparable, returnList, allKeyMap, keyParentKeyMap, setChildrenFunction, sortFunction,
                    sortedFlag);
        }

        return returnList;
    }

    private static <T> void setChildren(Comparable comparable, List<T> childrenList, Map<Comparable, T> childrenKeyMap,
                                        Map<Comparable, List<Comparable>> keyParentKeyMap, BiConsumer<T, List<T>> setChildrenFunction,
                                        Function<T, ? extends Comparable> sortFunction, Integer sortedFlag) {
        T t = childrenKeyMap.get(comparable);
        if (keyParentKeyMap.containsKey(comparable)) {
            List<T> subChildrenList = new ArrayList<>();
            List<Comparable> childrenComparable = keyParentKeyMap.get(comparable);
            for (Comparable c : childrenComparable) {
                setChildren(c, subChildrenList, childrenKeyMap, keyParentKeyMap, setChildrenFunction, sortFunction,
                        sortedFlag);
            }
            subChildrenList = sortList(subChildrenList, sortFunction, sortedFlag);
            setChildrenFunction.accept(t, subChildrenList);

        }
        childrenList.add(t);

    }

    private static <T> List<T> sortList(List<T> list, Function<T, ? extends Comparable> sortFunction,
                                        Integer sortedFlag) {
        if (sortFunction != null) {
            if (sortedFlag == 1) {
                return (List<T>) list.stream().sorted(Comparator.comparing(sortFunction).reversed())
                        .collect(Collectors.toList());
            } else {
                return (List<T>) list.stream().sorted(Comparator.comparing(sortFunction)).collect(Collectors.toList());
            }
        }
        return list;
    }
}
```
***

### 使用
```java
@Override
    public List<Menu> treeMenu(List<Menu> menuList) {
        /*
         * 构建树
         *
         * @param listData 需要构建的结果集              这里是menuList
         * @param parentKeyFunction 父节点             这里是parentId
         * @param keyFunction 主键                     这里是id
         * @param setChildrenFunction 子集             这里是children
         * @param rootParentValue 父节点的值            这里null
         * @return java.util.List<T>
         */
        return TreeUtil.buildTree(menuList, Menu::getParentId, Menu::getId, Menu::setChildren, null);
    }
```

---

> 作者: map[avatar:<nil> email:<nil> link:<nil> name:yxx]  
> URL: http://localhost:1313/tree/  

