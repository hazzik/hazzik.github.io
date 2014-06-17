---
layout: post
title: "Render Action Attribute"
date: 2014-06-17T11:43:03+12:00
tags:
  - ASP.NET MVC
  - DataAnnotations
  - RenderAction
---
###RenderActionOptions.cs
```csharp
public class RenderActionOptions 
{
    public static string MetadataKey = "RenderActionOptions";

    public string Action { get; set; }
    public string Controller { get; set; }
    public string Area { get; set; }
}
```
###RenderActionAttribute.cs
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
###Shared/EditorTemplates/RenderAction.cshtml
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
```csharp
public class Model 
{
    [Display(Name = "City"), RenderAction("ListCities", "Cities")]
    public int CityId { get; set; }
}
```