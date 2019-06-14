# Plugin examples

In this section, we give a few examples of plugins.

## Hello world

Here is how to write a useless plugin that displays a message every time there's a new cluster assignment:

```python
from phy import IPlugin, connect

class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        @connect(sender=controller)
        def on_gui_ready(sender, gui):
            """This is called when the GUI and all objects are fully loaded.
            This is to make sure that controller.supervisor is properly defined.
            """

            @connect(sender=controller.supervisor)
            def on_cluster(sender, up):
                """This is called every time a cluster assignment or cluster group/label changes."""
                print("Clusters update: %s" % up)
```

This displays the following in the console, for example:

```
Clusters update: <merge [212, 299] => [305]>
Clusters update: <metadata_group [305] => good>
Clusters update: <metadata_neurontype [81, 82] => interneuron>
```


## Changing the number of spikes in views

You can customize the maximum number of spikes in the different views as follows (the example below shows the default values):

```python
from phy import IPlugin, connect

class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        """Feel free to keep below just the values you need to change."""
        controller.n_spikes_waveforms = 100
        controller.batch_size_waveforms = 10
        controller.n_spikes_features = 2500
        controller.n_spikes_features_background = 1000
        controller.n_spikes_amplitudes = 5000
        controller.n_spikes_correlograms = 100000

        # Number of "best" channels kept for displaying the waveforms.
        controller.model.n_closest_channels = 16
```

*Note*: you need to manually delete the `.phy` subdirectory within your data directory when changing these parameters, otherwise errors will happen in the GUI.


## Defining a custom cluster metrics

In addition to cluster labels that you can create and modify in the cluster view, you can also define **cluster metrics**.

A cluster metrics is a function that assigns a scalar to every cluster. For example, any algorithm computing a kind of cluster quality is a cluster metrics.

You can define your own cluster metrics in a plugin. The values are then shown in a new column in the cluster view, and recomputed automatically when changing cluster assignments.

You can use the controller's methods to access the data. In rare cases, you may need to access the model directly (via `controller.model`).

For example, here is a custom cluster metrics that shows the mean inter-spike interval.

![image](https://user-images.githubusercontent.com/1942359/59463223-cf0a3e80-8e25-11e9-990f-e3e2dc9418a0.png)

```python
import numpy as np
from phy import IPlugin, connect

class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        """Note that this function is called at initialization time, *before* the supervisor is
        created. The `controller.cluster_metrics` items are then passed to the supervisor when
        constructing it."""

        def meanisi(cluster_id):
            t = controller.get_spike_times(cluster_id).data
            return np.diff(t).mean()

        # Use this dictionary to define custom cluster metrics.
        controller.cluster_metrics['meanisi'] = meanisi
```


## Writing a custom cluster statistics view

The ISI view, firing rate view, and amplitude histogram view all share the same characteristics. These views show histograms of cluster-dependent values. The number of bins and the maximum bin can be customized. All of these views derive from the **HistogramView**, a generic class for cluster statistics that can be displayed as histograms.

In the following example, we define a custom cluster statistics views using the PC features.

```python
from phy import IPlugin, Bunch
from phy.cluster.views import HistogramView


class FeatureHistogramView(HistogramView):
    n_bins = 100  # default number of bins
    x_max = .1  # maximum value on the x axis (maximum bin)
    alias_char = 'fh'  # provide `fhn` (set number of bins) and `fhm` (set max bin) snippets


class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):

        def feature_histogram(cluster_id):
            """Must return a Bunch object with data and optional x_max, plot, text items.

            The histogram is automatically computed by the view, this function should return
            the original data used to compute the histogram, rather than the histogram itself.

            """
            return Bunch(data=controller.get_features(cluster_id).data)

        def create_view():
            """Create and return a histogram view."""
            return FeatureHistogramView(cluster_stat=feature_histogram)

        # Maps a view name to a function that returns a view
        # when called with no argument.
        controller.view_creator['FeatureHistogram'] = create_view

```

![image](https://user-images.githubusercontent.com/1942359/58968835-fd00da80-87b6-11e9-8218-5adfc6e22a78.png)


## Customizing the styling of the cluster view

The cluster view is written in HTML/CSS/Javascript. The styling can be customized in a plugin as follows.

In this example, we change the text color of "good" clusters in the cluster view.

![image](https://user-images.githubusercontent.com/1942359/59463245-dd585a80-8e25-11e9-9fe3-56aa4c3733c7.png)

```python
from phy import IPlugin
from phy.cluster.supervisor import ClusterView

class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        # We add a custom CSS style to the ClusterView.
        ClusterView._styles += """

            /* This CSS selector represents all rows for good clusters. */
            table tr[data-group='good'] {

                /* We change the text color. Many other CSS attributes can be changed,
                such as background-color, the font weight, etc. */
                color: red;
            }

        """
```


## Adding a new action

In this example, we create a new action in the file menu, with keyboard shortcut `a`, to display a message in the status bar. We create another action to select the first N clusters in the cluster view, where N is a parameter that the user can type in a prompt dialog.

![image](https://user-images.githubusercontent.com/1942359/59464230-2b6e5d80-8e28-11e9-804d-b94a57920f32.png)

```python
from phy import IPlugin, connect

class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        @connect
        def on_gui_ready(sender, gui):

            # Add a separator at the end of the File menu.
            # Note: currently, there is no way to add actions at another position in the menu.
            gui.file_actions.separator()

            # Add a new action to the File menu.
            @gui.file_actions.add(shortcut='a')  # the keyboard shortcut is A
            def display_message():
                """Display Hello world in the status bar."""
                # This docstring will be displayed in the status bar when hovering the mouse over
                # the menu item.

                # We update the text in the status bar.
                gui.status_message = "Hello world"

            # We add a separator at the end of the Select menu.
            gui.select_actions.separator()

            # Add an action to a new submenu called "My submenu". This action displays a prompt
            # dialog with the default value 10.
            @gui.select_actions.add(
                submenu='My submenu', shortcut='ctrl+c', prompt=True, prompt_default=lambda: 10)
            def select_n_first_clusters(n_clusters):

                # All cluster view methods are called with a callback function because of the
                # asynchronous nature of Python-Javascript interactions in Qt5.
                @controller.supervisor.cluster_view.get_ids
                def get_cluster_ids(cluster_ids):
                    """This function is called when the ordered list of cluster ids is returned
                    by the Javascript view."""

                    # We select the first n_clusters clusters.
                    controller.supervisor.select(cluster_ids[:n_clusters])

```


## Saving cluster metadata in a TSV file

In this example, we show how to save the best channel of all clusters in a TSV file.

![image](https://user-images.githubusercontent.com/1942359/59463165-a1bd9080-8e25-11e9-9260-8065197fd56b.png)

```python
import logging

from phy import IPlugin, connect
from phylib.io.model import save_metadata

logger = logging.getLogger('phy')

class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        @connect
        def on_gui_ready(sender, gui):

            @connect(sender=gui)
            def on_request_save(sender):
                """This function is called whenever the Save action is triggered."""

                # We get the filename.
                filename = controller.model.dir_path / 'cluster_channel.tsv'

                # We get the list of all clusters.
                cluster_ids = controller.supervisor.clustering.cluster_ids

                # Field name used in the header of the TSV file.
                field_name = 'channel'

                # NOTE: cluster_XXX.tsv files are automatically loaded in phy, displayed
                # in the cluster view, and interpreted as cluster labels, *except* if their
                # name conflicts with an existing built-in column in the cluster view.
                # This is the case here, because there is a default channel column in phy.
                # Therefore, the TSV file is properly saved, but it is not displayed in the
                # cluster view as the information is already shown in the built-in channel column.
                # If you want this file to be loaded in the cluster view, just use another
                # name that is not already used, like 'best_channel'.

                # Dictionary mapping cluster_ids to the best channel id.
                metadata = {
                    cluster_id: controller.get_best_channel(cluster_id)
                    for cluster_id in cluster_ids}

                # Save the metadata file.
                save_metadata(filename, field_name, metadata)
                logger.info("Saved %s.", filename)
```


## Inject custom variables in the IPythonView

In this example, we show how to inject variables into the IPython console namespace. Specifically, we inject the first waveform view with the variable name `wv`.

```python
from phy import IPlugin, connect
from phy.cluster.views import WaveformView
from phy.gui.widgets import IPythonView

class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        @connect
        def on_add_view(gui, view):
            # This is called whenever a new view is added to the GUI.
            if isinstance(view, IPythonView):

                # We inject the first WaveformView of the GUI to the IPython console.
                view.inject(wv=gui.get_view(WaveformView))
```


## Customize the feature view

In this example, we show how to customize the subplots in the feature view.

![image](https://user-images.githubusercontent.com/1942359/59465531-6625c500-8e2b-11e9-8442-c00530878959.png)

```python
import re
from phy import IPlugin, connect
from phy.cluster.views import FeatureView

def my_grid():
    """In the grid specification, 0 corresponds to the best channel, 1
    to the second best, and so on. A, B, C refer to the PC components."""
    s = """
    0A,1A 1A,2A 2A,0A
    0B,1B 1B,2B 2B,0B
    0C,1C 1C,2C 2C,0C
    """.strip()
    return [[_ for _ in re.split(' +', line.strip())] for line in s.splitlines()]

class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):

        @connect
        def on_add_view(gui, view):
            if isinstance(view, FeatureView):
                # We change the specification of the subplots here.
                view.set_grid_dim(my_grid())
```


## Writing a custom matplotlib view

Most built-in views in phy are based on OpenGL instead of matplotlib, for performance reasons. Since writing OpenGL views is significantly more complex than with matplotlib, we cover OpenGL views later in this documentation.

In this example, we show how to create a custom view based on matplotlib. Specifically, we show a 2D histogram of spike features.

![image](https://user-images.githubusercontent.com/1942359/58970988-ccbb3b00-87ba-11e9-93e9-c900d76a54f2.png)

```python
from phy import IPlugin
from phy.cluster.views import ManualClusteringView  # Base class for phy views
from phy.plot.plot import PlotCanvasMpl  # matplotlib canvas

class FeatureDensityView(ManualClusteringView):
    plot_canvas_class = PlotCanvasMpl  # use matplotlib instead of OpenGL (the default)

    def __init__(self, features=None):
        """features is a function (cluster_id => Bunch(data, ...)) where data is a 3D array."""
        super(FeatureDensityView, self).__init__()
        self.features = features

    def on_select(self, cluster_ids=(), **kwargs):
        self.cluster_ids = cluster_ids
        # We don't display anything if no clusters are selected.
        if not cluster_ids:
            return

        # To simplify, we only consider the first PC component of the first 2 best channels.
        # Note that the features are in sparse format, where data's shape is
        # (n_spikes, n_best_channels, n_pcs). Only best channels for that clusters are
        # considered.
        # For this example, we just take the first 2 dimensions.
        x, y = self.features(cluster_ids[0]).data[:, :2, 0].T

        # We draw a 2D histogram with matplotlib.
        # The objects are:
        # - self.figure, a Figure instance
        # - self.canvas, a PlotCanvasMpl instance
        # - self.canvas.ax, an Axes object.
        self.canvas.ax.hist2d(x, y, 50)

        # Use this to update the matplotlib figure.
        self.canvas.update()


class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        def create_feature_density_view():
            """A function that creates and returns a view."""
            return FeatureDensityView(features=controller.get_features)

        controller.view_creator['FeatureDensityView'] = create_feature_density_view

```


## Writing a custom OpenGL view

For increased performance, all built-in views in phy are not based on matplotlib, but on OpenGL. OpenGL is a real-time graphics programming interface developed for video games. It also provides hardware acceleration for fast display of large amounts of data.

In phy, OpenGL views are written on top of a thin layer, a fork of `glumpy.gloo` (object-oriented interface to OpenGL). On top of that, the `phy.plot` module proposes a minimal plotting API. This interface is complex as it suffers from the limitations of OpenGL. As such, writing custom OpenGL views for phy is not straightforward.

Here, we give a minimal example of a plugin implementing a custom OpenGL view. There is no in-depth documentation at the moment. If you need a really specific view, send me (Cyrille Rossant) an email at `myfirstname.mylastname@gmail.com`.

### Example

In this example, we simply display the template waveform on the peak channel of selected clusters.

*Note*: OpenGL is most useful when there is a lot of data (between tens of thousands and millions of points). For a plot as simple as this, you could as well use matplotlib. However, the method presented here scales well to plots with many more points.

![image](https://user-images.githubusercontent.com/1942359/59127305-9a9b0c00-8967-11e9-9dea-468a13fc98bc.png)


### Code

```python
import numpy as np

from phylib.utils.color import selected_cluster_color

from phy import IPlugin
from phy.cluster.views import ManualClusteringView
from phy.plot.visuals import PlotVisual


class MyView(ManualClusteringView):
    """All OpenGL views derive from ManualClusteringView."""

    def __init__(self, templates=None):
        """
        Typically, the constructor takes as arguments *functions* that take as input
        one or several cluster ids, and return as many Bunch instances which contain
        the data as NumPy arrays. Many such functions are defined in the TemplateController.
        """

        super(MyView, self).__init__()

        """
        The View instance contains a special `canvas` object which is a `̀PlotCanvas` instance.
        This class derives from `BaseCanvas` which itself derives from the PyQt5 `QOpenGLWindow`.
        The canvas represents a rectangular black window where you can draw geometric objects
        with OpenGL.

        phy uses the notion of **Layout** that lets you organize graphical elements in different
        subplots. These subplots can be organized in several ways:

        * Grid layout: a `(n_rows, n_cols)` grid of subplots (example: FeatureView).
        * Boxed layout: boxes arbitrarily located (example: WaveformView, using the
          probe geometry)
        * Stacked layout: one column with `n_boxes` subplots (example: TraceView,
          one row per channel)

        In this example, we use the stacked layout, with one subplot per cluster. This number
        will change at each cluster selection, depending on the number of selected clusters.
        But initially, we just use 1 subplot.

        """
        self.canvas.set_layout('stacked', n_plots=1)

        self.templates = templates

        """
        phy uses the notion of **Visual**. This is a graphical element that is represented with
        a single type of graphical element. phy provides many visuals:

        * PlotVisual (plots)
        * ScatterVisual (points with a given marker type and different colors and sizes)
        * LineVisual (for lines segments)
        * HistogramVisual
        * PolygonVisual
        * TextVisual
        * ImageVisual

        Each visual comes with a single OpenGL program, which is defined by a vertex shader
        and a fragment shader. These are programs written in a C-like language called GLSL.
        A visual also comes with a primitive type, which can be points, line segments, or
        triangles. This is all a GPU is able to render, but the position and the color of
        these primitives can be entirely customized in the shaders.

        The vertex shader acts on data arrays represented as NumPy arrays.

        These low-level details are hidden by the visuals abstraction, so it is unlikely that
        you'll ever need to write your own visual.

        In ManualClusteringViews, you typically define one or several visuals. For example
        if you need to add text, you would add `self.text_visual = TextVisual()`.

        """
        self.visual = PlotVisual()

        """
        For internal reasons, you need to add all visuals (empty for now) directly to the
        canvas, in the view's constructor. Later, we will use the `visual.set_data()` method
        to update the visual's data and display something in the figure.

        """
        self.canvas.add_visual(self.visual)

    def on_select(self, cluster_ids=(), **kwargs):
        """
        The main method to implement in ManualClusteringView is `on_select()`, called whenever
        new clusters are selected.

        *Note*: `cluster_ids` contains the clusters selected in the cluster view, followed
        by clusters selected in the similarity view.

        """

        """
        This method should always start with these few lines of code.
        """
        self.cluster_ids = cluster_ids
        if not cluster_ids:
            return

        """
        We update the number of boxes in the stacked layout, which is the number of
        selected clusters.
        """
        self.canvas.stacked.n_boxes = len(cluster_ids)

        """
        We obtain the template data.
        """
        bunchs = self.templates(cluster_ids)

        """
        For performance reasons, it is best to use as few visuals as possible. In this example,
        we want 1 waveform template per subplot. We will use a single visual covering all
        subplots at once. This is the key to achieve good performance with OpenGL in Python.
        However, this comes with the drawback that the programming interface is more complicated.

        In principle, we would have to concatenate all data (x and y coordinates) of all subplots
        to pass it to `self.visual.set_data()` in order to draw all subplots at once. But this
        is tedious.

        phy uses the notion of **batch**: for each subplot, we set *partial data* for the subplot
        which just prepares the data for concatenation *after* we're done with looping through
        all clusters. The concatenation happens in the special call
        `self.canvas.update_visual(self.visual)`.

        We need to call `visual.reset_batch()` before constructing a batch.

        """
        self.visual.reset_batch()

        """
        We iterate through all selected clusters.
        """
        for idx, cluster_id in enumerate(cluster_ids):
            bunch = bunchs[cluster_id]

            """
            In this example, we just keep the peak channel. Note that `bunch.template` is a
            2D array `(n_samples, n_channels)` where `n_channels` in the number of "best"
            channels for the cluster. The channels are sorted by decreasing template amplitude,
            so the first one is the peak channel. The channel ids can be found in
            `bunch.channel_ids`.
            """
            y = bunch.template[:, 0]

            """
            We decide to use, on the x axis, values ranging from -1 to 1. This is the
            standard viewport in OpenGL and phy.
            """
            x = np.linspace(-1., 1., len(y))

            """
            phy requires you to specify explicitly the x and y range of the plots.
            The `data_bounds` variable is a `(xmin, ymin, xmax, ymax)` tuple representing the
            lower-left and upper-right corners of a rectangle. By default, the data bounds
            of the entire view is (-1, -1, 1, 1), also called normalized device coordinates.
            Eventually, OpenGL uses this coordinate system for display, but phy provides
            a transform system to convert from different coordinate systems, both on the CPU
            and the GPU.

            Here, the x range is (-1, 1), and the y range is (m, M) where m and M are
            respectively the min and max of the template.
            """
            m, M = y.min(), y.max()
            data_bounds = (-1, m, +1, M)

            """
            This function gives the color of the i-th selected cluster. This is a 4-tuple with
            values between 0 and 1 for RGBA: red, green, blue, alpha channel (transparency,
            1 by default).
            """
            color = selected_cluster_color(idx)

            """
            The plot visual takes as input the x and y coordinates of the points, the color,
            and the data bounds.
            There is also a special keyword argument `box_index` which is the subplot index.
            In the stacked layout, this is just an integer identifying the subplot index, from
            top to bottom. Note that in the grid view, the box index is a pair (row, col).
            """
            self.visual.add_batch_data(
                x=x, y=y, color=color, data_bounds=data_bounds, box_index=idx)

        """
        After the loop, this special call automatically builds the data to upload to the GPU
        by concatenating the partial data set in `add_batch_data()`.
        """
        self.canvas.update_visual(self.visual)

        """
        After updating the data on the GPU, we need to refresh the canvas.
        """
        self.canvas.update()


class MyPlugin(IPlugin):
    def attach_to_controller(self, controller):
        def create_my_view():
            return MyView(templates=controller.get_templates)

        controller.view_creator['MyView'] = create_my_view

```