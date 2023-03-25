# 手动分页工具类

通用手动分页工具类,返回mybatis-plus中IPage\<T\>格式。
<!--more-->
***
## 前言

在开发中，有时会遇到手动分页的情况。现在用mybatis-plus比较多。为了给前端分页格式统一。在手动分页的时候也返回mybatis-plus的分页格式。

## 手动分页通用工具类
pageUtils
```java

package com.yunjia.eduction.utils;

import com.baomidou.mybatisplus.core.metadata.IPage;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;

import java.util.List;
/**
 * @author yxx
 * @since 2023-03-25 14:13
 */
public class PageUtils {
    /**
     * 手动分页
     *
     * @param list     分页前的集合
     * @param pageNo   页码
     * @param pageSize 页数
     * @return 分页后的集合
     */
    public static <T> IPage<T> pageList(List<T> list, int pageNo, int pageSize) {
        //计算总页数
        int page = list.size() % pageSize == 0 ? list.size() / pageSize : list.size() / pageSize + 1;
        //兼容性分页参数错误
        pageNo = pageNo <= 0 ? 1 : pageNo;
        pageNo = Math.min(pageNo, page);
        // 开始索引
        int begin;
        // 结束索引
        int end;
        if (pageNo != page) {
            begin = (pageNo - 1) * pageSize;
            end = begin + pageSize;
        } else {
            begin = (pageNo - 1) * pageSize;
            end = list.size();
        }
        IPage<T> pg = new Page<>();
        pg.setTotal(list.size());
        pg.setCurrent(pageNo);
        pg.setSize(pageSize);
        return pg.setRecords(list.subList(begin, end));
    }
}
```


---

> 作者: map[avatar:<nil> email:yangxx@88.com link:<nil> name:yxx]  
> URL: http://localhost:1313/pageutil/  

