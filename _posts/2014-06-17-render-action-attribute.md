---
layout: post
title: "Render Action Attribute"
date: 2014-06-17T11:43:03+12:00
tags:
  - ASP.NET MVC
  - DataAnnotations
  - RenderAction
---
The purpose of this attribute is to be able to render action as an editor or display template for your form fields. First of all we will need the data structure which will hold options for the our editor template.

####RenderActionOptions.cs
```csharp
public class RenderActionOptions 
{
    public static string MetadataKey = "RenderActionOptions";

    public string Action { get; set; }
    public string Controller { get; set; }
    public string Area { get; set; }
}
```

Then we can use custom attribute, which implements [IMetadataAware](http://msdn.microsoft.com/en-us/library/system.web.mvc.imetadataaware) interface, to fill the options and pass them to a metadata.

####RenderActionAttribute.cs
```csharp
public class RenderActionAttribute : Attribute, IMetadataAware
{
    public RenderActionAttribute([AspMvcAction] string action)
    {
        Action = action;
    }
 
    public RenderActionAttribute([AspMvcAction] string action, [AspMvcController] string controller)
    {
        Action = action;
        Controller = controller;
    }
 
    public RenderActionAttribute([AspMvcAction] string action, [AspMvcController] string controller, [AspMvcController] string area)
    {
        Action = action;
        Controller = controller;
        Area = area;
    }
 
    public string Action { get; set; }
    public string Controller { get; set; }
    public string Area { get; set; }
 
    public void OnMetadataCreated(ModelMetadata metadata)
    {
        metadata.AdditionalValues[RenderActionOptions.MetadataKey] = new RenderActionOptions
        {
            Action = Action,
            Controller = Controller,
            Area = Area
        };
        metadata.TemplateHint = "RenderAction";
    }
}
```

And finally at our EditorTemplate we will retrieve the options from the metadata and use `Html.RenderAction` helper method to render our action. Please note, that `Html.RenderAction` nor `Html.Action` do not accept "area" as an argument, so we need to add the "area" key to our routes dictionary. Also, we pass `data` to the `Html.RenderAction` method to be able supply some additional information/metadata to the rendered action.

####Shared/EditorTemplates/RenderAction.cshtml
```csharp
@{
    var data = new RouteValueDictionary(ViewData);
    object o;
    ViewData.ModelMetadata.AdditionalValues.TryGetValue(RenderActionOptions.MetadataKey, out o);
    var options = o as RenderActionOptions;
    if (!string.IsNullOrEmpty(options.Area))
    {
        data["area"] = options.Area;
    }
    if (!string.IsNullOrEmpty(options.Controller))
    {
        Html.RenderAction(options.Action, options.Controller, data);
    }
    else
    {
        Html.RenderAction(options.Action, data);
    }
}
```
##Example of Usage
####MyModel.cs
```csharp
public class MyModel 
{
    [Display(Name = "City"), RenderAction("ListCities", "Cities")]
    public int CityId { get; set; }
}
```
####MyForm.cshtml
```csharp
@model MyModel
// ...
@Html.EditorFor(m => m.CityId, new { htmlAttributes = new { @class = "my-editor" } } )
// ...
```
