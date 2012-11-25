---
layout: post
title: "Coxcomb Charts with Raphael.js"
date: 2012-11-25 18:21
comments: true
categories: 
- programming
- visualization
- raphael
---
The coxcomb chart was first used by Florence Nightingale to persuade Queen Victoria about improving
conditions in the military hospitals during the Crimean War.

{% img center /images/blog/coxcomb-nightingale.jpg 480 680 'Nightingale coxcomb' %} 

As you see it serves the same purpose as a traditional barchart, but displays the information in
a coxcomb flower pattern.

I couldn't find something already done that suited my needs, so I made one my self.

It's slightly modified from the original design, since it doesn't display the bars stacked but side by side, I think it's better
to display superposed labels that way.

**Death and Mortality**
<script type="text/javascript" src="https://raw.github.com/guilespi/resume-of-sorts/master/scripts/jquery-1.7.2.min.js"></script>
<script type="text/javascript" src="https://raw.github.com/guilespi/resume-of-sorts/master/scripts/raphael-min.js"></script>
<script type="text/javascript" src="https://raw.github.com/guilespi/coxcomb-chart/master/src/coxcomb.js"></script>
<div id="deaths"></div>
<script type="text/javascript">
  var deaths = {
     data : { 
         jan : {
            disease : 50,
            wounds : 20
         },
         feb : {
            disease : 60,
            wounds : 35
         },
         mar : {
            disease : 80,
            wounds : 50
         },
         apr : {
            disease : 70,
            wounds : 20
         },
         may : {
            disease : 30,
            wounds : 60
         },
         jun : {
            disease : 75,
            wounds : 22
         },
         jul : {
            disease : 66,
            wounds : 65
         },
         aug : {
            disease : 30,
            wounds : 14
         },
         sep : {
            disease : 50,
            wounds : 14
         },
         oct : {
            disease : 24,
            wounds : 32
         },
         nov : {
            disease : 30,
            wounds : 44
         },
         dec : {
            disease : 50,
            wounds : 5
         },
     },
     colors : {
        category : "#2B2B2B",
        opacity : 0.8,
        fontColor: "#fff",
        bySeries : {
            disease  : {
              color : "#E9E581",
              opacity : 0.8,
              fontColor: "#000",
            },
            wounds : {
              color: "#DE1B1B",
              opacity: 0.7,
              fontColor: "#fff"
            }
        }
      } 
    }; 
    var paperWidth = 600;
    var paperHeight = paperWidth * 0.8;
    var lastSelection;
    var properties = {
        categorySize : 0.20,
        categoryFontSize: paperWidth > 500 ? 10 : 6,
        seriesFontSize: paperWidth > 500 ? 10 : 6,
        onClick: function(polygon, text) {
            if (lastSelection) {
                lastSelection.remove();
            }
            lastSelection = polygon.glow();
        },
        stroke: "#fff"
    };
    Raphael("deaths", paperWidth, paperHeight)
            .coxCombChart(paperWidth / 2,paperHeight / 2, paperHeight / 2, deaths, properties);
</script>
Feeding data into the chart is straightforward

``` javascript
deaths = {
   data : { 
       jan : {
          disease : 180,
          wounds : 20
       },
       feb : {
          disease : 140,
          wounds : 35
       },
       mar : {
          disease : 80,
          wounds : 50
       },
       may : {
          disease : 40,
          wounds : 40
       },
       ...
    colors : {
        ...
    }
}
Raphael("deaths", width, height).
    coxCombChart(width / 2, height / 2, height / 2, deaths, properties);
```
I've used it to show some skills in a [resume-of-sorts][1] if you wanna see a color strategy by category and not by series.

**Lie Factor Warning**

The received values are normalized and the maximum value takes the complete radius of the coxcomb. Be warned,
each value is normalized and only the radius is affected, not the complete area of the disc sector. 
This may introduce visualization problems as the ones pointed by [Edward Tufte][3], with x10 lie factors or more, as in the following known case with a 9.4
lie factor.

{% img center /images/blog/barreltufte.jpg 280 480 'Lie factor 9.4, Edward Tufte, The Visual Display of Quantitative Information' %} 

I may fix it if someone founds this useful, the area for the formulas are on [this website][2]. The source code is [on github][4].

_Follow me on [Twitter][5]_

[1]: http://resume.guillermowinkler.com
[2]: http://understandinguncertainty.org/node/214
[3]: http://www.amazon.com/Visual-Display-Quantitative-Information/dp/0961392142
[4]: https://github.com/guilespi/coxcomb-chart
[5]: http://www.twitter.com/guilespi
