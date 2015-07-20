---
layout: post
title: AngularJS Data Binding Performance Tweak
excerpt: "Some tips about finetuning angularjs data-binding for performance"
modified: 2015-06-17
tags: [javascript, angularjs, web-development]
comments: true
published: false
image:
  feature: banner.jpg
  credit: WeGraphics
  creditlink: http://wegraphics.net/downloads/free-ultimate-blurred-background-pack/
---

I have been using AngularJS framework - an MVVM client-side Javascript framework that is famous for its two-way data-binding system, for about one year now in most web application projects. During that time, the binding system has been very useful and saved me a lot of time trying to display data to the UI while also keeping the two in sync.

However, sometimes I found when I have a lot of data, is that it is getting somewhat slow. After a thorough googling around, I have founded solutions and they worked for me:

### Use `ng-repeat` with `track by`:

If every object in the array has its own ID, then just utilize it:

{% highlight html %}
    <tr ng-repeat="item in items track by item.id">
{% endhighlight %}

Usually, data from a database (even noSQL database) will most likely have ID associated with every object. However, in some cases when data is of primitive type or the objects somehow do not have a unique ID property, then let's go with:

{% highlight html %}
    <tr ng-repeat="item in items track by $index">
{% endhighlight %}

{% highlight html %}
<!-- Two lines below are the same. By default 'ng-repeat' use track by with the object identity itself. -->

<tr ng-repeat="item in items">
<tr ng-repeat="item in items track by $id(item)">
{% endhighlight %}

Speed gain is significant as seen [here][example-1].

The reason is `ng-repeat` will try to reuse elements that are unchanged, and only drop/create modified elements. This is the most common speed improvement tweak and should be applied first and foremost.

For array of primitive values, then using `track by $index` is the only choice. It lets `ng-repeat` keeps track of things by using its own internal IDs when it generates DOM elements.

For array of objects without IDs, we can have `ng-repeat` generate an ID for each object by `track by $id(item)`. This is also the default mode of `ng-repeat` whenever `track by` is not present. What does that means? That means, by default, `ng-repeat` with primitive-value array is performance optimized, while with object array it is not.

Using `track by $id(item)` is not quite good, personally speaking. Let's imagine that we have an object like this:

{% highlight javascript %}
{
    name: 'Test',
    role: 'Hello World',
    description: 'Something fancy',
    other: [1, 2, 3, 4, 5]
}
{% endhighlight %}

Inside `ng-repeat`, we only display out the propery `name` and `description`. What happen if we change that object's `other` or `role` property? Yes, `ng-repeat` with `track by $id(item)` will see that the object has been changed. It does not care or know about whether the chaged property is relavant to what are displayed or not. Once it detects a change, it drops and recreates the DOM element regardless of the fact that changed `role` or `other` has nothing to do with `name` and `description`.

However, using the above method alone is unlikely to gain much performance without the following trick, especially when the data source is an array of complex objects (If we are working with a web application then it would be very likely to have array of complex objects):

### Do not replace the whole array if only some data get changed:

Let's say our HTML will be like this, either one:

{% highlight html %}
<tr ng-repeat="item in items">

<!-- OR -->

<tr ng-repeat="item in items track by $id(item)">
{% endhighlight %}

And here is our JS:

{% highlight javascript %}
$scope.items = [];

$http.get('SAMPLE').success(function (data, status, headers, config) {
    // let's say we ajax and get back an array of data
    $scope.items = data;
});
{% endhighlight %}

Now, let's say we ajax and get back an array of data that has some changed elements and some added elements. If we simply assign the newly returned data array to the original one, `ng-repeat` will destroy and recreate all DOM elements, since some those new elements, despite being exactly the same, doesn't have the same `$$hash` that `ng-repeat` created by implicit or explicit `$id()` when enumerate a collection on binding to keep track of object/value in the array:

{% highlight javascript %}
$http.get('SAMPLE').success(function (data, status, headers, config) {
    $scope.items = data;
});
{% endhighlight %}

A better way is, of course, have objects without their own ID and use `track by item.id` so that unchanged object will not waste `ng-repeat` time.

### Do not use inline method call for calculating the data:

Do not do this:

{% highlight html %}
<div ng-repeat="cat in getCats() track by cat.id"></div>
{% endhighlight %}

Because every times digest cycle runs, it will need to do one further step of `getCats()`

Instead, let's do this:

{% highlight html %}
<div ng-repeat="cat in cats track by cat.id"></div>
{% endhighlight %}

Because it saves 1 step in the expensive digest cycle. `cats` should be an array of values or objects in `$scope`.

### Use `bindonce` for those that don't need two-way data-binding:

Most of the time, we don't need two-way data-binding for presentational information. By dropping two-way and go with one-way one-time data-binding, we save that many watchers and thus, every digest cycle runs faster.

Use of `bindonce` module with AngularJS:

{% highlight html %}
<div bindonce bo-bind="fullName"></div>
{% endhighlight %}

{% highlight html %}
<tr bindonce ng-repeat="song in songs track by song.id">
    <!-- Inside this, use "bo-*' instead of 'ng-*' or interpolation -->
    <!-- Example: -->
    <td bo-bind="song.name"></td>
    <td bo-bind="song.actor"></td>
</tr>
{% endhighlight %}

This one is one-way one-time data-binding, with one exception that the model fetch its data from a source. So when the state load, the model is empty and is only filled after a delay. Called data-source validation:

{% highlight html %}
<div bindonce="fullName" bo-bind="fullName"></div>
{% endhighlight %}

{% highlight html %}
<tr bindonce="song" ng-repeat="song in songs track by song.id"></tr>
{% endhighlight %}

Be aware that `bindone` is not included with AngularJS < 1.3, so [external library][bo] must be used. Install it using `bower`, include it then load it to angular module. And you are good to go.

For AngularJS 1.3+, a simple bindonce feature has been included:

{% highlight html %}
<div>{% raw %}{{::fullName}}{% endraw %}</div>

<!-- use with ng-repeat -->

<div ng-repeat="student in ::students track by student.id">
    <!-- bind data here, as normal -->
    <h6>{% raw %}{{student.fullName}}{% endraw %}</h6>
</div>
{% endhighlight %}

By using this expression, after the value is stabilized and bind, their watchers are deregistered and freed from memory.

### `ng-repeat` with filter? Consider a debounce:

Usually when using filters with `ng-repeat` for filtering display result (i.e search), it is almost realtime: every time a key is pressed, filters kick in and do a bunch of things. This realtime filter is very expensive.

{% highlight html %}
<div ng-repeat="cat in cats | filter:filterCats track by cat.id"></div>
{% endhighlight %}

How many of us, in most cases, will need such realtime filtering? I do not think a few seconds of delay will kill our users, yet most users when typing things to search, what they want to search is the final typed string, so why not wait a little bit and then grab the string value, then perform the search?

That is the idea of `debounce`: we introduce a delay before getting the value, discard other delay timers if at all, then perform necessary actions.

A hacky way is using `ng-change` and delay:

{% highlight html %}
<input ng-model="inputCatName" ng-change="performFilterWithDelay()"/>
<div ng-repeat="cat in cats | filter:filterCats track by cat.id"></div>
{% endhighlight %}

{% highlight javascript%}
$scope.performFilterWithDelay = function () {
    // do nothing until 350ms later
    setTimeout(function () {
        // get the input value and perform filter.
        $scope.filterCats = someFilter;
    }, 350);
}
{% endhighlight %}

A more proper way would be to make an Angular factory with this [debounce algorithm][algo], but then I have no experience on this yet, so it's up to you to explore.

[example-1]: http://speed.examples.500tech.com/ngrepeat/after/angular.html
[example-2]: http://jsfiddle.net/Warspawn/6K7Kd/
[algo]: http://unscriptable.com/2009/03/20/debouncing-javascript-methods/
[bo]: https://github.com/Pasvaz/bindonce
