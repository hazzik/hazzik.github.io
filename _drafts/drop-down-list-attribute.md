---
layout: post
title: "Drop Down List Attribute"
date:
tags:
  - ASP.NET MVC
  - DataAnnotations
  - Drop down lists
---
###DropDownListOptions.cs
```csharp
public class DropDownListOptions 
{
    public static string MetadataKey = "DropDownListOptions";
    public string FieldName { get; set; }
}
```
###DropDownListAttribute.cs
```csharp
public class DropDownListAttribute : Attribute, IMetadataAware
{
    public DropDownListAttribute(string fieldName)
    {
        FieldName = fieldName;
    }

    public DropDownListAttribute(string fieldName, string templateHint)
    {
        FieldName = fieldName;
        TemplateHint = templateHint;
    }
 
    public string FieldName { get; set; }
    public string TemplateHint { get; set; }
 
    public void OnMetadataCreated(ModelMetadata metadata)
    {
        metadata.AdditionalValues[DropDownListOptions.MetadataKey] = new DropDownListOptions
        {
            FieldName = FieldName
        };
        metadata.TemplateHint = TemplateHint ?? "DropDownList";
    }
}
```
###Shared/EditorTemplates/DropDownList.cshtml
```csharp
@{
    var data = new RouteValueDictionary(ViewData);
    object o;
    ViewData.ModelMetadata.AdditionalValues.TryGetValue(DropDownListOptions.MetadataKey, out o);
    var options = o as DropDownListOptions;
    @Html.DropDownList("", ViewData[options.FieldName] as IEnumerable<SelectListItem>);
}
```
##Example of Usage
```csharp
public class Model 
{
    [Display(Name = "City"), DropDownList("Cities")]
    public int CityId { get; set; }
}
```