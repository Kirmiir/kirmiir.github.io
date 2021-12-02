---
layout: post
title: ASP.NET Core. Store css in Database and combine it in Bundle. Part 1
---

There are a lot of different ways how to include js and CSS files into the website. It can be done by using bundling that will organize and put all your files in little files. It can be done just by adding the js files directly to the web folder. But sometimes the files that need to be included can be different due to different environments or different settings.

In this case, one of the possible solutions is to try to store these files in the Database. In this post, I will show one of the ways how to achieve that. In the beginning, we need to create a simple page with the following content:

{% highlight html %}
<div class="text-center">
    <h1 class="display-4">Database Css</h1>
    <div class="example a"></div>
    <div class="example b"></div>
    <div class="example c"></div>
</div>
{% endhighlight %}

The next step is updating site.css to define the default size of div.

{% highlight html %}
div.example{
    width: 100px;
    height: 100px;
    background-color: darkgray;
}
{% endhighlight %}

To create a bundle we need to add the NuGet package for this. In this example, we will use LigerShark.WebOptimizer.Core so we need to install it and enable it.

{% highlight csharp %}
dotnet add package LigerShark.WebOptimizer.Core
{% endhighlight %}

In Startup.cs we need to add code to add Web Optimizer.

{% highlight csharp %}
services.AddWebOptimizer(pipeline =>
{
	pipeline.AddCssBundle("/css/static.css", "/css/site.css");
});
{% endhighlight %}

For now, if we open this site on browser we will see this:
![default]({{ site.baseurl }}/images/2021-11-29-1/web-default.jpg)

## Create a Database infrastructure to store CSS in a Database.
For now, we will use EntityFramework. The following packages need to be installed to work with SQL Database:

{% highlight csharp %}
dotnet add package LigerShark.WebOptimizer.Core
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design

dotnet tool install --global dotnet-ef
{% endhighlight %}

Let’s create a Simple DatabaseContext to work with a database.

{% highlight csharp %}
public class DataBaseContext : DbContext
{
    public DbSet<CssFile> CssFiles { get; set; }

    public DataBaseContext(DbContextOptions<DataBaseContext> options)
        : base(options)
    {
    }
}
{% endhighlight %}

The DataBaseContext is a simple context that we use to work with DB. The CssFile is a class that represents the CSS. There are two columns Css and LastUpdateDate

{% highlight csharp %}
public class CssFile
{
    public string Css { get; set; }
    public DateTimeOffset LastUpdateDate { get; set; }
}
{% endhighlight %}

Create migration:

{% highlight csharp %}
 dotnet ef migrations add InitialCreate
{% endhighlight %}

## Load CSS into the application.
The next step is to add a way to load CSS from a database into the bundle. Let start with creating a simple custom IFileProvider.
{% highlight csharp %}
public class DatabaseFileProvider : IFileProvider
{
	private readonly DataBaseContext _context;
	public DatabaseFileProvider(DataBaseContext context)
	{
	    _context = context;
	}

	public IDirectoryContents GetDirectoryContents(string subpath)
	{
	    return new NotFoundDirectoryContents();
	}

	public IFileInfo GetFileInfo(string subpath)
	{
	    if (!subpath.StartsWith("/database/")) return new DataBaseFileInfo(null);
	    
	    var file = _context.CssFiles.Single(file => file.Name.Equals(subpath));
	    return new DataBaseFileInfo(file);

	}

	public IChangeToken Watch(string filter)
	{
	    return new CancellationChangeToken(new CancellationToken());
	}
}
{% endhighlight %}

This is a simple class and for now, we implement public IFileInfo GetFileInfo(string subpath) method. The main idea is the return IFileInfo for a file if the path has a prefix “/database/”. In this case, we need to load the entry from the database and return it.

The implementation of DataBaseFileInfo is also quite simple for now.
{% highlight csharp %}
private class DataBaseFileInfo : IFileInfo
{
    private readonly string _content;
    public DataBaseFileInfo(CssFile file)
    {
        if (file != null){
            Name = file.Name;
            PhysicalPath = file.Name;
            LastModified = file.LastUpdateDate;
            Length = Encoding.GetEncoding("UTF-8").GetByteCount(file.Css);
            Exists = true;
            _content = file.Css;
        }
    }
    public Stream CreateReadStream()
    {
        return new MemoryStream(Encoding.GetEncoding("UTF-8").GetBytes(_content ?? ""));
    }

    public bool Exists { get; }

    public bool IsDirectory => false;
    public DateTimeOffset LastModified { get; }
    public long Length { get; }
    public string Name { get; }
    public string PhysicalPath { get; }
}
{% endhighlight %}

The most important part here is the method CreateReadStream. This method is used to create a bundle.

The last thing before seeing the result is enabling this provider in asp net core and inserting the CSS into a database. To do this, we need to change the code in startup.cs

{% highlight csharp %}
var tempServiceProvider = services.BuildServiceProvider();
    services.AddWebOptimizer(pipeline =>
    {
        var provider = ActivatorUtilities.CreateInstance<DatabaseFileProvider>(tempServiceProvider);
        pipeline.AddCssBundle("/css/bundle.css", "/database/a.css", "database/b.css").UseFileProvider(provider);
        pipeline.AddCssBundle("/css/static.css", "/css/site.css");
    });
{% endhighlight %}

The insert in a database is a simple SQL query.

{% highlight sql %}
INSERT INTO CssFiles 
VALUES ('div.example.a {background-color:yellow} div.example.b{background-color:aqua}', '/database/a.css', GETDATE())
{% endhighlight %}

For now, if we open a browser we can see that the CSS can be loaded from a database.

![before]({{ site.baseurl }}/images/2021-11-29-1/web-before-change.jpg)

The first step is done and CSS is successfully loaded to the browser. At the same time if we go to the database and change CSS the bundle will be the same. The reason for it is the implementation of Watch method. For now, there is no logic in it. This logic will be added in the [next part]({%link _posts/2021-12-01-CSS-in-database-ASP-Net-Part-2.md %}).

## Conclusion:

In this part, we can store the CSS in the database and create a bundle that can load this CSS into the page. The next step is auto-updating this bundle if CSS in the database is changed.

The source code can be found here: [Source Code](https://github.com/Kirmiir/DatabaseCss)
