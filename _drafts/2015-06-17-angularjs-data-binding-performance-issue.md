---
layout: post
title:  "AngularJS Data Binding Performance Issue"
date:   2015-06-15 16:30:00
categories: javascript angularjs web-development
---

I have been doing my web development with the fancy AngularJS framework - an MVVM client-side Javascript framework that is famous for its two-way data-binding system, for about one year now. During that time, the binding system has been very useful and saved me a lot of time trying to display data to the UI while also keeping the two in sync.

However, sometimes I found that it's getting somewhat slow when I have a lot of data. After a thorough googling around, I have founded the solution and it worked for me:

* Use `ng-repeat` with `track by`:

{% highlight html %}
<!-- If the item has its own unique ID then we can use it -->
<tr ng-repeat="item in items track by item.id">

<!-- If we don't have item ID to use, make it! -->

<!-- Or, we can use $index -->

<tr ng-repeat="item in items track by $index">

<!-- FYI: By default and without specifying 'track by', it is like this: -->
<!-- Two lines below are the same. By default 'ng-repeat' use track by with the object identity itself. -->

<tr ng-repeat="item in items">
<tr ng-repeat="item in items track by $id(item)">
{% endhighlight %}

Speed gain is significant as seen [here][example-1]. The reason is `ng-repeat` will try to reuse elements that are unchanged, and only drop/create modified elements. This most common speed improvement is unfortunately badly documented and not auto-suggested by the framework.

However, using the above method alone is unlikely to gain much performance without the following trick, especially when the data source is an array of complex objects (If we are working with a web application then it would be very likely to have array of complex objects):

* Do not replace the whole array if only some data get changed:

{% highlight javascript %}
// let's say our HTML will be like this, either one:
// <tr ng-repeat="item in items">
// <tr ng-repeat="item in items track by $id(item)">

$scope.items = [];

$http.get('SAMPLE').success(function (data, status, headers, config) {
    // let's say we ajax and get back an array of data
    $scope.items = data;
});

$http.get('SAMPLE').success(function (data, status, headers, config) {
    // now, let's say we ajax and get back an array of data that has some changed elements and some added elements

    // if we do this, ng-repeat will destroy and recreate all DOM elements,
    // since some those new elements, despite being exactly the same,
    // doesn't have the same $$hash that ng-repeat created by implicit or explicit $id()
    // when enumerate a collection on binding to keep track of object/value in the array.
    $scope.items = data;

    // Better way?
    // Hmm, either don't use this way of ng-repeat, but the 'track by' one with the unique ID from the object itself.
    // Or,
    // We sacrifice ourselves and handle the hard part for AngularJS to sail smooth (!),
    // by manually loop through the 2 arrays and compare. Migrate changes and add new elements to $scope.items as needed.
    // This might be applicable to simple value/obj array, not complex object array
    // since the time it takes to run all loops inside loops and do comparison
    // might not compensate the ng-repeat DOM expense.
    for (var i = 0; i < data.length; i++) {
        if (data[i] !== $scope.items[i]) {
            $scope.items[i] = data[i];
            // We make the change inside Angular world, so $apply should kick in and do its job automatically.
        }
    }
});
{% endhighlight %}



* Do not use inline method call for calculating the data:

{% highlight html %}
<!-- don't do this -->
<!-- because every times digest cycle runs, it will need to do one further step of getCats() -->

<div ng-repeat="cat in getCats() track by cat.id"></div>

<!-- do this, please -->
<!-- because it saves 1 step in the expensive digest cycle -->
<!-- cats should be an [] of values or objects in the scope -->

<div ng-repeat="cat in cats track by cat.id"></div>
{% endhighlight %}

* If filter is used with ng-repeat, consider a debounce:

{% highlight html %}
<!-- don't do this -->
<!-- because every times user type/del 1 single character/keystroke,
AngularJS will perform a bunch of expensive tasks: filter, sub array creation, rebinding, redrawing UI, etc. -->

<div ng-repeat="cat in cats | filter:filterCats track by cat.id"></div>

<!-- do this, please -->
<!-- because it will waits some ms before performing the whole expensive actions bundle -->

<input ng-model="inputCatName"/>
<div ng-repeat="cat in cats | filter:filterCats track by cat.id"></div>
{% endhighlight %}

This is a simple debounce feature, written as an Angular factory. Playground [here][example-2].

{% highlight javascript %}
// a debounce function as an Angular service.

// http://unscriptable.com/2009/03/20/debouncing-javascript-methods/
// adapted from angular's $timeout code
.factory('$debounce', ['$rootScope', '$browser', '$q', '$exceptionHandler',
    function($rootScope,   $browser,   $q,   $exceptionHandler) {
        var deferreds = {},
            methods = {},
            uuid = 0;
        function debounce(fn, delay, invokeApply) {
            var deferred = $q.defer(),
                promise = deferred.promise,
                skipApply = (angular.isDefined(invokeApply) && !invokeApply),
                timeoutId, cleanup,
                methodId, bouncing = false;
            // check we dont have this method already registered
            angular.forEach(methods, function(value, key) {
                if(angular.equals(methods[key].fn, fn)) {
                    bouncing = true;
                    methodId = key;
                }
            });
            // not bouncing, then register new instance
            if(!bouncing) {
                methodId = uuid++;
                methods[methodId] = {fn: fn};
            } else {
                // clear the old timeout
                deferreds[methods[methodId].timeoutId].reject('bounced');
                $browser.defer.cancel(methods[methodId].timeoutId);
            }
            var debounced = function() {
                // actually executing? clean method bank
                delete methods[methodId];
                try {
                    deferred.resolve(fn());
                } catch(e) {
                    deferred.reject(e);
                    $exceptionHandler(e);
                }
                if (!skipApply) $rootScope.$apply();
            };
            timeoutId = $browser.defer(debounced, delay);
            // track id with method
            methods[methodId].timeoutId = timeoutId;
            cleanup = function(reason) {
                delete deferreds[promise.$$timeoutId];
            };
            promise.$$timeoutId = timeoutId;
            deferreds[timeoutId] = deferred;
            promise.then(cleanup, cleanup);
            return promise;
        }

        // similar to angular's $timeout cancel
        debounce.cancel = function(promise) {
            if (promise && promise.$$timeoutId in deferreds) {
                deferreds[promise.$$timeoutId].reject('canceled');
                return $browser.defer.cancel(promise.$$timeoutId);
            }
            return false;
        };
        return debounce;
    }
]);

// Watch the queryInput and debounce the filtering by 350 ms.
$scope.$watch('inputCatName', function(newValue, oldValue) {
    if (newValue === oldValue) { return; }
    $debounce(applyQuery, 350);
});
var applyQuery = function() {
    // do filter here
    $scope.filterCats = someFilter;
};
{% endhighlight %}

A more hack-to-go way is using `ng-change` and delay:

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

Use `bindonce` for those that don't need two-way data-binding

{% highlight html %}
<!-- 'bindone' is not included with AngularJS < 1.3,
so external library must be used: https://github.com/Pasvaz/bindonce -->
<!-- Install it using bower, include it then load it to angular module -->

<!-- then whenever we want to bind something one-way and one-time,
specify 'bindonce' along with a bindonce 'bo-*' directive we want to use, instead of 'ng-*' -->
<!-- Example: -->

<!-- This one is two-way data-binding -->

<div ng-bind="fullName"></div>

<!-- This one is one-way one-time data-binding -->


<div bindonce bo-bind="fullName"></div>

<!-- This one is one-way one-time data-binding -->
<!-- With one exception that the model fetch its data from a source.
So when the state load, the model is empty and is only filled after a delay. Called data-source validation -->

<div bindonce="fullName" bo-bind="fullName"></div>

<!-- This one is two-way data-binding ng-repeat -->

<tr ng-repeat="song in songs track by song.id"></tr>

<!-- This one is one-way one-time data-binding ng-repeat -->

<tr bindonce ng-repeat="song in songs track by song.id">
    <!-- Inside this, use "bo-*' instead of 'ng-*' or interpolation -->
    <!-- Example: -->
    <td bo-bind="song.name"></td>
    <td bo-bind="song.actor"></td>
</tr>

<!-- This one is one-way one-time data-binding ng-repeat -->
<!-- With data-source validation -->

<tr bindonce="song" ng-repeat="song in songs track by song.id"></tr>
{% endhighlight %}

AngularJS 1.3+ also provide one-time binding expression: Playground here

{% highlight html %}
<div>{{::fullName}}</div>

<!-- use with ng-repeat -->

<div ng-repeat="student in ::students track by student.id">
    <!-- bind data here, as normal -->
    <h6>{{student.fullName}}</h6>
</div>
{% endhighlight %}

By using this expression, after the value is stabilized and get bind, resources - watchers get deregistered and freed up.

Most of the time, we don't need two-way data-binding for presentational information. By dropping two-way and go with one-way one-time data-binding, we save that many watchers and thus, every digest cycle runs faster.


[example-1]: http://speed.examples.500tech.com/ngrepeat/after/angular.html
[example-2]: http://jsfiddle.net/Warspawn/6K7Kd/
