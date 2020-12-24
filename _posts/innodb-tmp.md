---
layout: post
title: "Innodb-tmp"
author: "Dapaopao"
tags: ["innodb"]
---

# InnoDB XXX

## Record Format

下面是mysql文档中的旧格式的record描述。

| **Name**            | **Size**             |
| :------------------ | -------------------- |
| Field Start Offsets | (F*1) or (F*2) bytes |
| Extra Bytes         | 6 bytes              |
| Field Contents      | depends on content   |

| **Name**         | **Size** | **Description**                                              |
| ---------------- | -------- | ------------------------------------------------------------ |
| **info_bits:**   | ??       | ??                                                           |
| ()               | 1 bit    | unused or unknown                                            |
| ()               | 1 bit    | unused or unknown                                            |
| deleted_flag     | 1 bit    | 1 if record is deleted                                       |
| min_rec_flag     | 1 bit    | 1 if record is predefined minimum record                     |
| n_owned          | 4 bits   | number of records owned by this record                       |
| heap_no          | 13 bits  | record's order number in heap of index page                  |
| n_fields         | 10 bits  | number of fields in this record, 1 to 1023                   |
| 1byte_offs_flag  | 1 bit    | 1 if each Field Start Offsets is 1 byte long (this item is also called the "short" flag) |
| **next 16 bits** | 16 bits  | pointer to next record in page                               |
| **TOTAL**        | 48 bits  | ??                                                           |

1. 当record的size小于127，`Field Start Offsets`中每个field的offset只占1个byte。否则占2个byte，通过1byte_offs_flag标识。
2. `Field Start Offsets`中每个field的offset的第一个bit表明field是否为null。对于2byte大小的field offset，第二个bit表明字段内容是否在其他page，这种情况说明是一个large blob。
3. Field offset的顺序和后面字段内容出现的顺序相反。
4. Pointer to next record in page指向的是下一个record中field contents的起始偏移。
5.  每个record默认有these system columns，the row ID, the transaction ID, and the rollback pointer。

以上为官方文档中的格式，看了下代码目前新的格式是`compact`，有一点变化。主要多了一个bitmap判断nullable的字段是否为null，对于null字段，不在field start offset和field content中占据任何空间。定长字段不在field start offset中占据空间。



`compact`类型的`record`格式

| 变长字段offset列表     | NULL标志位                         | 记录头信息 | 列1数据 | 列2数据 | ...  |
| ---------------------- | ---------------------------------- | ---------- | ------- | ------- | ---- |
| 只包含变长字段的offset | bitmap表示nullable的字段是否为null | 5 byte     | ...     | ...     | ...  |





```cpp
/** The following function is used to get an offset to the nth data field in a
record.
@param[in] offsets    array returned by rec_get_offsets()
@param[in] n  index of the field
@param[out]    len    length of the field; UNIV_SQL_NULL if SQL null;
                        UNIV_SQL_ADD_COL_DEFAULT if it's default value and no
value inlined
@return offset from the origin of rec */
UNIV_INLINE
ulint rec_get_nth_field_offs(const ulint *offsets, ulint n, ulint *len);
// 获取第n个字段的在record的偏移和长度。

/** The following function determines the offsets to each field
 in the record. It can reuse a previously allocated array.
 Note that after instant ADD COLUMN, if this is a record
 from clustered index, fields in the record may be less than
 the fields defined in the clustered index. So the offsets
 size is allocated according to the clustered index fields.
 @return the new offsets */
ulint *rec_get_offsets_func(
    const rec_t *rec,          /*!< in: physical record */
    const dict_index_t *index, /*!< in: record descriptor */
    ulint *offsets,            /*!< in/out: array consisting of
                               offsets[0] allocated elements,
                               or an array from rec_get_offsets(),
                               or NULL */
    ulint n_fields,            /*!< in: maximum number of
                              initialized fields
                               (ULINT_UNDEFINED if all fields) */
#ifdef UNIV_DEBUG
    const char *file,  /*!< in: file name where called */
    ulint line,        /*!< in: line number where called */
#endif                 /* UNIV_DEBUG */
    mem_heap_t **heap) /*!< in/out: memory heap */
    MY_ATTRIBUTE((warn_unused_result));
// 获取所有字段在record的偏移和长度。

/** Determine the offset to each field in a leaf-page record
 in ROW_FORMAT=COMPACT.  This is a special case of
 rec_init_offsets() and rec_get_offsets_func(). */
UNIV_INLINE
void rec_init_offsets_comp_ordinary(
    const rec_t *rec,          /*!< in: physical record in
                               ROW_FORMAT=COMPACT */
    bool temp,                 /*!< in: whether to use the
                               format for temporary files in
                               index creation */
    const dict_index_t *index, /*!< in: record descriptor */
    ulint *offsets);           /*!< in/out: array of offsets;
                               in: n=rec_offs_n_fields(offsets) */
// compact格式下获取所有字段在record的偏移和长度。
```



## 代码文件

include/rem0cmp.h

include/rem0cmp.ic

include/rem0rec.h

include/rem0rec.ic

rem/rec.cc

rem/rec.h

rem/rem0cmp.cc

rem/rem0rec.cc

rem指的是record manager，cmp指的是comparison，rec指的是record。



## Page Format

16KB固定大小。

### Fil Header

include/fil0fil.h

fil/fil0fil.cc

`FIL_PAGE_SPACE_OR_CHKSUM:` 这个占用四字节，主要用来存储数据页的checksum。注意，计算校验值的时候，并不是整个数据页都计算，有几个地方是不计算进去的(`buf_calc_page_crc32`和`buf_calc_page_new_checksum`)，例如头尾存checksum的地方，存space_id的地方(历史原因导致)。Checksum的计算方式详见`数据页Corruption`这一小节。

`FIL_PAGE_OFFSET:` 这个就是对应数据页的page number，每个表空间从0开始，即这个值乘以数据页的大小就可以得到数据页在文件中的起始偏移量。`fio_io`函数读取以及写入数据页的时候依赖这个规则。

`FIL_PAGE_PREV,FIL_PAGE_NEXT:` 这两个是指针，分别指向前一个数据页和后一个数据页。注意，这里的前后是指按照用户记录排序的先后顺序，也是逻辑顺序。因为在InnoDB数据页不断的分配和释放中，会导致逻辑上连续的数据页在物理上不连续。所以需要指针链接。前后两个指针共同构建了一个双向链表。

`FIL_PAGE_LSN:` 当前数据页最新被修改的lsn。这个字段非常重要，InnoDB redolog幂等的特性就依赖此字段。在奔溃恢复应用日志阶段，如果发现redolog的lsn小于等于这个值，就不需要再次应用redolog了。

`FIL_PAGE_TYPE:` 当前页面是哪种类型的数据页。包括，索引页，Undo页，Inode页，系统页，BloB页等十几种。

`FIL_PAGE_FILE_FLUSH_LSN:` ibdata文件第一个数据页才有意义，记录ibdata成功刷到磁盘的lsn。

`FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID:` 现在的版本就是用来存spaceid的。

```cpp
/** Get the successor of a file page.
@param[in] page      File page
@return FIL_PAGE_NEXT */
page_no_t fil_page_get_next(const byte *page)
    MY_ATTRIBUTE((warn_unused_result));
// 直接读取page头部偏移12个字节开始的大端int。
```



### Page Header

include/page0page.h

include/page0types.h

include/page0page.ic



include/page0cur.h

page/page0cur.cc

```cpp
/** Inserts a record next to page cursor on an uncompressed page.
 Returns pointer to inserted record if succeed, i.e., enough
 space available, NULL otherwise. The cursor stays at the same position.
 @return pointer to record if succeed, NULL otherwise */
rec_t *page_cur_insert_rec_low(
    rec_t *current_rec,  /*!< in: pointer to current record after
                     which the new record is inserted */
    dict_index_t *index, /*!< in: record descriptor */
    const rec_t *rec,    /*!< in: pointer to a physical record */
    ulint *offsets,      /*!< in/out: rec_get_offsets(rec, index) */
    mtr_t *mtr)          /*!< in: mini-transaction handle, or NULL */
    MY_ATTRIBUTE((warn_unused_result));
// page内写入一个新的record。
```

page_cur_insert_rec_low：

1. 从函数参数的offsets获得record的总大小。
2. 



- Infimum + Supremum Records
- User Records
- Free Space
- Page Directory
- Fil Trailer







## 参考资料

https://dev.mysql.com/doc/internals/en/innodb-record-structure.html



