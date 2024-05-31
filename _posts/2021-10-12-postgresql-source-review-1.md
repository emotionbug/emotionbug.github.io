---
title: 'PostgreSQL 소스분석(1) - 데이터 삽입#1'
author: Alex Kwak
date: '2021-10-12 13:16:00 +0900'
categories: [Database, PostgreSQL]
tags: [postgresql, 소스분석]
published: true
---
해당 게시물은 아래 블로그의 내용을 번역한 글입니다.

http://blog.itpub.net/6906/viewspace-2374915/



## 들어가는 글

이 글에서는 PostgreSQL의 데이터 삽입 부분의 소스 코드를 간략히 소개하고, 주로 `PageAddItemExtended` 함수의 로직을 소개함과 동시에 앞서 소개한 페이지 저장 구조를 결합하여 gdb를 통해 데이터 구조를 추적하고 분석한다.

## 1. 테스트 쿼리
```
testdb=# drop table if exists t_insert;
NOTICE: table "t_insert" does not exist, skipping
DROP TABLE
testdb=# create table t_insert(id int,c1 char(10),c2 char(10),c3 char(10));
CREATE TABLE
```



## 2. 소스 코드 분석

데이터 삽입의 주요 구현은 bufpage.c에 있으며, 중요한 함수는 `PageAddItemExtended`입니다. 이 부분에 대한 영어 주석은 매우 명확했지만, 이해를 돕기 위해 여기에 한국어 주석이 추가되었습니다.

### 1. Page
char 포인터
```c
typedef char *Pointer;
typedef Pointer Page;
```

### 2. Item
char 포인터
```c
typedef Pointer Item;
```

### 3. Size
```c
typedef size_t  Size;
```

### 4. OffsetNumber
unsigned short(16bits)
```c
typedef uint16 OffsetNumber;
```

### 5. PageHeader
PageHeaderData 구조체 포인터
```c
typedef struct PageHeaderData
 {
     /* XXX LSN is member of *any* block, not only page-organized ones */
     PageXLogRecPtr pd_lsn;      /* LSN: next byte after last byte of xlog
                                  * record for last change to this page */
     uint16      pd_checksum;    /* checksum */
     uint16      pd_flags;       /* flag bits, see below */
     LocationIndex pd_lower;     /* offset to start of free space */
     LocationIndex pd_upper;     /* offset to end of free space */
     LocationIndex pd_special;   /* offset to start of special space */
     uint16      pd_pagesize_version;
     TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
     ItemIdData  pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
 } PageHeaderData;

 typedef PageHeaderData *PageHeader;

#define PD_HAS_FREE_LINES   0x0001  /* are there any unused line pointers? */
#define PD_PAGE_FULL        0x0002  /* not enough free space for new tuple? */
#define PD_ALL_VISIBLE      0x0004  /* all tuples on page are visible to
                                     * everyone */
#define PD_VALID_FLAG_BITS  0x0007  /* OR of all valid pd_flags bits */
```

### 6. SizeOfPageHeaderData
long형, 헤더의 크기를 정의한다.
```c
#define offsetof(type, field)   ((long) &((type *)0)->field)
#define SizeOfPageHeaderData (offsetof(PageHeaderData, pd_linp))
```

### 7. BLCKSZ
```c
#define BLCKSZ 8192
```

### 8. PageGetMaxOffsetNumber
여유 공간이 Header 크기보다 작거나 같으면 값이 0이고, 그렇지 않으면 하한 값에서 헤더 크기(24Byte)를 뺀 다음 ItemId(4Byte) 크기로 나눕니다.
```c
 #define PageGetMaxOffsetNumber(page) \
     (((PageHeader) (page))->pd_lower <= SizeOfPageHeaderData ? 0 : \
      ((((PageHeader) (page))->pd_lower - SizeOfPageHeaderData) / sizeof(ItemIdData)))
```

### 9. InvalidOffsetNumber
```c
#define InvalidOffsetNumber     ((OffsetNumber) 0)
```

### 10. OffsetNumberNext
입력 값 + 1
```c
#define OffsetNumberNext(offsetNumber) \
     ((OffsetNumber) (1 + (offsetNumber)))
```


### 11. ItemId
ItemIdData 구초체 포인터
```c
typedef struct ItemIdData
{
  unsigned  lp_off:15,		/* offset to tuple (from start of page) */
            lp_flags:2,		/* state of item pointer, see below */
            lp_len:15;		/* byte length of tuple */
} ItemIdData;

typedef ItemIdData *ItemId;

#define LP_UNUSED 0     /* unused (should always have lp_len=0) */
#define LP_NORMAL 1     /* used (should always have lp_len>0) */
#define LP_REDIRECT 2   /* HOT redirect (should have lp_len=0) */
#define LP_DEAD 3       /* dead, may or may not have storage */
```

### 12. PageGetItemId
page와 offesetNumber에 해당하는 ItemIdData의 포인터를 가져옵니다.
```c
#define PageGetItemId(page, offsetNumber) \
    ((ItemId) (&((PageHeader) (page))->pd_linp[(offsetNumber) - 1]))
```

### 13. PAI_OVERWRITE
비트 마크，실제 값은 1이다.
```c
#define PAI_OVERWRITE           (1 << 0)
```
### 14. PAI_IS_HEAP
비트 마크, 실제 값은 2이다.
```c
#define PAI_IS_HEAP             (1 << 1)
```
### 15. ItemIdIsUsed
사용중인 ItemId인지 판별한다.
```c
#define ItemIdIsUsed(itemId) \
     ((itemId)->lp_flags != LP_UNUSED)
```

### 16. ItemIdHasStorage
```c
#define ItemIdHasStorage(itemId) \
     ((itemId)->lp_len != 0
```

### 17. PageHasFreeLinePointers/PageClearHasFreeLinePointers

페이지의 머리글과 하단 사이에 공백이 있는지 확인합니다.
```c
#define PageHasFreeLinePointers(page) \
    (((PageHeader) (page))->pd_flags & PD_HAS_FREE_LINES)
```

빈 마커 지우기(마커 오류인 경우 지우기)
```c
#define PageClearHasFreeLinePointers(page) \
    (((PageHeader) (page))->pd_flags &= ~PD_HAS_FREE_LINES)
```

### 18. MaxHeapTuplesPerPage
각 페이지가 보유할 수 있는 최대 튜플 수를 계산하는 공식은 다음과 같습니다.

(Block Size - Page Header Size) / (Head size after alignment + Row Pointer Size)
```c
#define MaxHeapTuplesPerPage    \
     ((int) ((BLCKSZ - SizeOfPageHeaderData) / \
             (MAXALIGN(SizeofHeapTupleHeader) + sizeof(ItemIdData))))
```


## 3. PageAddItemExtended 함수 분석
```c
/*
PageAddItemExtended 함수：
입력：
  page-페이지 포인터
  item-데이터 포인터
  size-데이터 크기
  offsetNumber-데이터 저장소의 오프셋 지정
  flags-marker bit（덮어쓸지 여부/힙 데이터 가용성）
출력：
  OffsetNumber-데이터 저장소의 실제 오프셋
*/
OffsetNumber
PageAddItemExtended(Page page,
                    Item item,
                    Size size,
                    OffsetNumber offsetNumber,
                    int flags)
{
    PageHeader  phdr = (PageHeader) page; // 헤더 포인터
    Size        alignedSize; // 정렬된 크기
    int         lower; // 여유 공간의 상한
    int         upper; // 여유 공간의 하한
    ItemId      itemId; // 행 포인터
    OffsetNumber limit; // 행 오프셋，사용 가능한 공간 중 첫번째 위치
    bool        needshuffle = false; // 원본 데이터를 옮길 필요가 있는가?

    /*
     * Be wary about corrupted page pointers
     */
    if (phdr->pd_lower < SizeOfPageHeaderData ||
        phdr->pd_lower > phdr->pd_upper ||
        phdr->pd_upper > phdr->pd_special ||
        phdr->pd_special > BLCKSZ)
        ereport(PANIC,
                (errcode(ERRCODE_DATA_CORRUPTED),
                 errmsg("corrupted page pointers: lower = %u, upper = %u, special = %u",
                        phdr->pd_lower, phdr->pd_upper, phdr->pd_special)));

    /*
     * Select offsetNumber to place the new item at
     */
    // 저장소 데이터의 오프셋을 가져온다. ( 하한과 상한 사이 )
    limit = OffsetNumberNext(PageGetMaxOffsetNumber(page));

    /* was offsetNumber passed in? */
    if (OffsetNumberIsValid(offsetNumber))
    {
        // 데이터 저장소 오프셋이 지정된 경우 (매개변수로 전달된 오프셋이 유효함)
        /* yes, check it */
        if ((flags & PAI_OVERWRITE) != 0) // 원본 데이터를 덮어쓰지 않는 경우
        {
            if (offsetNumber < limit)
            {
                // 지정된 오프셋에서 ItemId를 가져온다.
                itemId = PageGetItemId(phdr, offsetNumber);
                // 지정된 데이터 오프셋이 사용중이거나, 저장소 공간이 할당되어 있을 경우 오류가 발생한 것으로 간주한다.
                if (ItemIdIsUsed(itemId) || ItemIdHasStorage(itemId))
                {
                    elog(WARNING, "will not overwrite a used ItemId");
                    return InvalidOffsetNumber;
                }
            }
        }
        else // 원본 데이터를 덮어쓰는 경우
        {
            // 지정된 행 오프셋에 사용가능한 공간이 없을 경우, 원래 데이터를 새로운 공간으로 이동해야 한다.
            if (offsetNumber < limit)
                needshuffle = true; /* need to move existing linp's */
        }
    }
    else // 데이터 저장을 위한 행 오프셋이 지정되지 않았을 경우
    {
        /* offsetNumber was not passed in, so find a free slot */
        /* if no free slot, we'll put it at limit (1st open slot) */
        if (PageHasFreeLinePointers(phdr)) // 페이지에 빈 행 공간이 있을 경우
        {
            /*
             * Look for "recyclable" (unused) ItemId.  We check for no storage
             * as well, just to be paranoid --- unused items should never have
             * storage.
             */
            // 페이지를 순회하여, 사용 가능한 첫 번째 행 오프셋을 찾는다.
            for (offsetNumber = 1; offsetNumber < limit; offsetNumber++)
            {
                itemId = PageGetItemId(phdr, offsetNumber);
                if (!ItemIdIsUsed(itemId) && !ItemIdHasStorage(itemId))
                    break;
            }
            // 유휴 공간을 찾지 못했을 경우, 헤더가 잘못된 플래그를 가지고 있다고 판단한다.
            // 같은 오류가 반복되지 않도록, 헤더의 마크를 갱신한다.
            if (offsetNumber >= limit)
            {
                /* the hint is wrong, so reset it */
                PageClearHasFreeLinePointers(phdr);
            }
        }
        else // 회수 가능한 공간 및 여유 공간이 없는 경우
        {
            /* don't bother searching if hint says there's no free slot */
            offsetNumber = limit;
        }
    }

    /* Reject placing items beyond the first unused line pointer */
    if (offsetNumber > limit)
    {
        // 지정된 오프셋이 사용 가능한 첫 번째 위치보다 큰 경우 오류를 발생시킨다.
        elog(WARNING, "specified item offset is too large");
        return InvalidOffsetNumber;
    }

    /* Reject placing items beyond heap boundary, if heap */
    if ((flags & PAI_IS_HEAP) != 0 && offsetNumber > MaxHeapTuplesPerPage)
    {
        // 힙 데이터. 하지만, 페이지에 최대로 저장할 수 있는 튜플 수보다 오프셋이 큰 경우 오류를 발생시킴.
        elog(WARNING, "can't put more than MaxHeapTuplesPerPage items in a heap page");
        return InvalidOffsetNumber;
    }

    /*
     * Compute new lower and upper pointers for page, see if it'll fit.
     *
     * Note: do arithmetic as signed ints, to avoid mistakes if, say,
     * alignedSize > pd_upper.
     */
    if (offsetNumber == limit || needshuffle)
        // 데이터가 여유 공간에 저장된 경우, 하한 값을 수정한다.
        lower = phdr->pd_lower + sizeof(ItemIdData);
    else
        // 그렇지 않을 경우, 빈 공간을 찾아 원래의 하한 값을 사용한다.
        lower = phdr->pd_lower;

    alignedSize = MAXALIGN(size); // 정렬된 크기

    upper = (int) phdr->pd_upper - (int) alignedSize;

    if (lower > upper)
        return InvalidOffsetNumber;

    /*
     * OK to insert the item.  First, shuffle the existing pointers if needed.
     */
    // 행의 포인터를 가져온다.
    itemId = PageGetItemId(phdr, offsetNumber);

    if (needshuffle)
        // 공간을 확보하기 위해, 원본 데이터를 뒤로 이동시킨다.
        memmove(itemId + 1, itemId,
                (limit - offsetNumber) * sizeof(ItemIdData));

    /* set the item pointer */
    // 새 데이터 행 포인터 설정
    ItemIdSetNormal(itemId, upper, size);

    /*
     * Items normally contain no uninitialized bytes.  Core bufpage consumers
     * conform, but this is not a necessary coding rule; a new index AM could
     * opt to depart from it.  However, data type input functions and other
     * C-language functions that synthesize datums should initialize all
     * bytes; datumIsEqual() relies on this.  Testing here, along with the
     * similar check in printtup(), helps to catch such mistakes.
     *
     * Values of the "name" type retrieved via index-only scans may contain
     * uninitialized bytes; see comment in btrescan().  Valgrind will report
     * this as an error, but it is safe to ignore.
     */
    VALGRIND_CHECK_MEM_IS_DEFINED(item, size);

    /* copy the item's data onto the page */
    // 페에지에 데이터 적재
    memcpy((char *) page + upper, item, size);

    /* adjust page header */
    // 페이지의 헤더를 갱신한다.
    phdr->pd_lower = (LocationIndex) lower;
    phdr->pd_upper = (LocationIndex) upper;
    // 실제 행 오프셋을 반환한다.
    return offsetNumber;
}

```





## 4. 디버깅

다음은 `gdb`를 이용한 `PageAddItemextended` 함수의 디버깅입니다.

테스트 시나리오: 먼저 데이터 8줄을 삽입한 다음 두 번째 줄을 삭제하고 데이터 1줄을 삽입합니다.

```
testdb=# -- 8개 행의 데이터 삽입
testdb=# insert into t_insert values(1,'11','12','13');
insert into t_insert values(6,'11','12','13');
insert into t_insert values(7,'11','12','13');
insert into t_insert values(8,'11','12','13');

checkpoint;INSERT 0 1
testdb=# insert into t_insert values(2,'11','12','13');
INSERT 0 1
testdb=# insert into t_insert values(3,'11','12','13');
INSERT 0 1
testdb=# insert into t_insert values(4,'11','12','13');
INSERT 0 1
testdb=# insert into t_insert values(5,'11','12','13');
INSERT 0 1
testdb=# insert into t_insert values(6,'11','12','13');
INSERT 0 1
testdb=# insert into t_insert values(7,'11','12','13');
INSERT 0 1
testdb=# insert into t_insert values(8,'11','12','13');
INSERT 0 1
testdb=#
testdb=# checkpoint;
CHECKPOINT
testdb=# -- 2번 행 삭제
testdb=# delete from t_insert where id = 2;
DELETE 1
testdb=#
testdb=# checkpoint;
CHECKPOINT
testdb=#
testdb=# -- pid를 가져온다.
testdb=# select pg_backend_pid();
 pg_backend_pid
----------------
           1572
(1 row)
```

gdb를 사용하여 데이터의 마지막 줄을 삽입하는 프로세스를 추적합니다.

```
# gdb 실행 및 프로세스 바인딩
[root@localhost demo]# gdb -p 1572
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<
Attaching to process 1572
...
(gdb)
# Breakpoint를 설정한다.
(gdb) b PageAddItemExtended
Breakpoint 1 at 0x845119: file bufpage.c, line 196.
(gdb)

# PSQL로 전환하여, 데이터를 삽입한다.
testdb=# insert into t_insert values(9,'11','12','13');
（보류）

# GDB로 돌아와서..
(gdb) c
Continuing.

Breakpoint 1, PageAddItemExtended (page=0x7feaaefac300 "\001", item=0x29859f8 "2\234\030", size=61, offsetNumber=0, flags=2) at bufpage.c:196
196     PageHeader  phdr = (PageHeader) page;
# 중단점은 함수 첫번째 줄에 있음.
# bt는 call stack을 출력한다.
(gdb) bt
#0  PageAddItemExtended (page=0x7feaaefac300 "\001", item=0x29859f8 "2\234\030", size=61, offsetNumber=0, flags=2) at bufpage.c:196
#1  0x00000000004cf4f9 in RelationPutHeapTuple (relation=0x7feac6e2ccb8, buffer=141, tuple=0x29859e0, token=false) at hio.c:53
#2  0x00000000004c34ec in heap_insert (relation=0x7feac6e2ccb8, tup=0x29859e0, cid=0, options=0, bistate=0x0) at heapam.c:2487
#3  0x00000000006c076b in ExecInsert (mtstate=0x2984c10, slot=0x2985250, planSlot=0x2985250, estate=0x29848c0, canSetTag=true) at nodeModifyTable.c:529
#4  0x00000000006c29f3 in ExecModifyTable (pstate=0x2984c10) at nodeModifyTable.c:2126
#5  0x000000000069a7d8 in ExecProcNodeFirst (node=0x2984c10) at execProcnode.c:445
#6  0x0000000000690994 in ExecProcNode (node=0x2984c10) at ../../../src/include/executor/executor.h:237
#7  0x0000000000692e5e in ExecutePlan (estate=0x29848c0, planstate=0x2984c10, use_parallel_mode=false, operation=CMD_INSERT, sendTuples=false, numberTuples=0, direction=ForwardScanDirection,
    dest=0x2990dc8, execute_once=true) at execMain.c:1726
#8  0x0000000000690e58 in standard_ExecutorRun (queryDesc=0x2981020, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:363
#9  0x0000000000690cef in ExecutorRun (queryDesc=0x2981020, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:306
#10 0x0000000000851d84 in ProcessQuery (plan=0x2990c68, sourceText=0x28c5ef0 "insert into t_insert values(9,'11','12','13');", params=0x0, queryEnv=0x0, dest=0x2990dc8, completionTag=0x7ffdbc052d10 "")
    at pquery.c:161
#11 0x00000000008534f4 in PortalRunMulti (portal=0x292b490, isTopLevel=true, setHoldSnapshot=false, dest=0x2990dc8, altdest=0x2990dc8, completionTag=0x7ffdbc052d10 "") at pquery.c:1286
#12 0x0000000000852b32 in PortalRun (portal=0x292b490, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x2990dc8, altdest=0x2990dc8, completionTag=0x7ffdbc052d10 "") at pquery.c:799
#13 0x000000000084cebc in exec_simple_query (query_string=0x28c5ef0 "insert into t_insert values(9,'11','12','13');") at postgres.c:1122
#14 0x0000000000850f3c in PostgresMain (argc=1, argv=0x28efaa8, dbname=0x28ef990 "testdb", username=0x28ef978 "xdb") at postgres.c:4153
#15 0x00000000007c0168 in BackendRun (port=0x28e7970) at postmaster.c:4361
#16 0x00000000007bf8fc in BackendStartup (port=0x28e7970) at postmaster.c:4033
#17 0x00000000007bc139 in ServerLoop () at postmaster.c:1706
#18 0x00000000007bb9f9 in PostmasterMain (argc=1, argv=0x28c0b60) at postmaster.c:1379
#19 0x00000000006f19e8 in main (argc=1, argv=0x28c0b60) at main.c:228

# next 명령을 사용하여, 단계적으로 절차를 진행한다.
(gdb) next
207     if (phdr->pd_lower < SizeOfPageHeaderData ||
(gdb) next
208         phdr->pd_lower > phdr->pd_upper ||
(gdb) next
207     if (phdr->pd_lower < SizeOfPageHeaderData ||
(gdb) next
209         phdr->pd_upper > phdr->pd_special ||
(gdb) next
208         phdr->pd_lower > phdr->pd_upper ||
(gdb) next
210         phdr->pd_special > BLCKSZ)
(gdb) next
209         phdr->pd_upper > phdr->pd_special ||
(gdb) next
219     limit = OffsetNumberNext(PageGetMaxOffsetNumber(page));
(gdb) next
222     if (OffsetNumberIsValid(offsetNumber))
(gdb) p offsetNumber
$1 = 0
# offsetNumber는 0번이며，삽입을 위한 행 오프셋이 지정되지 않았다.
(gdb) next
247         if (PageHasFreeLinePointers(phdr))
# 페이지 헤더에 대한 정보를 확인한다.
(gdb) p *phdr
$2 = {pd_lsn = {xlogid = 1, xrecoff = 3677462648}, pd_checksum = 0, pd_flags = 0, pd_lower = 56, pd_upper = 7680, pd_special = 8192, pd_pagesize_version = 8196, pd_prune_xid = 1612849,
  pd_linp = 0x7feaaefac318}
(gdb)
# 행 포인터에 대한 정보를 확인한다.
(gdb) p phdr->pd_linp[0]
$3 = {lp_off = 8128, lp_flags = 1, lp_len = 61}
(gdb) p phdr->pd_linp[1]
$4 = {lp_off = 8064, lp_flags = 1, lp_len = 61}
...
(gdb) p phdr->pd_linp[8]
$11 = {lp_off = 0, lp_flags = 0, lp_len = 0}
# 행 포인터 오프셋이 구조체는 모두 lp_flags가 1(LP_NORMAL)로 사용중임을 가르킨다.
...
298     alignedSize = MAXALIGN(size);
(gdb) p size
$17 = 61
(gdb) p alignedSize
$18 = 5045956
(gdb) next
300     upper = (int) phdr->pd_upper - (int) alignedSize;
(gdb) p alignedSize
$19 = 64 # 원본 61， 정렬된 크기 64
(gdb) next
302     if (lower > upper)
(gdb) p lower
$20 = 60
(gdb) p upper
$21 = 7616
(gdb) next
308     itemId = PageGetItemId(phdr, offsetNumber);
(gdb)
310     if (needshuffle)
(gdb) next
315     ItemIdSetNormal(itemId, upper, size);
(gdb) next
332     memcpy((char *) page + upper, item, size);
(gdb) p *itemId
$23 = {lp_off = 7616, lp_flags = 1, lp_len = 61}
...
(gdb) next
338     return offsetNumber;
(gdb)
# 데이터는 행 오프셋이 9인 위치에 삽입됩니다 (행 포인터 값 = 8).
(gdb) p offsetNumber
$24 = 9
(gdb) p phdr->pd_linp[8]
$25 = {lp_off = 7616, lp_flags = 1, lp_len = 61}
...
# 설명: 새로 삽입된 데이터가 두 번째 오프셋에 배치되지 않은 이유는 두 번째 삭제된 행이 반환되지 않았기 때문입니다.
# 페이지에서 데이터 확인
testdb=# select * from heap_page_items(get_raw_page('t_insert',0));
 lp | lp_off | lp_flags | lp_len | t_xmin  | t_xmax  | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |                                    t_data

----+--------+----------+--------+---------+---------+----------+--------+-------------+------------+--------+--------+-------+---------------------------------------------------------------------------
---
  1 |   8128 |        1 |     61 | 1612841 |       0 |        0 | (0,1)  |           4 |       2306 |     24 |        |       | \x010000001731312020202020202020173132202020202020202017313320202020202020
20
  2 |   8064 |        1 |     61 | 1612842 | 1612849 |        0 | (0,2)  |        8196 |        258 |     24 |        |       | \x020000001731312020202020202020173132202020202020202017313320202020202020
20
  3 |   8000 |        1 |     61 | 1612843 |       0 |        0 | (0,3)  |           4 |       2306 |     24 |        |       | \x030000001731312020202020202020173132202020202020202017313320202020202020
20
  4 |   7936 |        1 |     61 | 1612844 |       0 |        0 | (0,4)  |           4 |       2306 |     24 |        |       | \x040000001731312020202020202020173132202020202020202017313320202020202020
20
  5 |   7872 |        1 |     61 | 1612845 |       0 |        0 | (0,5)  |           4 |       2306 |     24 |        |       | \x050000001731312020202020202020173132202020202020202017313320202020202020
20
  6 |   7808 |        1 |     61 | 1612846 |       0 |        0 | (0,6)  |           4 |       2306 |     24 |        |       | \x060000001731312020202020202020173132202020202020202017313320202020202020
20
  7 |   7744 |        1 |     61 | 1612847 |       0 |        0 | (0,7)  |           4 |       2306 |     24 |        |       | \x070000001731312020202020202020173132202020202020202017313320202020202020
20
  8 |   7680 |        1 |     61 | 1612848 |       0 |        0 | (0,8)  |           4 |       2306 |     24 |        |       | \x080000001731312020202020202020173132202020202020202017313320202020202020
20
  9 |   7616 |        1 |     61 | 1612850 |       0 |        0 | (0,9)  |           4 |       2050 |     24 |        |       | \x090000001731312020202020202020173132202020202020202017313320202020202020
20
(9 rows)
```



## 5. 결론

1. `PageAddItemExtended` function 구현에 대한 기본적인 지식

2. GDB 디버깅을 통한 구현 로직에 대한 지식



다음 섹션에서는 bottom-up 관점에서 `RelationPutHeapTuple` 함수를 살펴보도록 합니다.

