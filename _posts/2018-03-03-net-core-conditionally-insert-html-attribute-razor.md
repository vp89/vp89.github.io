---
layout: post
title:  ".NET Core conditionally insert HTML attribute in a Razor view"
date:   2018-03-03 00:00:00 -0500
categories: programming
---

For my first post of the year, I would like to share an example of how to create a Razor tag helper that lets you
conditionally append an attribute to a HTML element.

I couldn't find any examples of this that worked with the new .NET Core hotness so I pieced one together using
various sources.


```
public static HtmlString ConditionalAttribute(this IHtmlHelper helper, string attributeName, string attributeValue, bool condition)
{
    if (string.IsNullOrEmpty(attributeName) || string.IsNullOrEmpty(attributeValue))
    {
        return HtmlString.Empty;
    }

    return condition ? new HtmlString($"{attributeName}=\"{attributeValue}\"") : HtmlString.Empty;
}
```

In this example we are using Bootstrap and we want to set our table row to have the class "success" if SomeValue 
in our Model is true. We can do this in our Razor views like this:

```
<tr @Html.ConditionalAttribute("class", "success", Model.SomeValue)>
```

This will result in the following HTML on the page:

```
<tr class="success">
```

I like to wrap components like this in a static class called ViewHelpers so that I don't have them scattered all over
my project.

There are ways you could beef this component up. For example, you could have it receive a `Func<bool>` instead of a plain bool. This could help reduce lines of code in your view if you're evaluating a bunch of values in your view 
in a repeatable and non-trivial way.

You could also have it take a `Dictionary<string, string>` if you want to be able to easily inject multiple attributes
based on the same condition.