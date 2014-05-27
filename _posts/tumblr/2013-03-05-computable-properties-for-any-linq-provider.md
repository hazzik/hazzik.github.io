---
layout: post
title: Computable properties for any LINQ provider
date: '2013-03-05T10:31:14+13:00'
tags:
- NHibernate
- linq
- computed-properties
- decompiler
- delegate
tumblr_url: http://hazzik.tumblr.com/post/44564275956/computable-properties-for-any-linq-provider
---
In this blog post I want to present a little library which I wrote a while ago. This library can decompile methods to their λ-representation.

Why

Sometimes you need to use computable properties in LINQ queries, for example you have Employee class with computable property FullName

class Employee
{
    public string FullName
    {
        get { return FirstName + " " + LastName; }
    }

    public string LastName { get; set; }

    public string FirstName { get; set; }
}




And now your manager comes to you and says that we need to have ability to search by employee’s full name. And you without any thinking write following query:

var employees = (from employee in db.Employees
                 where (employee.FirstName + " " + employee.LastName) == "Test User"
                 select employee).ToList();


Yes, with such simple property as FullName it can work, but what will you do if field would be not simple as this one? For example there is one field from one of my previous projects, and it should be queryable!

class WayPoint 
{
    // all other is skipped due to clarity reasons
    public virtual bool IsValid
    {
        get 
        {
            return (Account == null) ||
               (Role == null || Account.Role == Role) &amp;&amp;
               (StructuralUnit == null || Account.State.StructuralUnit == StructuralUnit);
        }
    }
}


This is much harder…

What do we have to solve such puzzles?

<formula> element in NHibernate

If you are using NHibernate then you can map this field as formula, but this way is full of big scare bears, and such code is hard to maintain, and the <formula> element supports only small subset of SQL, and if you are developing software which have to work with different RDBMS you have to be very-very careful.

It works only in NHibernate

Microsoft.Linq.Translations

With this great and powerful library we would rewrite our class like this

class Employee 
{
    private static readonly CompiledExpression fullNameExpression
        = DefaultTranslationOf.Property(e => e.FullName).Is(e => e.FirstName + " " + e.LastName);

    public string FullName 
    {
        get { return fullNameExpression.Evaluate(this); }
    }

    public string LastName { get; set; }

    public string FirstName { get; set; }
}

var employees = (from employee in db.Employees
                 where employee.FullName == "Test User"
                 select employee).WithTranslations().ToList()


All right, and the query looks good but the field declaration is terrible! And Evaluate compiles the λ-expression in runtime, which is no less terrible than the computable field decalaration.

And now meet my small library - DelegateDecompiler

DelegateDecompiler

The property should be marked with [Computed] attribute and it is all you need, and add .Decompile() method invocation to the query.

class Employee 
{
    [Computed]
    public string FullName 
    {
        get { return FirstName + " " + LastName; }
    }

    public string LastName { get; set; }

    public string FirstName { get; set; }
}

var employees = (from employee in db.Employees
                 where employee.FullName == "Test User"
                 select employee).Decompile().ToList()


In my opinion it is gracefully (If you don’t blow your own horn, no one will do it for you). But bear in mind that the source of your computable property should be supported using LINQ provider.

How does it work?

.Decompile() method finds all the properties and methods marked with [Computed] attribute and expand them to their source expressions. In other words the query above will become as following query:

var employees = (from employee in db.Employees
                 where (employee.FirstName + " " + employee.LastName) == "Test User"
                 select employee).ToList();


The library uses Mono.Reflection (GitHub, NuGet) as decompiler. The Mono.Reflection library is written by Jean-Baptiste Evain the creator of Mono.Cecil.

PS: This is an alpha version, use at your own risk.

Links

Source code at GitHub
Nuget package
