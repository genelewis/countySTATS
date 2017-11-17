# Oklahoma Policy Institute: CountySTATS
This project is a remake of the five charts used on OKPolicy CountySTATS:
https://okpolicy.org/resources/countystats-2015/

These charts have been simplified, and now rely on HTML/JavaScript using the [D3.js library](https://d3js.org/) for added transitions and user interactivity.

To display these charts on a local machine, download the files into a directory of your choice and view `index.html` locally on your web browser.  (Downloading [Brackets](http://brackets.io/) and using its Live Preview feature with Chrome is my recommended way of doing this.)

### Updating Data Points
- - - -
All the data for these charts are held in .CSV files, which can be updated in a text editor or in Excel.  I’ve included @genelewis’s additional .CSV files that have a bit more information in the folder `unusedStatistics`, but the CSV’s that the charts rely on are stripped down versions with _only_ the necessary data and nothing else.

The code will take care of formatting the numbers; if editing the data points in Excel, there’s no need to convert into a specific number format (e.g., “Percentage”).  There’s also no hard-coded limit to the number of data points one chart can interpret (if a 78th county was created in Oklahoma, it could be safely added to the .CSV files and the selector on `index.html` to be viewed.

### Converting Charts to an Image Format (for static county factsheets)
- - - -
After county data has been updated, the printable, 2-page factsheets will also need to have their images of each chart updated to reflect the changes.  

Each of the 5 charts here is built as an SVG, and this format can be saved and used to build the printable factsheets for each county.  In my estimation, the best way of doing this is by downloading the New York Times’ Chrome tool [SVG Crowbar](http://nytimes.github.io/svg-crowbar/).  The Crowbar bookmarklet can be used to save each chart individually, if albeit somewhat tediously.  The charts should all have unique, dynamic class names to help with providing a clean file tree/nomenclature.

Since the printable charts aren’t interactive, you may wish for some of the charts to automatically display their values during this process (instead of having to hover over a bar to see its value, say).  This can be done by locating the following line of code at the top of `index.html` :

```
var INTERACTIVITY = True
```

and changing the value from **True** to **False**.

### General Code Outline
- - - -
In the event any of these charts need to be modified, here’s a quick rundown of how the code works to create these charts.  Again, all of this code relies heavily on the [D3.js library](https://d3js.org/); it’s helpful to have a general understanding of how D3 works (particularly with regards to its ENTER > UPDATE > EXIT strategy) before continuing on, but not necessary.

I’ll start by going through the code for the charts, and then briefly talk about the strategy I used in making the `index.html` page to display them all (this is likely similar, but **NOT** the same as how these charts are actually implemented on the OKPolicy website; however, it may be helpful if you’re just doing data updates and want to use this page as a quick reference tool).

##### Creating the Charts:

Each chart has a basic HTML header that defines any of its necessary style elements, source scripts, and the overarching SVG container element, which looks something like this:

```js
<!DOCTYPE html>
<style>
  //CSS style goes here
</style>

<title>County Demographics</title>

<script src="https://d3js.org/d3.v4.min.js"></script>
<script src="d3-tip.js"></script> //Responsible for showing bar tooltips on mouseover

<svg class="Demograpics" width="480" height="260"><title>Demographics</title></svg>
```

The rest is all Javascript code embedded in a `<script>` element.  For the most part, each chart’s code follows the same basic outline:

1. Define the SVG using [margin conventions](https://bl.ocks.org/mbostock/3019563) and use D3 to set variables for the scales along the x- and y-axes (as well as a coloring scale, if needed).

2. Open the data file using `d3.csv`.  This method may use an optional callback function to help read in and format the data, and will always use another callback function to build the chart itself during the following steps.

3. Create any other helpful variables, set the default selection value (‘Adair County’), and set the _domains_ of the x- and y-axes (note that this step is different than the way we previously set the _scale_).  Note also that at this point, we haven’t created any visuals inside of our original SVG container, we’ve just done a bit of math and housekeeping to make creating those visuals easier in the next step.

4. Now we start creating visuals by appending other kinds of SVG elements (path, rect, g, etc.) to our initial container.  We accomplish this using the D3 ENTER > UPDATE > EXIT strategy, where we can essentially create one data point (i.e., a point on a line graph, a bar in a bar graph) for each item in the array of data that we loaded in from our CSV (though that is an oversimplification).

What’s important to keep in mind here is that at any given time our data-array is really just a slice of the whole CSV— the slice that has to do with whatever county we’ve selected in the drop-down menu (from `index.html`).  When we change our selection, say by switching the drop-down from ‘Adair County’ to ‘Tulsa County,’ we’ll have to update this data, and that’s what we’ll do in the next step.

5. Use D3 to grab the dropdown element from the parent document (`index.html`); then, set an event listener to look out for any time the selection on this dropdown menu gets changed.  The code for this part looks nearly identical in every chart:

``` js
var selector = window.parent.document.getElementById('dropdown');
selector = d3.select(selector);
  .on("change.demo", function(d) {
```

One weird thing to watch out for is that each event listener must have its own **unique namespace**— otherwise, the listeners from each chart will overwrite each other when they get put together.  The code from the above example comes from the ‘Demographics’ chart, so I’ve given the event listener the unique namespace of `.demo`.  The other charts follow a similar convention.

The lambda callback function here then defines what parts of the chart should shift to the new county.  This varies depending on the chart, and the charts use D3 transitions to help make sure that the shift is smooth and able to be followed by the user’s eye.  In most instances, this means recalculating which slice of data from the .CSV we’re working with, and then some combination of moving plot points/bar lines and rescaling the x- and y-axes to keep the new data to a good fit.  We also update the labels on the legend here.

One problem I encountered was that in several of the graphs, the x-axis labels were too long and had to be wrapped to multiple lines.  Unfortunately, I couldn’t get D3 transitions to play nice with the wrapped labels (they reset to one line of text every time a new county is selected).  The labels can be wrapped again, but it results in a bit of “jarring-ness” during transition.

6. Finally, the code finishes out by appending the legend to each chart.  (It may make more sense for this to happen with Step 4, but I found it useful/more readable to separate the legend from the chart itself.)

In a few instances, there are also some helper functions at the bottom of the code that help us to get the exact slice of the CSV that we want and convert it to useable data with D3.  For example, I included a couple of helper methods in the chart for Top Employment Sectors that will break the CSV file down into manageable components, and then sort all of the sectors from largest to smallest by county value.
- - - -
##### Putting It All Together:

After creating the charts, the next thing we’d like to do is put them together so all 5 are visible on one page.  We’d also like to have a dropdown menu where users can select one of Oklahoma’s counties, and then have all 5 charts transition to data for that particular county.  “One dropdown menu to rule them all,” so-to-speak.

I did this in the `index.html` file, which was primarily my prototype for how things might work and look on the actually OKPolicy website.  Essentially, all that is needed are some iFrames to hold each chart; the code for that looks like this (note that the source file is just relative to my own local machine):

```html
  <h2>Population</h2> 
  <div class="population">
    <iframe id="population" sandbox="allow-popups allow-scripts allow-forms allow-same-origin" src="/Population/Population.html" marginwidth="0" marginheight="0" style="height:260px; width:480px" scrolling="no"> </iframe>
  </div>
```

There’s some additional CSS styling to make the prototype page more compact and pleasing to the eye, but this is the main gist.

The only other major component of the `index` prototype is the inclusion of ~15 or so other statistics in list form.  Since we want these stats to change when the user selects a new county (just like the charts), we use a bit of D3-scripting in the index file as well.  It follows much the same pattern as the charts do, with data being read in from a CSV that contains the all the extraneous stats and rankings.  **Note that some values have been hard-coded in this section to make displaying these stats a bit easier.**  A dynamic link to the printable factsheet for the selected county is also incorporated.

All said in done, the finished prototype looks like this:

![](Oklahoma%20Policy%20Institute:%20CountySTATS/Oklahoma%20Policy%20Institute:%20CountySTATS/E48D9DD3-4A23-4C75-8204-2589E66C3B3C.png)

And you can keep using this prototype page on your local machine to view any updates or tweaks before they’re published to the official OKPolicy website, and/or to save static SVG “images” with the NYT Crowbar bookmarklet if need be.

- - - -

### Credits and Other Helpful Resources

Much of the code that went into these charts was directly adapted from previous examples in D3 found on bl.ocks.org.  I’ve tried to credit all I can here.

* [Grouped Bar Chart - mbostock](https://bl.ocks.org/mbostock/3887051)
* [Multi-Series Line Chart - mbostock](https://bl.ocks.org/mbostock/3884955)
* [Line chart with dropdown selector - jhubley](http://bl.ocks.org/jhubley/17aa30fd98eb0cc7072f)
* [Interactive Donut Charts - erichoco](http://bl.ocks.org/erichoco/6694616)
* [Simple graph with grid lines in v4 - d3noob](https://bl.ocks.org/d3noob/c506ac45617cf9ed39337f99f8511218)
* [X-Value Mouseover - mbostock](https://bl.ocks.org/mbostock/3902569)
* [D3 mouseover multi-line chart - larsenmtl](https://bl.ocks.org/larsenmtl/e3b8b7c2ca4787f77d78f58d41c3da91)

I also relied on the D3.tip library, which can be viewed on GitHub here: [GitHub - Caged/d3-tip: d3 tooltips](https://github.com/Caged/d3-tip)

And finally, here are a couple of other resources that I felt I should list.  I didn’t know anything about D3 or JavaScript coming into this project; these pages helped me tremendously, and I hope they help anyone who may update this project and the CountySTATS page in the future.

* [D3.js - Data-Driven Documents](https://d3js.org/)
* [Working with Transitions](https://bost.ocks.org/mike/transition/)
* [Codecademy - JavaScript](https://www.codecademy.com/catalog/language/javascript)
* [D3 Tutorial Table of Contents | DashingD3js.com](https://www.dashingd3js.com/table-of-contents)
* And countless StackOverflow searches.

I hope this helps you, and I hope that this project can give us a better visual understanding of what kinds of problems (and successes) people are facing right now on a local level— and subsequently, statewide and nationally. Three cheers for data—

Aaron Krusniak, OKPolicy Intern, Fall 2017
#OKPolicy