# Build an app

At this point you should have [set up your environment and installed Panel](installation.md) and been introduced some of the basic ideas and APIs behind Panel in the [Core Concepts](core_concepts.md) guide.

However the easiest way to get started with Panel is to play around with it yourself. In this section we're going to be building a basic application using a public dataset and add some interactivity. The easiest way to follow along is to copy the examples and launch a Jupyter notebook session as described in the [installation guide](installation.md).

:::{important}
This guide is written for an interactive environment such as Jupyter notebooks. The interactive widgets will not work in a static version of this documentation.
:::

## Fetch some data

Panel lets you add interactive controls for just about anything you can display in Python. Panel can help you build simple interactive apps, complex multi-page dashboards, or anything in between. As a simple example, let's say we have loaded the [UCI ML dataset measuring the environment in a meeting room](http://archive.ics.uci.edu/ml/datasets/Occupancy+Detection+):


```{pyodide}
import pandas as pd; import numpy as np; import matplotlib.pyplot as plt

csv_file = 'https://raw.githubusercontent.com/holoviz/panel/master/examples/assets/occupancy.csv'
data = pd.read_csv(csv_file, parse_dates=['date'], index_col='date')

data.tail()
```

## Visualize the data

And we've written some code that smooths a time series and plots it using Matplotlib with outliers highlighted:


```{pyodide}
import matplotlib as mpl
mpl.use('agg')

from matplotlib.figure import Figure

def mpl_plot(avg, highlight):
    fig = Figure()
    ax = fig.add_subplot()
    avg.plot(ax=ax)
    if len(highlight): highlight.plot(style='o', ax=ax)
    return fig

def find_outliers(variable='Temperature', window=30, sigma=10, view_fn=mpl_plot):
    avg = data[variable].rolling(window=window).mean()
    residual = data[variable] - avg
    std = residual.rolling(window=window).std()
    outliers = (np.abs(residual) > std * sigma)
    return view_fn(avg, avg[outliers])
```

We can call the function with parameters and get a plot:


```{pyodide}
find_outliers(variable='Temperature', window=20, sigma=10)
```

It works! But exploring all these parameters by typing Python is slow and tedious. Plus we want our boss, or the boss's boss, to be able to try it out.

If we wanted to try out lots of combinations of these values to understand how the window and sigma affect the plot, we could reevaluate the above cell lots of times, but that would be a slow and painful process, and is only really appropriate for users who are comfortable with editing Python code. In the next few examples we will demonstrate how to use Panel to quickly add some interactive controls to some object and make a simple app.

## Reactive functions

Instead of editing code, it's much quicker and more straightforward to use sliders to adjust the values interactively. You can easily make a Panel app by binding some widgets to the arguments of a function using `pn.bind`:


```{pyodide}
import panel as pn
pn.extension()

window = pn.widgets.IntSlider(name='window', value=30, start=1, end=60)
sigma = pn.widgets.IntSlider(name='sigma', value=10, start=0, end=20)

interactive = pn.bind(find_outliers, window=window, sigma=sigma)
```

Once you have bound the widgets to the function's arguments you can lay out the component being returned using Panel layout components such as `Row`, `Column`, or `FlexBox`:


```{pyodide}
first_app = pn.Column(window, sigma, interactive)

first_app
```

As long as you have a live Python process running, dragging these widgets will trigger a call to the `find_outliers` callback function, evaluating it for whatever combination of parameter values you select and displaying the results. A Panel like this makes it very easy to explore any function that produces a visual result of a [supported type](https://github.com/pyviz/panel/issues/2), such as Matplotlib (as above), Bokeh, Plotly, Altair, or various text and image types.

## Deploying Panels

The above panels all work in the notebook cell (if you have a live Jupyter kernel running), but unlike other approaches such as ipywidgets, Panel apps work just the same in a standalone server. For instance, the app above can be launched as its own web server on your machine by uncommenting and running the following cell:


```{pyodide}
#first_app.show()
```

Or, you can simply mark whatever you want to be in the separate web page with `.servable()`, and then run the shell command `panel serve --show Create_App.ipynb` to launch a server containing that object. (Here, we've also added a semicolon to avoid getting another copy of the occupancy app here in the notebook.)


```{pyodide}
first_app.servable();
```

During development, particularly when working with a raw script using `panel serve --show --autoreload` can be very useful as the application will automatically update whenever the script or notebook or any of its imports change.

## Declarative Panels

The above compositional approach is very flexible, but it ties your domain-specific code (the parts about sine waves) with your widget display code. That's fine for small, quick projects or projects dominated by visualization code, but what about large-scale, long-lived projects, where the code is used in many different contexts over time, such as in large batch runs, one-off command-line usage, notebooks, and deployed dashboards?  For larger projects like that, it's important to be able to separate the parts of the code that are about the underlying domain (i.e. application or research area) from those that are tied to specific display technologies (such as Jupyter notebooks or web servers).

For such usages, Panel supports objects declared with the separate [Param](http://param.pyviz.org) library, which provides a GUI-independent way of capturing and declaring the parameters of your objects (and dependencies between your code and those parameters), in a way that's independent of any particular application or dashboard technology. For instance, the above code can be captured in an object that declares the ranges and values of all parameters, as well as how to generate the plot, independently of the Panel library or any other way of interacting with the object:


```{pyodide}
import param
import hvplot.pandas

def hvplot(avg, highlight):
    return avg.hvplot(height=200) * highlight.hvplot.scatter(color='orange', padding=0.1)

class RoomOccupancy(param.Parameterized):
    variable  = param.Selector(objects=list(data.columns))
    window    = param.Integer(default=10, bounds=(1, 20))
    sigma     = param.Number(default=10, bounds=(0, 20))

    def view(self):
        return find_outliers(self.variable, self.window, self.sigma, view_fn=hvplot)

obj = RoomOccupancy()
obj
```

The `RoomOccupancy` class and the `obj` instance have no dependency on Panel, Jupyter, or any other GUI or web toolkit; they simply declare facts about a certain domain (such as that smoothing requires window and sigma parameters, and that window is an integer greater than 0 and sigma is a positive real number).  This information is then enough for Panel to create an editable and viewable representation for this object without having to specify anything that depends on the domain-specific details encapsulated in `obj`:


```{pyodide}
pn.Row(obj.param, obj.view)
```

To support a particular domain, you can create hierarchies of such classes encapsulating all the parameters and functionality you need across different families of objects, with both parameters and code inheriting across the classes as appropriate, all without any dependency on a particular GUI library or even the presence of a GUI at all.  This approach makes it practical to maintain a large codebase, all fully displayable and editable with Panel, in a way that can be maintained and adapted over time.

## Exploring further

For a quick reference of different Panel functionality refer to the [overview](../user_guide/Overview.md). If you want a more detailed description of different ways of using Panel, each appropriate for different applications see the following materials:

- [APIs](../user_guide/APIs.rst): An overview of the different APIs offered by Panel.
- [Parameters](../user_guide/Param.rst): Capturing parameters and their links to actions declaratively

Just pick the style that seems most appropriate for the task you want to do, then study that section of the user guide. Regardless of which approach you take, you'll want to learn more about Panel's panes and layouts:

- [Components](../user_guide/Components.rst): An overview of the core components of Panel including Panes, Widgets and Layouts
- [Customization](../user_guide/Customization.rst): How to set styles and sizes of Panel components
- [Display & Export](../user_guide/Display_and_Export.rst): An overview on how to display and export Panel components and apps.
- [Server Configuration](../user_guide/Server_Configuration.md): An overview on how to configure applications and dashboards for deployment.
- [Templates](../user_guide/Templates.rst): Composing one or more Panel objects into a template with full control over layout and styling.
