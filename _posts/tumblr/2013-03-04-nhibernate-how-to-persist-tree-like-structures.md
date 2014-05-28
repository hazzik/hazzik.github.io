---
layout: post
title: 'NHibernate: How to persist Tree-like structures'
date: '2013-03-04T18:21:00+13:00'
tags:
- Tree
- NHibernate
- Tree-Structures
- Tree-Hierarchy
- Persistent-Tree
tumblr_url: http://hazzik.tumblr.com/post/44522958800/nhibernate-how-to-persist-tree-like-structures
redirect_from:
- /post/44522958800/nhibernate-how-to-persist-tree-like-structures/
- /post/44522958800/
---
Many of you have seen a simple problem - how to store hierarchy data in relational database and efficient work with it. It seems that there is no simpler things to do - just create PARENT_ID column and store the parent identifier of this record.

```csharp
class Tree {
    int Id;
    Tree Parent;
}
```

But it is only at first glance.

Everything is fine until you are working with only one level of an hierarchy: an parent and his children. But the funny part begins when you need to extend your levels, for example you need to check an ancestor of entity on any of the upper levels.



This task can not be handled by any ORM - in best case you will get [SELECT N+1](http://nhprof.com/Learn/Alerts/SelectNPlusOne) problem. To solve this issue you need to write custom database depend query with WITH statement for Microsoft Sql Server or CONNECT BY PRIOR for Oracle, or even custom stored procedure, use temporary tables and other scary things.

In article “[How to map a tree in NHibernate](http://nhibernate.hibernatingrhinos.com/16/how-to-map-a-tree-in-nhibernate)" [Gabriel Schenker](http://lostechies.com/gabrielschenker/author/gabrielschenker/) offers an alternate approach - adding an helper table to store hierarchy relations. This table will store all ancestors and descendants for each entity. Descendants will be mapped to Descendants collection and ancestors will be mapped to Ancestors collection. Both collections are many-to-many:

```csharp
class Tree {
    int Id;
    Tree Parent;
    IEnumerable<Tree> Children;
    IEnumerable<Tree> Ancestors;
    IEnumerable<Tree> Descendant;
}
```


You can easy handle this structure.

But pros always go with cons. From cons I see that you have to manage this hierarchy table. You can do it from code or from trigger, query or stored procedure in your database.

Happily if you will decide to manage it from code you can write it once and use everywhere. And it is what [I did eventually](https://github.com/BrandyFx/Grapes).

Brandy.Grapes
=============

[Brandy.Grapes](https://github.com/BrandyFx/Grapes) - this is a small set of libraries (only three of them) which allows to easy manage such persisting hierarchy data with [NHibernate](http://nhforge.org/).

You need to install the library through [nuget](http://nuget.org/) (currently NHibernate By Code and FluentNHibernate are supported):

    > install-package Brandy.Grapes.NHibernate

or

    > install-pacakge Brandy.Grapes.FluentNhibernate

Inherit your entity from TreeEntry`1

```csharp
public class MySuperTree : TreeEntry<MySuperTree> {
    public virtual int Id { get; set; }

    public virtual string Name { get; set; }
}
```

At last write a mapping, for example for FluentNHibernate:

```csharp
using Brandy.Grapes.FluentNHibernate;
public class MySuperTreeMap : ClassMap<MySuperTree> {
    public MySuperTreeMap() {
        Id(x => x.Id);
        Map(x => x.Name);

        this.MapTree("MySuperTreeHierarchy"); // all magic goes here
    }
}
```

Enjoy: now Brandy.Grapes will track hierarchy changes and persist it to a database for you.
But when you add anything to the hierarchy the library need to load almost all nodes for the hierarchy from database to correct handle the relations. If your hierarchy is changed frequently you can turn off changes tracking by code and do it in database. There are few ways to do so.

Call the stored procedure by trigger or by code to update hierarchy:

```sql
-- Example for Microsoft Sql Server
CREATE PROCEDURE [dbo].[FillHierarchy] (@table_name nvarchar(MAX), @hierarchy_name nvarchar(MAX))
AS
BEGIN
    DECLARE @sql nvarchar(MAX), @id_column_name nvarchar(MAX)
    SET @id_column_name = ''['' + @table_name + ''_ID]''
    SET @table_name = ''['' + @table_name + '']''
    SET @hierarchy_name = ''['' + @hierarchy_name + '']''

    SET @sql = ''''
    SET @sql = @sql + ''WITH Hierachy(CHILD_ID, PARENT_ID) AS ( ''
    SET @sql = @sql + ''SELECT '' + @id_column_name + '', [PARENT_ID] FROM '' + @table_name + '' e ''
    SET @sql = @sql + ''UNION ALL ''
    SET @sql = @sql + ''SELECT e.'' + @id_column_name + '', e.[PARENT_ID] FROM '' + @table_name + '' e ''
    SET @sql = @sql + ''INNER JOIN Hierachy eh ON e.'' + @id_column_name + '' = eh.[PARENT_ID]) ''
    SET @sql = @sql + ''INSERT INTO '' + @hierarchy_name + '' ([CHILD_ID], [PARENT_ID]) ( ''
    SET @sql = @sql + ''SELECT [CHILD_ID], [PARENT_ID] FROM Hierachy WHERE [PARENT_ID] IS NOT NULL ''
    SET @sql = @sql + '') ''

    EXECUTE (@sql)
END
GO
```

Create a view for each hierarchy (the same query as in stored proceure) and map Ancestors and Descendants collections to that view:

```sql
-- Example for Microsoft Sql Server
CREATE VIEW [MySuperTreeHierarchy]
AS
    WITH Hierachy (CHILD_ID, PARENT_ID) 
    AS 
    (
        SELECT [MySuperTree_ID], [PARENT_ID] FROM [MySuperTree] AS e
        UNION ALL
        SELECT e.[MySuperTree_ID], e.[PARENT_ID] FROM [MySuperTree] AS e 
            INNER JOIN Hierachy AS eh ON e.[MySuperTree_ID] = eh.[PARENT_ID]
    )

    SELECT [CHILD_ID], [PARENT_ID] FROM Hierachy WHERE [PARENT_ID] IS NOT NULL
GO
```

Both approaches much powerful than calling dynamic queries to database on pure SQL from code.

PS: an contract of TreeEntry`1 abstract class:

```csharp
public abstract class TreeEntry<T> where T : TreeEntry {
    public virtual T Parent { get; set; }

    public virtual IEnumerable<T> Children { get; }

    public virtual IEnumerable<T> Ancestors { get; }

    public virtual IEnumerable<T> Descendants { get; }

    public virtual void AddChild(T child);

    public virtual void RemoveChild(T child);
}
```

PPS: Currently EntityFramework does not support hiding collections by IEnumerable`1 interface, so I decided to not provide implementation for EntityFramework.
