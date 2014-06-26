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


####The model

```csharp
public class Movie {
    public int GenreId { get; set; }
    public SelectList Genres { get; set; }    
    //...
} 
```

####The controller

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

1.  Model has redundant fields
2.  Need to create a model for a creation form (when there is no entity in DB yet)
3.  Code duplication for select list options retrieval logic. <small>This problem grows if you need to select form the same list in many places.</small>
4.  Not possible to use `Html.EditorForModel()` to display all you form at once with auto-generated markup, as we do not have access to neighbour fields when ASP.NET renders the model

## Solution using ViewBag / ViewData

The solution is similar to the previous one with one simple change: selection list is passing through ViewBag / ViewData

####The model

```csharp
public class Movie {
    [UIHint("Genres")]
    public int GenreId { get; set; }
    //...
}
```

####The controller
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

###Pros over the previous solution

1.  The model does not have redundant fields
2.  Not needed to create a model for a creation form (when there is no entity in DB yet)
3.  Possible to use `Html.EditorForModel()` with own Editor Template for every drop-down list field

###Cons

1.  Using of the dynamics or magic-strings*, which complicates refactoring and code modification
2.  Code duplication for select list options retrieval logic
3.  Custom editor template per field

## Improved solution with ViewBag / ViewData

To improve the solution and reduce code duplication of retrieving of selection options logic we move the code from the controller action to an action filter

####The model

The same as in the previous example

####ActionFilter
```csharp
public class PopulateGenresAttribute: ActionFilterAttribute {
    public override void OnActionExecuted(ActionExecutedContext filterContext) {
        filterContext.Controller.ViewData["Genres"] = GetAllGenresFromDatabase();
    }
    //...
}
```

####The controller

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

###Pros over the previous solution

1.  No code duplication for select list options retrieval logic

###Cons

1.  Using of the dynamics or magic-strings*, which complicates refactoring and code modification
2.  Custom editor template per field
3.  Application specific logic in the filter, which smells

## Imporved solution with ViewBag / ViewData + [MvcExtnsions](http://mvcextensions.github.io)

MvcExtensions have beautiful [methdos to work with drop down lists](http://github.com/MvcExtensions/Core/blob/master/src/MvcExtensions/ModelMetadata/HtmlSelectModelMetadataItemBuilderExtensions.cs): AsDropDownList / AsListBox (the first one for drop down list and the second one for multi-select lists). These are extension methods for model metadata builder. These methods set the template and allow to pass the ViewData's field name which stores the selection list to the View. So we are solving the need of having an template per field.

####The model

```csharp
public class Movie {
    public int GenreId { get; set; }
} 
```

####The metadata

```csharp
public class MovieMetadata : ModelMetadataConfiguration {
    public MovieMetadata {
        Configure(movie => movie.GenreId).AsDropDownList("Genres"/*шаблон*/);
    }
}
```

####The controller

The same as in the previous example

###Pros over the previous solution

1.  Using of two generic templates (DropDownList / ListBox) for all lists, or specific one if needed

###Cons

1.  Using of the dynamics or magic-strings*, which complicates refactoring and code modification

## The solution with ChildAction

This is a good idea to put logic of the selection list to a separate action. This will lead to better separation of concern and give you some benifits like caching. But yf you'll try to use just a child action, then it'll not work at all: you will not have client-side validation from the box, you will not be able to use nested forms, etc., because the field names will be broken. To make it work properly you need to set view data's model metadata to the same as in parent action.

####The model

```csharp
public class Movie {
    [UIHint("Genres")]
    public int GenreId { get; set; }
} 
```

####The controllers

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

###Pros over the previous solution

1.  No using of dynamics and magic-strings
2.  No code duplication of select list options retrieval logic
3.  No application specific logic in ActionFilter

###Cons

1.  Duplication of the boilerplate code
2.  Custom template per select list
3.  The Post-Redirect-Get scenario is not supported

### Solution using ChildAction + MvcExtensions

I decided to imporve the previouse solution and apply the ActionFilter experience, and now, from the version [2.5.0-rc8000](http://nuget.org/packages/MvcExtensions) MvcExtension do support child-action based drop down lists out of the box. I've added extension methods, which allows to use <em>RenderAction</em> to render the model fields. Also `SelectListActionAttribute` attibute were added, which serves the action providing selection list data. Also this solution supports Post-Redirect-Get scenario.

####The model

```csharp
public class Movie {
    public int GenreId { get; set; }
}
```

####The metadata

```csharp
public class MovieMetadata : ModelMetadataConfiguration {
    public MovieMetadata {
        Configure(movie => movie.GenreId).RenderAction("List", "Genres");
    }
}
```

####The controllers

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

###Pros over the previous solution

1. No boilerplate code duplication
2. Use an generic template
3. MultiSelect "out of the box"
4. Post-Redirect-Get is supported

# Ending

For me, as a developer of MvcExtensions the methdos using the library is preferrable.

The sample of code with ViewBag / ViewData + MvcExtensions is here: [http://github.com/MvcExtensions/Core/tree/master/samples](http://github.com/MvcExtensions/Core/tree/master/samples)

The sample of code with ChildAction + MvcExtensions is here: [http://github.com/hazzik/DropDowns](http://github.com/hazzik/DropDowns)

* * *

*magic-strings can be easily solved by using of constans, and so because of that, I prefere using string over dynamic properties.

PS: The MvcExtensions abilities to extend ASP.NET MVC are limitless
