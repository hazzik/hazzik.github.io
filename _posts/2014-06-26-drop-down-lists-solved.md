---
layout: post
title: Solving DropDown list in ASP.NET MVC with MvcExtensions
date: 2014-06-26T17:23:16+12:00
tags:
- MvcExtensions
- ASP.NET MVC
- Drop down lists
---
This is an translation of [my article](http://blog.hazzik.ru/2012/04/02/mvcextensions-solving-drop-down-lists-problem/) from 2nd April of 2012.

From the first version of [ASP.NET MVC](http://www.asp.net/mvc) couple of years ago I found that there is no common solution how to display drop down lists properly. Probably every of you [asked yourself](http://www.google.ru/webhp#q=asp.net+mvc+dropdownlist): &#8220;How to pass the selection list to a drop down list?&#8221; And so I was asking the same question to myself, I could not even sleep;)

<!-- more -->

Introduction

For example we have an form to create a film, and we need to select film's genre from a drop down list. I'm not going to show any "newbuy" solutions as, for instance, getting the genres in the view file.

## "Head on" solution

The programmers bumping that problem try to solve it by brute force: they create additional property "Genres" of type <em>SelectList</em> at the model class, and they fill it at the controller's action. Like this:


###The model

```csharp
public class Movie {
    public int GenreId { get; set; }
    public SelectList Genres { get; set; }    
    //...
} 
```

###The controller

```csharp
public class MoviesController {
    [HttpGet] public ActionResult Create() {
        var model = new Movie() { Genres = GetAllGenresFromDatabase(); }
        return View(model); 
    }

    [HttpPost] public ActionResult Create(Movie form) {
        // do something with movie.
    }

    [HttpGet] public ActionResult Edit(int id) {
        var model = GetMovieFromDatabase();
        model.Genres = GetAllGenresFromDatabase(); 
        return View(model); 
    }

    [HttpPost] public ActionResult Edit(EditMovie form) {
        // do something with movie.
    }
    //...
}
```

This works so far, but the solution has some major problems

1.  Model has the redundant field
2.  You need to create a model for a creation form
3.  Code duplication of the list filling logic. <small>This problem grow if you need to select form the same list in many places.</small>
4.  You can not use `Html.EditorForModel()`, as you do not have access to neighbour fields when ASP.NET renders the model

## Solution using ViewBag / ViewData

The solution is similar to the previous one with one simple change: selection list is passing through ViewBag / ViewData

###The model

```csharp
public class Movie {
    [UIHint("Genres")]
    public int GenreId { get; set; }
    //...
}
```

###The controller
```csharp
public class MoviesController {
    [HttpGet] public ActionResult Create() {
        ViewBag.Genres = GetAllGenresFromDatabase();
        return View(); 
    }

    [HttpPost] public ActionResult Create(Movie form) {
        // do something with movie.
    }

    [HttpGet] public ActionResult Edit(int id) {
        var model = GetMovieFromDatabase();
        ViewBag.Genres = GetAllGenresFromDatabase(); 
        return View(model); 
    }

    [HttpPost] public ActionResult Edit(EditMovie form) {
        // do something with movie.
    }
    //...
}
```

Pros over the previous solution

1.  The model does not have redundant fields anymore
2.  You do not need to create the model in the creation action
3.  You can use `Html.EditorForModel()` with own Editor Template for every drop-down list field

Cons

1.  The dynamic or magic-strings* are used, which complicates refactoring and code modification
2.  You still have code duplication of ddl fill logic
3.  You need to have an editor template for every field

### Improved solution with ViewBag / ViewData

To reduce code duplication of filling logic we move the code to action filter

###The model

The same as in the previous example

###ActionFilter
```csharp
public class PopulateGenresAttribute: ActionFilterAttribute {
    public override void OnActionExecuted(ActionExecutedContext filterContext) {
        filterContext.Controller.ViewData["Genres"] = GetAllGenresFromDatabase();
    }
    //...
}
```

###The controller

```csharp
public class MoviesController {
    [HttpGet, PopulateGenres] public ActionResult Create() {
        return View(); 
    }

    [HttpPost] public ActionResult Create(Movie form) {
        // do something with movie.
    }

    [HttpGet, PopulateGenres] public ActionResult Edit(int id) {
        var model = GetMovieFromDatabase();
        return View(model); 
    }

    [HttpPost] public ActionResult Edit(EditMovie form) {
        // do something with movie.
    }
    //...
}
```

Pros over the previous solution

1.  We do not have code duplication of selection list filling logic

Cons

1.  The dynamic or magic-strings* are used, which complicates refactoring and code modification
2.  You need to have an editor template for every field

### Imporved solution with ViewBag / ViewData + [MvcExtnsions](http://mvcextensions.github.io)

MvcExtensions have beautiful [methdos to work with drop down lists](http://github.com/MvcExtensions/Core/blob/master/src/MvcExtensions/ModelMetadata/HtmlSelectModelMetadataItemBuilderExtensions.cs): AsDropDownList / AsListBox (the first one for drop down list and the second one for multi-select lists). These are extension methods for model metadata builder. These methods set the template and allow to pass the ViewData's field name which stores the selection list to the View. So we are solving the need of having an template per field.

###The model

```csharp
public class Movie {
    public int GenreId { get; set; }
} 
```

###The metadata

```csharp
public class MovieMetadata : ModelMetadataConfiguration {
    public MovieMetadata {
        Configure(movie => movie.GenreId).AsDropDownList("Genres"/*шаблон*/);
    }
}
```

###The controller

The same as in the previous example


Pros over the previous solution

1.  You use two generic templates (DropDownList / ListBox) for all of your lists, but you still have an ability to set your own template if you need so

Cons

1.  The dynamic or magic-strings* are used, which complicates refactoring and code modification

## The solution with ChildAction

If you'll try to use just a child action, then it'll not work at all: you will not have client-side validation from the box, you will not be able to use nested forms, etc. <!-- В [статье](http://pashapash.com/2010/12/dropdown-v-asp-net-mvc-chast-1/) ([часть 2](http://pashapash.com/2011/01/dropdown-v-asp-net-mvc-chast-2/)) неизвестного автора (быстрый поиск выдал только [профиль](http://habrahabr.ru/users/PashaPash/) на хабре) решены эти проблемы, и по-этому я буду рассматривать _окончательное решение автора_. -->

###The model

```csharp
public class Movie {
    [UIHint("Genres")]
    public int GenreId { get; set; }
} 
```

###The controllers

```csharp
public class MoviesController {
    [HttpGet] public ActionResult Create() {
        return View(); 
    }

    [HttpPost] public ActionResult Create(Movie form) {
        // do something with movie.
    }

    [HttpGet] public ActionResult Edit(int id) {
        var model = GetMovieFromDatabase();
        return View(model); 
    }

    [HttpPost] public ActionResult Edit(EditMovie form) {
        // do something with movie.
    }
}
```
```csharp
public class GenresController {
    public ActionResult List() {
        var selectedGenreId = this.ControllerContext.ParentActionViewContext.ViewData.Model as int?;

        var genres = GetGenresFormDatabase();

        var model = new SelectList(genres, "Id", "DisplayName", selectedGenreId);

        this.ViewData.Model = model;
        this.ViewData.ModelMetadata = this.ControllerContext.ParentActionViewContext.ViewData.ModelMetadata;

        return View("DropDown");
    }
}
```

Pros over the previous solution

1.  dynamic and magic-strings are not used anymore
2.  We do not have code duplication of selection list filling logic

Cons

1.  Duplication of the boilerplate code
2.  Need to have an template for each of your selection lists
3.  The Post-Redirect-Get scenario is not supported

### Solution using ChildAction + MvcExtensions

I decided to imporve the previouse solution and apply the ActionFilter experience, and now, from the version [2.5.0-rc8000](http://nuget.org/packages/MvcExtensions) MvcExtension do support child-action based drop down lists out of the box. I've added extension methods, which allows to use <em>RenderAction</em> to render the model fields. Also `SelectListActionAttribute` attibute were added, which serves the action providing selection list data. Also this solution supports Post-Redirect-Get scenario.

###The model

```csharp
public class Movie {
    public int GenreId { get; set; }
}
```

###The metadata

```csharp
public class MovieMetadata : ModelMetadataConfiguration {
    public MovieMetadata {
        Configure(movie => movie.GenreId).RenderAction("List", "Genres");
    }
}
```

###The controllers

```csharp
public class MoviesController {
    [HttpGet] public ActionResult Create() {
        return View(); 
    }

    [HttpPost] public ActionResult Create(Movie form) {
        // do something with movie.
    }

    [HttpGet] public ActionResult Edit(int id) {
        var model = GetMovieFromDatabase();
        return View(model); 
    }

    [HttpPost] public ActionResult Edit(EditMovie form) {
        // do something with movie.
    }
}
```

```csharp
public class GenresController {
    [ChildActionOnly, SelectListAction] public ActionResult List(int selected) {
        var model = GetGenresFormDatabase(selected);
        return View("DropDown", model);
    }
}
```

Pros over the previous solution

1.  No boilerplate code duplication
2.  Use generic template
3.  MultiSelect "out of the box"
4.  Post-Redirect-Get is supported

# Ending

For me, as a developer of MvcExtensions the methdos using the library is preferrable.

The sample of code with ViewBag / ViewData + MvcExtensions is here: [http://github.com/MvcExtensions/Core/tree/master/samples](http://github.com/MvcExtensions/Core/tree/master/samples)

The sample of code with ChildAction + MvcExtensions is here: [http://github.com/hazzik/DropDowns](http://github.com/hazzik/DropDowns)

* * *

*magic-strings can be easily solved by using of constans, and so because of that, I prefere using string over dynamic properties.

PS: The MvcExtensions abilities to extend ASP.NET MVC are limitless
