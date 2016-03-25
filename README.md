# robservable

`robservable` is an R package that brings observables to R. It provides a generic framework to create reactive widgets that can interact with each other using a shiny-like API. This will allow R users to build entire web apps with interactive widgets that can be rendered purely in a browser. 

![caution](https://cdn1.iconfinder.com/data/icons/Koloria-Icon-Set/30/Error.png)
__CAUTION: This is just an experiment at this point. The API is highly likely to change. So please DONT use it yet.__

[![Imgur](http://i.imgur.com/QbkhTzT.png)](https://www.youtube.com/watch?v=IEjYznTDRFo)


## Installation

You would need the development version of `htmlwidgets` and `robservable`

```r
devtools::install_github('ramnathv/htmlwidgets')
devtools::install_github('ramnathv/robservable')
```

## Example 1

In this example, `rangeInput` and `colorInput` communicate with `d3circles`. This interaction is enabled using an observable named `input` that updates on changes to keep the various elements in sync. Note that this API is heavily inspired by `shiny`, and is implemented using the awesome [mobservable](http://mweststrate.github.io/mobservable/) library.



```
library(robservable)
library(magrittr)
myscript <- "
  document.body.addEventListener('allwidgetsrendered', function(){
    console.log('All Widgets Rendered')
  });
"

renderMessage = function(x){
  onRender(x, "function(el, x){
    console.log(el.id + ' is rendered')
  }")
}


app1 <- App(
  colorInput('#ff0000', elementId = 'fill', width = 60),
  htmltools::h3("Reactive Widgets"),
  rangeInput(5, min = 2, max = 10, elementId = 'radius1', grid = TRUE),
  d3circles(
    radius = input('radius'),
    fill = input('fill')
  ),
  rangeInput(5, min = 2, max = 10, elementId = 'radius2', grid = TRUE),
  reactiveVar('radius', 'input.radius1() + input.radius2()')
)

app1
```



## Example 2

In this example, we introduce the idea of reactive variables, which are essentially computed from other inputs. The variable `radius` is computed to be the sum of values of the two `rangeInput` elements and is passed to `d3circles` as the radius. 

```r
app2 <- App(
  rangeInput(5, min = 2, max = 10, elementId = 'radius1'),
  rangeInput(2, min = 2, max = 10, elementId = 'radius2'),
  colorInput('#ff0000', elementId = 'fill', width = 60),
  reactiveVar('radius', 'input.radius1() + input.radius2()'),
  d3circles(
    radius = input('radius'),
    fill = input('fill'),
    width = '100%', height = 300
  )
)

app2
```


## Example 3

In this example, we hook up a select input to update the data sent to a morris line chart.

Note that we write some javascript to compute the data to be sent to the chart. 

1. First, the `jsVar` function is used to create a javascript variable named `data` that is assigned the value `dat2` (converted to JSON). 
2. Second, we use the `reactiveVar` function to create an observable named `mydata` that always evaluates to `data[input.key()]`, where `input.key` is an observable that syncs with the value chosen in the select input. 

```r
# get data
dat = reshape2::melt(t(USPersonalExpenditure))
names(dat)[1:2] = c('year', 'category')
dat$year = as.character(dat$year)
dat2 = split(dat, dat$category)
d = jsonlite::toJSON(dat2)
app5 = App(
  selectizeInput(htmlwidgets::JS(d), 'Personal Care', elementId = 'key')
)

# create app
app4 <- App(
  selectizeInput(names(dat2), 'Personal Care', elementId = 'key'),
  #selectizeInput(list('year', 'value'), 'year', elementId = 'xkey'),
  # pass dat2 as a javascript variable named "data"
  jsVar(data = dat2),
  # compute observable mydata from data using the observable input.key()
  reactiveVar('mydata', "data[input.key()]"),
  # pass observable mydata as data to the morris chart
  morris(x = 'year', y = 'value', data = input("mydata"), 
    width = '100%', height = 300)
)

app4
```

I believe that it is possible to create simple utility functions that do away with the need to write any javascript, while at the same time retaining the flexibility to write arbitrary javascript filters to benefit the power users.
