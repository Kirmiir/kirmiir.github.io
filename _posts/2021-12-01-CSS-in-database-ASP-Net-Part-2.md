---
layout: post
title: ASP.NET Core. Store css in Database and combine it in Bundle. Part 2
---

At the end of the previous part, we created a bundle with CSS from the database. Now we need to add the logic to update it if there are any changes.

The first step is tracking the SQL changes.
It can be done by using a SqlDependencyEx.

{% highlight charp %}
public class DataBaseWatcher
{
    private SqlDependencyEx _listener;
    private readonly ConcurrentBag<Action> _callbacks;

    private static DataBaseWatcher _instance;
    
    public static DataBaseWatcher GetInstance()
    {
        return _instance ??= new DataBaseWatcher();
    }

    private DataBaseWatcher()
    {
        _callbacks = new ConcurrentBag<Action>();
    }

    public void RegisterCallback(Action callback)
    {
        _callbacks.Add(callback);
    }
    
    public void Initialization(string connectionString)
    {
        _listener = new SqlDependencyEx(connectionString, "DatabaseCss", "CssFiles");
        _listener.TableChanged += (_, _) =>
        {
            foreach (var callback in _callbacks)
            {
                callback();
            }
        };
        _listener.Start();
    }

    public void Termination()
    {
        _listener.Stop();
    }
}
{% endhighlight %}
This class contains the collection of _callbacks that we need to call as soon as there are changes in the database for the particular table ("CssFiles") 
There is a simple method to add a new callback to this collection RegisterCallback.

The next step is adding a new DataBaseChangeToken class.
{% highlight charp %}
private class DataBaseChangeToken : IChangeToken
{
    private bool _hasChanged;
    public DataBaseChangeToken(DataBaseWatcher watcher)
    {
        watcher.RegisterCallback(() => _hasChanged = true);
    }

    public IDisposable RegisterChangeCallback(Action<object> callback, object state)
    {
        throw new NotImplementedException();
    }

    public bool ActiveChangeCallbacks => false;
    public bool HasChanged => _hasChanged;
}
{% endhighlight %}

There is no complicated logic here but the main idea is to return true from HasChange if the CSS was updated in the database.

The last thing that we need to change is to update Watch method in DatabaseFileProvider to use DataBaseChangeToken class.

{% highlight charp %}
public IChangeToken Watch(string filter)
{
    return new DataBaseChangeToken(DataBaseWatcher.GetInstance());
}
{% endhighlight %}

Let open the browser and check that the current view is:
![before]({{ site.baseurl }}/images/2021-11-29-1/web-before-change.jpg)

Now we can go to the database and change the CSS field to the following:

{% highlight css %}
div.a {background-color:red} div.b{background-color:aqua}
{% endhighlight %}

After that, if we open the browser again and refresh the page the CSS will be updated and we will see:

![after]({{ site.baseurl }}/images/2021-11-29-1/web-after-change.jpg)

This is it. For now, there is a simple application that can use CSS from the database.

## Conclusion:
In this part, the auto-updating CSS was added to the CSS bundle that is stored in the database.
This example illustrates one of the ways how CSS can be stored in the database and integrated into ASP.Net Core website.

The [first part]({%link _posts/2021-11-29-CSS-in-database-ASP-Net.md %}).

The source code can be found here: [Source Code](https://github.com/Kirmiir/DatabaseCss)