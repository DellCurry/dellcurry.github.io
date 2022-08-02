---
title: GreenPlum Append计划生成
categories:
- DataBase
# - Greenplum
# aside: true
---

Append计划主要对应两种SQL，一种是UNION (ALL)，一种是分区表。

<!-- more -->
## 查询树结构
### 分区表
分区表的的查询树和普通表完全相同，此处不再单独列出。
### UNION
Query结构体中，对于UNION语句生成Append计划，最重要的字段是范围表rtable和集合操作setOperations：
```cpp
typedef struct Query
{
    NodeTag        type;
    CmdType        commandType;    /* select|insert|update|delete|utility */
    ...
    List          *rtable;         /* list of range table entries */
    FromExpr      *jointree;       /* table join tree (FROM and WHERE clauses) */
    ...
    Node          *setOperations;  /* set-operation tree if this is top level of
                                    * a UNION/INTERSECT/EXCEPT query */
```
在UNION ALL语句的Query结构体中，rtable是表示UNION ALL所连接的子查询组成的链表，rtable的结点RangesTblEntry的rtekind的类型为RTE_SUBQUERY；FromExpr的fromlist为空；setOperations则是一个SetOperationStmt类型的树状数据结构：
```cpp
typedef enum SetOperation
{
      SETOP_NONE = 0,
      SETOP_UNION,
      SETOP_INTERSECT,
      SETOP_EXCEPT
} SetOperation;

typedef struct SetOperationStmt
{
      NodeTag       type;
      SetOperation  op;                  /* type of set op */
      bool          all;                 /* ALL specified? */
      Node         *larg;                /* left child */
      Node         *rarg;                /* right child */
      /* Eventually add fields for CORRESPONDING spec here */

      /* Fields derived during parse analysis: */
      List         *colTypes;          /* OID list of output column type OIDs */
      List         *colTypmods;        /* integer list of output column typmods */
      List         *colCollations;     /* OID list of output column collation OIDs */
      List         *groupClauses;      /* a list of SortGroupClause's */
      /* groupClauses is NIL if UNION ALL, but must be set otherwise */
} SetOperationStmt;
```
larg和rarg表示子节点，当子节点是叶子结点时，类型为RangeTblRef，表示rtable中的一个subquery；否则，子节点仍为SetOperationStmt类型的结点。下面以一个简单的UNION ALL查询为例，给出Query结构体的关键信息：
```sql
(select * from a) UNION ALL (select * from b) UNION ALL (select * from c)
```
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/320533/1640070034257-75b92386-3498-4eec-b729-5c50b8245881.png#clientId=u3924e523-d89d-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=505&id=ub15c92e1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1010&originWidth=1730&originalType=binary&ratio=1&rotation=0&showTitle=false&size=514208&status=done&style=none&taskId=ub291a4fe-2487-4a44-8019-ea99691d9a9&title=&width=865)


## AppendRelInfo
优化器生成append的关键，是PlannerInfo中的append_rel_list字段：
```cpp
typedef struct PlannerInfo
{
    NodeTag               type;
    Query                *parse;           /* the Query being planned */
    PlannerGlobal        *glob;            /* global info for current planner run */
    Index                 query_level;     /* 1 at the outermost Query */
    struct PlannerInfo   *parent_root;     /* NULL at outermost Query */
    ...
    List                 *append_rel_list; /* list of AppendRelInfos */
    ...
} PlannerInfo;
```
这个字段是一个结点类型为AppendRelInfo的链表，表示了Append的所有子计划：
```cpp
typedef struct AppendRelInfo
{
    NodeTag        type;

    /*
     * These fields uniquely identify this append relationship.  There can be
     * (in fact, always should be) multiple AppendRelInfos for the same
     * parent_relid, but never more than one per child_relid, since a given
     * RTE cannot be a child of more than one append parent.
     */
    Index          parent_relid;    /* RT index of append parent rel */
    Index          child_relid;     /* RT index of append child rel */

    /*
     * For an inheritance appendrel, the parent and child are both regular
     * relations, and we store their rowtype OIDs here for use in translating
     * whole-row Vars.  For a UNION-ALL appendrel, the parent and child are
     * both subqueries with no named rowtype, and we store InvalidOid here.
     */
    Oid            parent_reltype;  /* OID of parent's composite type */
    Oid            child_reltype;   /* OID of child's composite type */

    /*
     * The N'th element of this list is a Var or expression representing the
     * child column corresponding to the N'th column of the parent. This is
     * used to translate Vars referencing the parent rel into references to
     * the child.  A list element is NULL if it corresponds to a dropped
     * column of the parent (this is only possible for inheritance cases, not
     * UNION ALL).  The list elements are always simple Vars for inheritance
     * cases, but can be arbitrary expressions in UNION ALL cases.
     *
     * Notice we only store entries for user columns (attno > 0).  Whole-row
     * Vars are special-cased, and system columns (attno < 0) need no special
     * translation since their attnos are the same for all tables.
     *
     * Caution: the Vars have varlevelsup = 0.  Be careful to adjust as needed
     * when copying into a subquery.
     */
    List          *translated_vars; /* Expressions in the child's Vars */

    /*
     * We store the parent table's OID here for inheritance, or InvalidOid for
     * UNION ALL.  This is only needed to help in generating error messages if
     * an attempt is made to reference a dropped parent column.
     */
    Oid            parent_reloid;   /* OID of parent relation */
```
planner.c的subquery_planner是优化器的主要函数，接收Query结构体，返回执行计划Plan。subquery_planner中，对UNION和分区表，分别调用flatten_simple_union_all()和expand_inherited_tables()来生成append_rel_list：
```cpp
//planner.c

Plan *
subquery_planner(PlannerGlobal *glob, Query *parse,
                 PlannerInfo *parent_root,
                 bool hasRecursion, double tuple_fraction,
                 PlannerInfo **subroot,
                 PlannerConfig *config)
{
    PlannerInfo *root;
    // ...

    /*
     * If this is a simple UNION ALL query, flatten it into an appendrel. We
     * do this now because it requires applying pull_up_subqueries to the leaf
     * queries of the UNION ALL, which weren't touched above because they
     * weren't referenced by the jointree (they will be after we do this).
     */
    if (parse->setOperations)
        flatten_simple_union_all(root);
    
    // ...
    
    /*
     * Expand any rangetable entries that are inheritance sets into "append
     * relations".  This can add entries to the rangetable, but they must be
     * plain base relations not joins, so it's OK (and marginally more
     * efficient) to do it after checking for join RTEs.  We must do it after
     * pulling up subqueries, else we'd fail to handle inherited tables in
     * subqueries.
     */
    expand_inherited_tables(root);
```
### 
### UNION
flatten_simple_union_all中，首先需要找到setoperations树(简称setops)中最左侧的叶子结点(RangeTblRef)，并通过其rtindex字段，找到rtable中对应的RTE：
```cpp
Query               *parse = root->parse;
SetOperationStmt    *topop;
//...
topop = castNode(SetOperationStmt, parse->setOperations);
// ...
leftmostjtnode = topop->larg;
while (leftmostjtnode && IsA(leftmostjtnode, SetOperationStmt))
    leftmostjtnode = ((SetOperationStmt *) leftmostjtnode)->larg;
    Assert(leftmostjtnode && IsA(leftmostjtnode, RangeTblRef));
    leftmostRTI = ((RangeTblRef *) leftmostjtnode)->rtindex;
    leftmostRTE = rt_fetch(leftmostRTI, parse->rtable);
    Assert(leftmostRTE->rtekind == RTE_SUBQUERY);
```
随后，复制一个完全相同的RTE，加入到rtable的末尾，这个新的RTE用来表示这个最左侧叶子结点RTR所指向的RTE，而RTR原本指向的RTE，则在之后用来表示整个append_rel：
```cpp
    /*
     * Make a copy of the leftmost RTE and add it to the rtable.  This copy
     * will represent the leftmost leaf query in its capacity as a member of
     * the appendrel.  The original will represent the appendrel as a whole.
     * (We must do things this way because the upper query's Vars have to be
     * seen as referring to the whole appendrel.)
     */
    childRTE = copyObject(leftmostRTE);
    parse->rtable = lappend(parse->rtable, childRTE);
    childRTI = list_length(parse->rtable);

    /* Modify the setops tree to reference the child copy */
    ((RangeTblRef *) leftmostjtnode)->rtindex = childRTI;

    /* Modify the formerly-leftmost RTE to mark it as an appendrel parent */
    leftmostRTE->inh = true;
```
在完成将origin RTE对应的RTR加入jointree和setops的清理工作后，调用pull_up_union_leaf_queries()，执行append_rel_list的构建工作：
```cpp
    /*
     * Form a RangeTblRef for the appendrel, and insert it into FROM.  The top
     * Query of a setops tree should have had an empty FromClause initially.
     */
    rtr = makeNode(RangeTblRef);
    rtr->rtindex = leftmostRTI;
    Assert(parse->jointree->fromlist == NIL);
    parse->jointree->fromlist = list_make1(rtr);

    /*
     * Now pretend the query has no setops.  We must do this before trying to
     * do subquery pullup, because of Assert in pull_up_simple_subquery.
     */
    parse->setOperations = NULL;

    /*
     * Build AppendRelInfo information, and apply pull_up_subqueries to the
     * leaf queries of the UNION ALL.  (We must do that now because they
     * weren't previously referenced by the jointree, and so were missed by
     * the main invocation of pull_up_subqueries.)
     */
    pull_up_union_leaf_queries((Node *) topop, root, leftmostRTI, parse, 0);
```
pull_up_union_leaf_queries()是一个递归函数，使用DFS，递归遍历整个setops树，遇到RTR类型的叶子结点时，就生成一个对应的AppendRelInfo，并加入到append_rel_list中。AppendRelInfo最重要的字段是parent_relid和child_relid，parent_relid即之前被替代的最左侧叶子结点的原始RTE的rtindex，所有的AppendRelInfo的parent_relid都指向这个RTE，因此上文说，这个RTE是用来表示整个append_rel；child_relid即当前RTR的rtindex。简单总结来说，AppendRelInfo就是把单个subquery所在的RTE作为child，把之前被替代的RTE作为parent，append_rel_list就是这样一个链表。在最后，调用pull_up_subqueries_recurse，在其中会判断是否调用pull_up_simple_subquery来提升subquery（如果提升subquery，AppendRelInfo对应的child_relid也会随之修改）:
```cpp

appinfo = makeNode(AppendRelInfo);
appinfo->parent_relid = parentRTindex;
appinfo->child_relid = childRTindex;
appinfo->parent_reltype = InvalidOid;
appinfo->child_reltype = InvalidOid;
make_setop_translation_list(setOpQuery, childRTindex,
                            &appinfo->translated_vars);
appinfo->parent_reloid = InvalidOid;
root->append_rel_list = lappend(root->append_rel_list, appinfo);

/*
* Recursively apply pull_up_subqueries to the new child RTE.  (We
* must build the AppendRelInfo first, because this will modify it.)
* Note that we can pass NULL for containing-join info even if we're
* actually under an outer join, because the child's expressions
* aren't going to propagate up to the join.  Also, we ignore the
* possibility that pull_up_subqueries_recurse() returns a different
* jointree node than what we pass it; if it does, the important thing
* is that it replaced the child relid in the AppendRelInfo node.
*/
rtr = makeNode(RangeTblRef);
rtr->rtindex = childRTindex;
(void) pull_up_subqueries_recurse(root, (Node *) rtr,
                                          NULL, NULL, appinfo, false);
```
### 分区表
分区表所对应的RTE的inh字段在编译阶段就已经为true。PG12之前，分区表的AppendRelInfo由prepunion.c中的expand_inherited_tables()生成。不论是gp还是pg，分区表都是通过继承表的方式来实现的，因此对于分区表，都是通过调用pg_inherits.c中的find_all_inheritors()，根据分区表的OID，查找其所有子表的OID，作为链表返回（分区表OID作为链表头）。之后，会复制分区表的RTE，将其relid改为子表的OID，将该复制的RTE加入到rtable中。最后，还将对每个子表生成AppendRelInfo，appinfo的parentOID是分区表的rtr，childOID是复制的RTE的rtr。
```cpp
//prepunion.c
/* Scan for all members of inheritance set, acquire needed locks */
inhOIDs = find_all_inheritors(parentOID, lockmode, NULL);

//pg_inherits.c
List *
find_all_inheritors(Oid parentrelId, LOCKMODE lockmode, List **numparents)

```
## 生成计划
对于Append计划节点，首先需要生成AppendPath。allpath.c中的set_rel_pathlist会对每个rte生成对应的path。当rte的inh字段为true时，调用set_append_rel_pathlist生成一个AppendPath。
​

set_append_rel_pathlist中，会遍历append_rel_list，对每个AppendRelInfo指向的子关系调用set_rel_pathlist，最后调用create_append_path，生成AppendPath并调用add_path，将AppendPath加入到RelOptInfo中。






