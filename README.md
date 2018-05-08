# Oklahoma Policy Institute: CountySTATS
This project is a redesign of the five (now six) charts used on OKPolicy CountySTATS:
https://okpolicy.org/resources/countystats-2015/

These charts have been simplified, and now rely on HTML/JavaScript using the [D3.js library](https://d3js.org/) for added transitions and user interactivity.

To display these charts on a local machine, download the files into a directory of your choice and view `countystats.html` locally on your web browser.  (Downloading [Brackets](http://brackets.io/) and using its Live Preview feature with Chrome is my recommended way of doing this.)

### Updating Data Points
- - - -
All the data for these charts are held in .CSV files, which can be updated in a text editor or in Excel.

The code will take care of formatting the numbers; if editing the data points in Excel, there’s no need to convert into a specific number format (e.g., “Percentage”).  There’s also no hard-coded limit to the number of data points one chart can interpret (if a 78th county was created in Oklahoma, it could be safely added to both the .CSV files and the selector on `countystats.html` to be viewed.

### Converting Charts to an Image Format (for static/printable county factsheets)
- - - -
After any time county data has been updated, the printable, 2-page factsheets will also need to have their images of each chart updated to reflect the changes.

Each of the 6 charts here is built as an SVG, and this format can be saved and used to build the printable factsheets for each county.  The best way of doing this (that I've found) is by downloading the New York Times’ Chrome tool [SVG Crowbar](http://nytimes.github.io/svg-crowbar/).  The Crowbar bookmarklet can be used to save each chart individually, if albeit somewhat tediously.

Since the printable charts aren’t interactive, you may wish for some of the charts to automatically and statically display their values during while you save them as images.  There's a feature for that!  This can be done by locating the following line of code at the top of `countystats.html` :

```js
/* Printable Charts (false by default):
   * 
   * Setting this variable to "true" will enable a "static mode" for
   * all charts. Rather than being interactive, the line charts will
   * display static gridlines and the bar charts will display their actual
   * values above each bar. The graphs will still update if the drop-
   * down menu is changed to a different county.
   * 
   * You should enable this setting when saving the charts as SVG images
   * to be used in a printable county fact sheet. */
   
  var INTERACTIVITY = True       // <----THIS IS THE LINE TO FIND
```

and then changing the value from `True` to `False`.

### General Code Outline
- - - -
In the event any of these charts need to be modified (or a new chart added), here’s a quick rundown of how the code works to create these charts.  Again, all of this code relies heavily on the [D3.js library](https://d3js.org/); it’s very helpful to have a general understanding of how D3 works (particularly with regards to its ENTER > UPDATE > EXIT strategy) before continuing on.

I’ll start by going through the code for the charts, and then briefly talk about the strategy I used in making the `countystats.html` page to display them.

##### Creating the Charts:
Each chart has a basic HTML header that defines any of its necessary style elements, source scripts, and the overarching SVG container element.  Then, the top of every body defines an SVG element, imports any necessary scripts, and then includes a nifty little JQuery function that's responsible for resizing the chart according to the width of the browser (in tandem with some CSS).  Here's what it looks like:

```js
<!DOCTYPE html>
<head>
  <style> /* CSS style goes here */ </style>
  <meta charset="utf-8">
  <title>Population</title>
</head>

<body>
  <svg class="Population" id="chart" width="480" height="260" viewBox="0 0 480 260"
  preserveAspectRatio="xMidYMid meet"></svg>
  
  <script src="http://d3js.org/d3.v4.min.js"></script>
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
  <script>
    
    var mychart = $("#chart"),
        aspect = mychart.width() / mychart.height(),
        container = mychart.parent();
    
    $(window).on("resize", function() {
        var targetWidth = container.width();
        console.log(aspect);
        mychart.attr("width", targetWidth);
        mychart.attr("height", Math.round(targetWidth / aspect));
    })
```

The rest is all Javascript code embedded in a `<script>` element.  For the most part, each chart’s code follows the same basic outline:

###### 1. Define the SVG container.
 We’ll use [margin conventions](https://bl.ocks.org/mbostock/3019563) and use D3 to set variables for the x- and y-axes scales (as well as a coloring scale, if needed).

```js
//Define the container SVG with margin conventions
  var svg = d3.select("svg"),
      margin = {top: 20, right: 20, bottom: 15, left: 40},
      width = +svg.attr("width") - margin.left - margin.right,
      height = +svg.attr("height") - margin.top - margin.bottom -20,
      graph = svg.append("g").attr("transform", "translate(" + margin.left + "," + margin.top + ")");

  //Define the scales of the x- and y-axes
  var x0 = d3.scaleBand()
      .rangeRound([0, width])
      .paddingInner(0.1);

  var x1 = d3.scaleBand()
      .padding(0.05);

  var y = d3.scaleLinear()
      .rangeRound([height, 0]);

  //Define a coloring scale
  var z = d3.scaleOrdinal()
      .range(["#2f507f", "#972421"]);
```

Notice that we also append a `g` element to our main SVG that will fit to its margins.  We'll call this element `graph` and it will be responsible for being the main holder of the axes, legend, and other pieces of the graph.

###### 2. Open the data file using `d3.csv`.  
Depending on the chart, you may see that this method uses an optional callback function to help read in and format the data.  It will always use another callback function to build the chart itself during the following steps.

```js
//Import the raw data from CSV
  d3.csv("data.csv", numericize, function(error, data) {
    if (error) throw error;
    
    //Begin to work with the data...
```

###### 3. Create any other helper variables, set the default selection value (‘Adair County’), and set the /domains/ of the x- and y-axes.
(N.B. that this step is different than the way we previously set the _scale_).  Note also that at this point, we haven’t created any visuals inside of our original SVG container— we’ve just done a bit of math and housekeeping to make creating those visuals easier in the next step.

```js
var selection = window.parent.selection, //(default selection value)
    keys = data.columns.slice(1),
    currentKeys = ['State', selection];

//Siphon out only the data relevant to the selected county
var dataset = formatData(data, selection);

//Set the domains of the axes
x0.domain(dataset.map(function(d) {return d.Ethnicity; }));
x1.domain(currentKeys).rangeRound([0, x0.bandwidth()]);
y.domain([0, 100]).nice();
```

We'll also use a helper function to create a variable called `dataset` that holds only the information that pertains to the currently selected county.  That will make it easier for us to use this data in a moment.

And, note that we set our default selection value according to a global variable held by the parent document (`countystats.html`). By default this will be Adair County, but it could be something else if the page is loaded with a URL parameter like `okpolicy.org/countystats?county=Kingfisher`.

###### 4. Start creating visuals by appending other kinds of SVG elements (path, rect, g, etc.) to our initial container.  
We accomplish this using the D3 ENTER > UPDATE > EXIT strategy, where we can essentially create one data point (e.g., one point on a line graph or one bar in a bar graph) for each item in the array of data that we loaded in from our CSV.  That process looks something like this:

```js
//Begin appending bars to the graph:

var bars = graph.append("g")
    .selectAll("rect")
    .data(dataset)
    .enter().append("rect")
      .attr("class", "bar")
      .attr("transform", function(d,i) { 
          if (i%2 == 0){
            var offset = 3;
          } else {
            var offset = x1.bandwidth()+4;
          }
          return "translate(" + (x0(d.Ethnicity) + offset) + ",0)"; })
      .attr("x", function(d) { return x1(d.Ethnicity); })
      .attr("y", function(d, i) { return y(d.Value); })
      .attr("width", x1.bandwidth())
      .attr("height", function(d) { return height - y(d.Value); })
      .attr("fill", function(d) { return z(d.Type); })

//Continue adding elements...
```

First we use D3 to select all the elements of the type `rect`. Because we haven't created any yet, this returns an empty array.  However, we then pair this selection with data— the dataset for the selected county.  In this example, that dataset will have around 5 entries for different racial/ethnic proportions of the county, plus 5 corresponding entries for state data. When we `enter` that data and then `append("rect")` to it, our D3 selection will now hold 10 new `rect` elements, each one paired with one of the values from our dataset array.  (The ENTER > UPDATE > EXIT strategy is better explained in D3's documentation.)

When we change our county selection by switching the drop-down menu, we’ll have to update this data— and that’s what we’ll do in the next step.

###### 5. Set up an event listener that will watch for the parent document to signal when updates are needed.
The parent document (i.e., the main county stats page) will contain each of our six charts inside its own iframe.  When we select a new county, we want all six charts to update simultaneously and instantaneously.  To do this, we’ll use Javascript’s `window.postMessage()` function.

Each time the selected county is changed in the drop-down menu (or the county map), the parent document will use this function to send a message to each iframe document.  This message both acts as a signal (essentially telling the iframe, “Hey, you need to update!”) and as a message, with the message being the name of the county to update to.  The parent document might send each iframe a message saying “Tulsa County,” for example.

We will then create an event listener for each chart— specifically, we’ll use the `.onmessage` event listener, which will fire every time a document receives a message that has been sent with `window.postMessage`.  Now when we send a message that says “Tulsa County,” each chart will do two things:

	1. The `.onmessage` event will fire, saying that a message was received.  At this point, we don’t know what the message is or where it came from, just that one was received.  This will prime the code to get ready to update.
	2. The message will be verified and then read to see which county should be displayed next.  If the content of the message is “Tulsa County,” the chart will know to switch to displaying data for Tulsa County.

_The keyword above is “verified,” which is VERY IMPORTANT._  Messages can be sent and received from anywhere, which opens up a whole host of cross-site scripting vulnerabilities.  So, as soon as `.onmessage` fires and we know we’ve received a message, we’re going to check the origin of that message to make absolutely sure it’s coming from the expected sender (okpolicy.org) before moving on.  If the origin is anything other than okpolicy.org, we’ll just return right away to avoid doing anything bad.  This will keep us safe from XSS attacks (as long as the rest of the domain stays protected, too).

```js
/* Listen for a signal from the parent document to change the selection:
      
        NOTE: MESSAGE PASSING HAS INHERENT SECURITY RISKS!
        Please make sure that listeners only accept authenticated addresses (i.e., okpolicy.org).
        For more information, see the Mozilla Dev Net documentation on Window.postMessage();. */
  	window.onmessage = function(event) { 
  		//Do we trust the sender of this message?
  		if (event.origin !== "http://okpolicy.org")
      	return;
```

The implicit else case here is where all of the updating actually happens.  This varies depending on the chart, and the charts use D3 transitions to help make sure that the shift is smooth and able to be followed by the user’s eye.  In most instances, this means recalculating which slice of data from the .CSV we’re working with, and then some combination of moving plot points/bar lines and rescaling the x- and y-axes to keep the new data to a good fit.  We also update the labels on the legend here.  The process will look something like this:

```js
//Update the data:
  selection = this.options[this.selectedIndex].value;
  currentKeys[1] = selection;
  dataset = formatData(data, selection);
  legendText.data(currentKeys.slice().reverse());

//Rearrange the order of ethnicities (from greatest > least)
  x0.domain(dataset.map(function(d) { return d.Ethnicity; }));

//Associate the new selection of data with the bars
  d3.selectAll(".bar")
    .data(dataset);

//Switch the x-axis labels to the new ethnicities
  d3.select(".xaxis").call(xAxis).selectAll(".tick text").each(insertLinebreaks);

//Transition to the new values for the bars
  var t0 = graph.transition().duration(450).ease(d3.easeBounce);
  t0.selectAll(".bar")
    .attr("height", function(d) { 
      return height - y(d.Value); })
    .attr("y", function(d, i) { return y(d.Value); })
    .ease(d3.easeBounce);

//Transition the legend
  legendText.transition()
      .text(function(d) {return d;})
```

###### 6. Take care of any other extraneous needs. 
At this point the basic chart will be built, but we'll sprinkle in a few more things for added functionality/code elegance.  There will be some helper functions to extract data from the CSV, some functions that utilize `D3.tip` to control mouseover functionality, some functions that calculate the nearest value to display for the Population and Unemployment line graphs, and possibly a bit more. Generally speaking, these functions need not be worried about (and could be safely copied for the same sorts of functionality if someone wanted to create a new chart).
- - - -
##### Putting It All Together:
After creating the charts, the next thing we’d like to do is put them together so all 6 are visible on one page.  We’d also like to have a dropdown menu so users can select one of Oklahoma’s counties, and then have all 6 charts transition to data for that particular county.  (Clicking on a county on the map of Oklahoma also works.)

We'd also like to have some additional statistics for the county that's displayed, and these stats should also get updated when a new county is selected in the drop-down menu (or map).

I did all this in the `countystats.html` file, where each chart is held in its own iframe.  The code looks like this (note that the source file is just relative to my own local machine):

```html
  <div class="chart population" align="center">
    <h2>Population</h2> 
    <div class="iwrapper">
      <iframe id="population" sandbox="allow-popups allow-scripts allow-forms allow-same-origin" src="/Population/Population.html" marginwidth="0" marginheight="0" style="height:260px; width:480px" scrolling="no" align="center"> </iframe>
    </div>
  </div>
```

I made use of CSS Grid to help make all the charts and stats align nicely.  There's several CSS breakpoints as well, so that the charts and stats will resize and realign when the size of the browser is changed, or the page is viewed on mobile, making this a responsive web page.

The `countystats.html` page itself uses D3 and a similar update method as the charts to assign values and rankings to all the metrics contained in the "Additional Stats" section. It also makes use of JavaScripts `pushState` method to control URL parameters; more about that can be found on Mozilla's website: https://developer.mozilla.org/en-US/docs/Web/API/History_API

Finally, let’s review how `window.postMessage` works from the sender side.  A change function is fired every time the dropdown menu selection changes (and we can also trigger it from map changes, too).  This function passes the same message, a variable called `selection`  which holds the name of the newly-selected county, to each of the 6 chart documents.  (The elements that are grabbed by `getElementByID` in this case are the iframes that hold the charts).  We’ll add the expected recipient’s domain as a parameter as well.  This parameter can be changed to the wildcard `*` if you want to test how this works on a local machine.  (You will need to comment out our `if (event.origin != “https://okpolicy.org”)` clause in the individual chart documents as well).

```js
/* This is the main change function. This function will trigger when the map is clicked,
       and it will also fire automatically when a new county in the drop-down menu is selected. */
    $('#dropdown').change(function(d) {

      // Ensures that the selection variable has updated to the new selection:
      var selection = this.options[this.selectedIndex].value;
      
      /* Passes a message to each chart prompting it to update.
      
         NOTE: MESSAGE PASSING HAS INHERENT SECURITY RISKS!
         Please make sure that listeners only accept authenticated addresses (i.e., okpolicy.org).
         For more information, see the Mozilla Dev Net documentation on Window.postMessage();. */
      document.getElementById('income').contentWindow.postMessage(selection, "https://okpolicy.org");
      document.getElementById('education').contentWindow.postMessage(selection, "https://okpolicy.org");
      document.getElementById('employment').contentWindow.postMessage(selection, "https://okpolicy.org");
      document.getElementById('population').contentWindow.postMessage(selection, "https://okpolicy.org");
      document.getElementById('unemployment').contentWindow.postMessage(selection, "https://okpolicy.org");
      document.getElementById('demographics').contentWindow.postMessage(selection, "https://okpolicy.org");
```

The layout of everything is a little bit different on our actual website, but `countystats.html`  works great for testing as its own prototype, too.  All said and done, the entire thing should look something like this:

![Screenshot image of prototype.](https://github.com/genelewis/countySTATS/blob/master/example_image.png)


You can keep using this prototype page on your local machine to view any updates or tweaks before they’re published to the official OKPolicy website, and/or to save static SVG “images” with the NYT Crowbar bookmarklet if need be.

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

For responsivity, I relied on CSS Grid (Bootstrap would've also been a decent alternative, but for so few elements Grid works just fine).  You can read a bit more about how CSS Grid works here: [Creative Bloq - Creating a responsive layout with CSS Grid](http://www.creativebloq.com/how-to/create-a-responsive-layout-with-css-grid)

And finally, here are a couple of other resources that I felt I should list.  I didn’t know anything about D3 or JavaScript coming into this project; these pages helped me tremendously, and I hope they help anyone who may update this project and the CountySTATS page in the future.

* [D3.js - Data-Driven Documents](https://d3js.org/)
* [Working with Transitions](https://bost.ocks.org/mike/transition/)
* [Codecademy - JavaScript](https://www.codecademy.com/catalog/language/javascript)
* [D3 Tutorial Table of Contents | DashingD3js.com](https://www.dashingd3js.com/table-of-contents)
* [Window.postMessage() | Mozilla Developer API](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)

I hope this helps you, and I hope that this project can give us a better visual understanding of what kinds of problems (and successes) people are facing right now on a local level— and subsequently, statewide and nationally. Three cheers for data—

Aaron Krusniak, OKPolicy Intern, Fall 2017 / Spring 2018
#OKPolicy
