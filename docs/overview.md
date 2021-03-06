[<< Back to start](../README.md)

# The overview of Razor-Express template engine (RAZ)

- [**Views and View Template Engine**](#views-and-view-template-engine)
- [**Rendering layout system**](#rendering-layout-system)
  - [Processing a view](#processing-a-view)
  - [Layouts](#layouts)
    - [Access data from a layout](#access-data-from-a-layout)
  - [Partial views](#partial-views)
    - [Access data from partial views](#access-data-from-partial-views)
  - [Partial view search algorithm](#partial-view-search-algorithm)
  - [Starting views](#starting-views-_viewstartraz)
  - [The order of processing views](#the-order-of-processing-views)
  - [A page components illustration examples](#a-page-components-illustration-examples)

## Views and View Template Engine
Most likely you know that the simplest [NodeJS](https://nodejs.org/) web server built with [Express library](https://expressjs.com/) can work without any template engine. Express library can just [serve static files](https://expressjs.com/en/starter/static-files.html) in response to a browser request. It can be any staic file including a file with HTML markup (which is essentially a regular text file). Although this method is still quite often used for simple small websites, it contains a number of disadvantages and is not suitable for more complex websites.

The main disadvantage is that your HTML file stores not only the structure of the document but also some data. When the data is mixed with HTML code you can't easily edit the data if you are not familiar with HTML and if you need to edit HTML you have to wade through tons of text that have nothing to do with HTML itself. This is where the idea of separating *markup* and *data* comes in. 

With this idea of separation, such terms as *"data"* and *"view"* are usually involved. Also, there are another two terms like *"Model-View"* or just *"Model"* which quite often mean the same thing and typically used in the context of [MVC pattern](https://docs.microsoft.com/en-us/aspnet/core/mvc/overview).

So, when the data is stored separately you need some mechanism that can correctly place it in HTML markup. To do this the HTML file must contain special fields or placeholders that have to be replaced with the appropriate data chunks. This file is usually called a *"view template"*, *"template"*, or just a *"view"*. And the mechanism which looks for the placeholders and fill them out with data chunks is usually called  *"View Template Engine"*. Different engines [have their differences](https://github.com/DevelAx/RazorExpress/blob/master/README.md#a-brief-comparison-of-syntax-of-nodejs-template-engines) but the basic idea is the same. 

Razor-Express engine is one of [many](https://github.com/expressjs/express/wiki#template-engines) working with Express. Razor-Express uses [Razor-like syntax](https://github.com/DevelAx/RazorExpress/blob/master/docs/syntax.md) based on the [ASP.NET MVC Razor concept](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/razor) which allows you to template views by mixing HTML markup with real server-side JavaScript code.

## Rendering layout system
When you have a *NodeJS Express web app* set up ([Express web-server example](https://github.com/DevelAx/RazorExpress/blob/master/README.md#express-web-server-example)) and run the *Express framework* starts to use the *Razor-Express engine* as a service to read *view template* files and process them into HTML. This process includes the following steps:

1. Express app receives a request from the browser, analyzes its URL and finds an appropriate route which determines a file to return in response to the browser request. 
2. Express makes sure that the file actually exists and then transfers control to the Razor-Express engine. Also Express may pass some *data (or model)* as a parameter to the engine to render it with the *view template* file content.
3. Razor-Express reads the *view template* file, runs server-side JavaScript incorporated in it, replaces placeholders with data from *model*, and finally renders HTML which is then returned back to Express.
4. Having control back the Express library sends that HTML to the browser.

It is a quite simplified description of the request handling to understand the role of the Razor-Express engine in this process. Now let's take a closer look at what happens in the Razor-Express engine while processing a *view template* and *data model*.

### Processing a view
> By default Razor-Experss uses `'raz'` extension for its view template files but you can use any other you'd like, for example `'html'` by passing it explicitly either via RazorExpress [`register`](api.md#registerapp-ext) method.

When Razor-Express gets control, it also gets the full filename of a *view template* file and *data model* both passed as parameters. The *data model* is optional though. If the *template* is successfully processed the engine returns HTML. If a failure occurs while reading, parsing, or rendering the file RAZ returns an error. In any case at this point, its work is done.  

After the file is found and read, the engine tries to find all [starting view files](#starting-views-_viewstartraz) named [*"_viewStart.raz"*](#starting-views-_viewstartraz) following the [partial views standard search algorithm logic](#partial-view-search-algorithm). If they are found they are added to the current file from its beginning in the order the search sequence goes (each next found is added to the very beginning of the current file and so on).

When this process is finished the parser starts analyzing that assembled template (if no *"_viewStart.raz"* files are found it contains only the main view template). 

> It's worth noting that the *parser doesn't try to fully analyze the validity of HTML* in a template. For example, it is not much concerned about mistakes in the attributes of HTML tags. *Neither it checks the correctness of JavaScript syntax*, except for the base constructs. It only checks the integrity of the HTML tags tree and extracts snippets of server-side JavaScript code to perform it in the next step. 

After parsing is done the execution process begins. At this point, the template placeholders are substituted with the appropriate values from the *data model* and all the server-side JavaScript code found in this template is executed. In this process, the references to other view templates could be found. If so, each referenced template file is read and processed the same way as the main view (with which the engine has started) with the exception that the *"_viewStart.raz"* files are not considered anymore. *Each referenced template file is processed separately from the main one and from the others.* This means that if you declare a variable in one template it won't be available in any referenced template because each processed file is run in its own scope (and in its own moment). If you need to share some data between those views it is possible to do via the `Model` and `ViewData` objects as will be discussed [later](#access-data-from-a-layout). However, the *data model* (represented by the `Model` object in views) is the same for all the view (unless it's explicitly set otherwise).

There are two types of view templates, which can be explicitly referenced from the rendering page template:
* [Layout](#layouts)
* [Partial view](#partial-views)

and one type that is referenced implicitly from any page view:
* [Starting view](#starting-views-_viewstartraz)


### Layouts
*Layout* is just a base markup for a group of website pages that have some common elements, such as header, footer, menu, as well as other structures such as scripts, stylesheets, etc. The use of layouts helps to reduce code duplication in views. From the Razor-Express engine's perspective, a layout is just a *normal view template* with the only difference being that the layout defines a top-level template for the other views.  Using the layout is optional. Apps can define more than one layout, with different views specifying different layouts. A layout can have a reference to another layout and so forth, which means that the layouts can be nested (an example can be found in [this repository](https://github.com/DevelAx/RazorExpressFullExample)). Layouts can have references to partial views as well.

Conventionally the default layout is named *"_layout.raz"*. To specify a layout for a view an `Html.layout` property must be set in that view (or in one of its [*"_viewStart.raz"*](#starting-views-_viewstartraz)):
```HTML+Razor
@{
    Html.layout = "_layout";
}
```
You can use either a partial name like in the example above or a full path (relative to the *"views"* folder):
```HTML+Razor
@{
    Html.layout = "/admin/_layout";
}
```
or relative to the current view directory path:
```HTML+Razor
@{
    Html.layout = "./_layout";
}
```
The layout file extension is optional in all these cases. When only the partial name is provided, the Razor-Express view engine searches for the layout file using its standard for [partial views discovery process](#partial-view-search-algorithm).

Each layout is supposed to call the `@Html.body()` method within itself where the contents of the current view have to be rendered. (As you already know, the page template is processed before the layout template, so this call will renderer an already compiled page HTML.) 

#### Access data from a layout
A layout has access to the current view `Model` object. It is passed implicitly and there is no way to pass any other object as `Model`. One possible way to transfer some data to the layout other than the view `Model` is to use the `ViewData` object. 

### Partial views
The term *"partial view"* clearly implies that HTML received as a result of processing a partial view template will become a part of another view which references it. Partial view can reference other partial views, but it _can't reference a layout_.

Partial views are supposed to be a means to reuse the same code snippets from different views thus avoiding duplication in them. Also, there is another advantage where large, complex markup file can be split into several logical pieces and each one can be isolated within a partial view for easier perception while working with them separately.

By convention, partial view file names begin with an underscore (`_`). It's not strictly required, although it helps to visually differentiate them from the page views.

To reference a partial view from any view use the `Html.partial` method:
```HTML+Razor
@Html.partial("_partial")
```
This method initiates [search](#partial-view-search-algorithm) the `_partial.raz` view file, compiling it into HTML, and then rendering this HTML in the parent view where this method is placed. The ways to specify the file path for the partial views are exactly the same as for [layouts](#layouts).

> :warning: It's worth emphasizing once again that *only the page main view template is processed together with the starting templates* as one whole one (as they are joined before being parsed and executed). All other *view templates* are parsed and executed separately, and only then included in each other in the form of ready-made HTML.

#### Access data from partial views
The `Html.partial` method can also take a second parameter through which you can pass a custom model. 
```HTML+Razor
@Html.partial("_partial", { text: "This is an explicit model passing to the partial view." })
```
If this argument is omitted the `Model` of the parent view is passed by default implicitly. Also partial views have access to the `ViewData` object as every view template does.

### Partial view search algorithm

If a partial view is specified *only by a file name* with or without an extension (as in the [Partial views section](#partial-views) example above) then the search begins with the directory in which the view that initiates the search is located. If the partial view is not found in the current directory the search goes on up the directory tree until it reaches the root views folder specified in the [Express app](https://expressjs.com/en/guide/using-template-engines.html) (which is set as `app.set('views', './views')` by default). If the file is still not found an error is returned.

To specify a partial view by a path relative to the *root views directory* use slash `'/'` as the first character of the path. For example, with the path `"/partial-views/_login"` the partial view `"_login.raz"`  file will be searched only in the `"[app-folder]/views/partial-views/"` directory, where `"[app-folder]"` is the full path to your NodeJS application script directory. 
**_Never include the views root folder name_** in the relative paths - it is added automatically.

To set a path *relative to the current view directory*, use the relative to the current directory path formats, examples: `'./partial-views/_login'` or `'../partial-views/_login'`.

Different partial views with the same file name are allowed when the partial views are in different folders.

Partial views can be chained — a partial view can call another partial view and so on (be careful not to create circular references).

### Starting views (`_viewStart.raz`)
Starting views named *"_viewStart.raz"* are intended to contain code that needs to run before the code of the main view of the page is executed. Starting views are hierarchical -  if a `_viewStart.raz` file is defined in the current view folder, it will be run after the one defined in the root views folder (if any).

Usually, the `_viewStart.raz` file is used to specify a [layout](#layouts) for a group of views (located in a specific folder or several folders). For example, you can define a `_viewStart.raz` file with the next code in the folder with other views instead of adding this code in the beginning of each of these views.
```HTML+Razor
@{
    Html.layout = "_layout";
}
```

### The order of processing views

*The order in which views are processed is important to remember* in case you decide to change some data in the model, for example, in one view and then use it in another. 
- The **current view template with all the [*"_viewStart.raz"*](#starting-views-_viewstartraz) views are processed first**, as already mentioned. 
- **Then all the partial views are processed** in the order they are referenced and **all [sections](https://github.com/DevelAx/RazorExpress/blob/master/docs/syntax.md#section) are rendered**. 
- **The last step is to find and render layouts** (with partial views and sections they have references to). Actually, it is not different from *ASP.NET MVC Razor* algorithm.

![The order of processing views](https://github.com/DevelAx/RazDoc/blob/master/Razor-Express-view-processing-flow.png)

### A page components illustration examples

#### Page design layout
![Page layout example](https://github.com/DevelAx/RazDoc/blob/master/PageLayoutExample.png?raw=true)

#### Razor-Express layout components
![Layout elements example](https://github.com/DevelAx/RazDoc/blob/master/PartsOfLayoutExample.png?raw=true)
