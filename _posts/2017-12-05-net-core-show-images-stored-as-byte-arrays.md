---
layout: post
title:  ".NET Core display image byte array in a view"
date:   2017-12-05 19:00:00 -0500
categories: programming
---

If you're storing your images in a database or you want to display a generated image on a page, you will need to be
able to pass a byte array to your view and then know how to use that inside the img tag.

Here is a simple example of a Razor view, it's model is just a byte array to keep the example code simple.

{% raw %}
```
@model byte[]

@if (Model != null && Model.Length > 0)
{
    string imageSource = $"data:image;base64,{Convert.ToBase64String(Model)}";

    <img src="@imageSource" />
}
```
{% endraw %}

There are pros and cons to embedding your images inside data URIs. One of the pros is that you reduce the amount of requests each page
hit has to make to render all the content. A downside is that the content can't be cached by the browser.

With regards to the mime type, you can provide a more specific one such as image/gif, image/jpg or image/png if your use case calls for it,
but browsers don't require you to specify the exact format of your image so I omitted it from the example.
