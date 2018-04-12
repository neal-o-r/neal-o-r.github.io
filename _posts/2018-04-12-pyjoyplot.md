---
layout: post

title: Joy Plots in Python

---


Joy Plots are a really handy way to get a quick, qualitative impression of a data set. They also look really cool. So I decided to put together a [Python package](https://github.com/neal-o-r/pyjoyplot) to make them.

Joy plots have been around a long time, but have become popular in the last while in the data visualisation community. The name doesn't come from the the sense of joy engendered by looking at the plots but from the band Joy Division, who put one on the cover of their classic Unknown Pleasures album. It's a slightly cryptic album cover, just a series of spiky line plots, white-on-black, stacked to create a kind of 3D mountain range effect. It's an image that you've definitely seen, even if you've never listened to music in your life.

![pleasures](/images/pyjoy/pleasures.jpg)

You'd have to agree that this is a very cool plot, and it's probably the most famous piece of data visualisation ever. Like most good things it's got an astrophysics connection, the data plotted here are observations of the first pulsar discovered, CP-1919. Pulsars are rapidly rotating neutron stars that emit pulses of radio waves as they spin - think of the beam from a light-house. This plot shows the intensity of the radio waves detected; each line represents one rotation, with time running left to right, and successive rotations are stacked from bottom to top. These objects were discovered in 1967 by Northern Irish astronomer Jocelyn Bell Burnell while she was a PhD student at Cambridge. She didn't make this image however, it comes from the PhD thesis of a guy called [Harold Craft](https://blogs.scientificamerican.com/sa-visual/pop-culture-pulsar-origin-story-of-joy-division-s-unknown-pleasures-album-cover-video/). A good plot is timeless, and in my own doctoral work I found myself making exactly this kind of plot to look at how light from binary stars changed as the stars orbited one another.


You don't need to be an astrophysicist to make these though, any time that you've got data that can be grouped according to some categorical variable then joy plots can be a good tool to use. Since the end of last summer I've seen a lot of these plots around, probably because of the release of the R package [ggjoy](http://blog.revolutionanalytics.com/2017/07/joyplots.html) (since rebranded ggridges). I decided to write a Python package to do the same job, the source code is [here](https://github.com/neal-o-r/pyjoyplot) and you can download it from Pip ```pip install pyjoyplot```.


The package itself is pretty straightforward, it's just a thin wrapper to matplotlib. It uses the same kind of API as seaborn, it consumes a dataframe along with some keywords and gives back an ```ax``` object. To give you an example, here's a data set that I can across online:

<table border="1" class="dataframe">  <thead>    <tr style="text-align: right;">      <th>activity</th>      <th>time</th>      <th>playing</th>      <th>Hours</th>    </tr>  </thead>  <tbody>    <tr>      <td>Playing football</td>      <td>380.0</td>      <td>0.000006</td>      <td>6.333333</td>    </tr>    <tr>      <td>Playing baseball</td>      <td>940.0</td>      <td>0.000367</td>      <td>15.666667</td>    </tr>    <tr>      <td>Playing baseball</td>      <td>630.0</td>      <td>0.000124</td>      <td>10.500000</td>    </tr>    <tr>      <td>Rollerblading</td>      <td>330.0</td>      <td>0.000004</td>      <td>5.500000</td>    </tr>    <tr>      <td>Wrestling</td>      <td>985.0</td>      <td>0.000072</td>      <td>16.416667</td>    </tr>    <tr>      <td>Dancing</td>      <td>760.0</td>      <td>0.000069</td>      <td>12.666667</td>    </tr>    <tr>      <td>Dancing</td>      <td>900.0</td>      <td>0.000185</td>      <td>15.000000</td>    </tr>    <tr>      <td>Dancing</td>      <td>100.0</td>      <td>0.000376</td>      <td>1.666667</td>    </tr>    <tr>      <td>Playing racquet sports</td>      <td>645.0</td>      <td>0.000424</td>      <td>10.750000</td>    </tr>    <tr>      <td>Wrestling</td>      <td>695.0</td>      <td>0.000022</td>      <td>11.583333</td>    </tr>  </tbody></table>
This data set contains a bunch of sporting activities, a set of times (in minutes since midnight), and a fraction(?) of people doing the acitivites. I think. It's not really important anyway, the point is we can use pyjoyplot to visualise these data.

```python
import pyjoyplot as pjp
import pandas as pd

df = pd.read_csv('sports.csv')
df['hours'] = df.time / 60

pjp.plot(data=df, x='hours',  y='playing', hue='activity')
```

We hand in the dataframe, and specify the $x$ axis, $y$ axis, and the hue, where hue designates the categorical variable to be grouped over, and we end up with this:

![pjp1](/images/pyjoy/sports1.png)

Easy! These data are a bit rough around the edges, so maybe we'd like to smooth them a little to make things cleaner. To do that we can just set the ```smooth``` parameter which computes a rolling mean using a sliding window whatever width you specify:
```python
pjp.plot(data=df, x='hours',
              y='playing', hue='activity', smooth=10)
```

and we get

![pjp2](/images/pyjoy/activities.png)

Nice and straightforward, and not a bad looking plot if I do say so myself.

One other thing that the package support is stacking histograms. Take the classic iris data set,

<table border="1" class="dataframe">  <thead>    <tr style="text-align: right;">      <th>sepal_length</th>      <th>sepal_width</th>      <th>petal_length</th>      <th>petal_width</th>      <th>species</th>    </tr>  </thead>  <tbody>    <tr>      <td>5.5</td>      <td>2.3</td>      <td>4.0</td>      <td>1.3</td>      <td>versicolor</td>    </tr>    <tr>      <td>7.3</td>      <td>2.9</td>      <td>6.3</td>      <td>1.8</td>      <td>virginica</td>    </tr>    <tr>      <td>5.2</td>      <td>4.1</td>      <td>1.5</td>      <td>0.1</td>      <td>setosa</td>    </tr>    <tr>      <td>6.7</td>      <td>3.1</td>      <td>4.7</td>      <td>1.5</td>      <td>versicolor</td>    </tr>    <tr>      <td>6.8</td>      <td>3.0</td>      <td>5.5</td>      <td>2.1</td>      <td>virginica</td></tr></tbody></table>

Here we have 4 parameters for 3 different kinds of iris. Let's say we wanted to get an idea of the distribution of sepal length for each type, we can do this:

```python
pjp.plot(data=iris, x='sepal_length', hue='species', bins=10, kind='hist')
```

Much as before we hand in the dataframe, specify the variable we want to bin as $x$, and the variable we want to group on has hue. This time however we say that we want a histogram, and give a number of bins too, and we get:

![iris](/images/pyjoy/iris.png)

Lovely.

I've seen a bit of interest in making plots like these in the last while, seaborn posted a [recipe](https://seaborn.pydata.org/examples/kde_joyplot.html), [bokeh](https://bokeh.pydata.org/en/latest/docs/gallery/joyplot.html) too, and there's at least one other [package](http://sbebo.github.io/blog/blog/2017/08/01/joypy/) out there, so you're spoiled for choice!
