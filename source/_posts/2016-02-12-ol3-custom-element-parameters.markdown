---
layout: post
title: "OL3 Custom Element: Parameters"
date: 2016-02-12 22:28:24 -0500
comments: true
categories: OL3, KnockoutJs
---

Now that we have our custom element created and the bindings are set up we can start to add in parameters to pass options to our new map.  Currently Knockout provides this utility for passing params to custom components: `<component params="{...}"></component>`.  This is fine, but it would be cool to provide our options through element attributes.

{% codeblock lang:html %}
<ol3map params="center: [0, 0], zoom: 2"></ol3map>
{% endcodeblock %}

Params are certainly useful, but I think something like this seems more intuitive:

{% codeblock lang:html %}
<ol3map center="[0, 0]" zoom="2"></ol3map>
{% endcodeblock %}

We'll want to add in support for these attributes and pass these options along to our map.  This should be easy enough to do.  Just parse out these attributes and pass the options.  Here is a simple example of an attribute parser with default options:

{% codeblock lang:js %}
var parseOptions = function(options, element) {
  if (options.$raw) {
    delete options.$raw;
  }
  Array.prototype.slice.call(element.attributes).forEach(function(k, i, a) {
    // Skip params attribute 
    if (a[i].name === 'params') {
      return;
    }
    var v = a[i].value;
    if (v.match(/^\(/)) {
      options[a[i].name] = eval(v);
    } else if (v.match(/^[\[\{\d]/)) {
      options[a[i].name] = JSON.parse(v);
    } else {
      options[a[i].name] = v;
    }
  });
  return options;
};
{% endcodeblock %}

Since `element.attributes` is actually a
[NamedNodeMap](//developer.mozilla.org/en-US/docs/Web/API/NamedNodeMap) which,
similar to the `attributes` object can be converted to an array.  We can
convert the `element.attributes` object to an array by doing a `slice` and
looping through each attribute, attaching each to the `options` object as we
go.  Finally, extend the options object using some defaults we added.  Also,
to make sure that we get the right values we'll need to watch out for things
to parse.  So for example when we have `sourceProjection="EPSG:4326"` the
quotes are stripped and the flat string is sent to `JSON.parse` and it doesn't
know what to do with it yeilding this helpful message: `Uncaught SyntaxError:
Unexpected token E`.  So let's look out for parseable strings, for example:

{% codeblock lang:js %}
// For now I am just eval()-ing things that start with "("
// definetly bad practice, but for this example it should be fine
if (v.match(/^\(/)) {
  // eval() the result
  // this way (Infinity), for example, doesn't end up as a string "(Infinity)"
}
v.match(/^[\[\{\d]/))
// Match on arrays, objects, or numbers
{% endcodeblock %}

Okay! so now we can do something like this:

{% codeblock lang:html %}
<ol3map
  params="
    center: [-40, 40],
    zoom: 2
  "
  rotation="90"
  sourceProjection="EPSG:4326
"></ol3map>
{% endcodeblock %}

and everything should work fine.  Here you can see it working great with our new customizable map element:

<iframe width="100%" height="300" src="//jsfiddle.net/timh06/x5mmyb1x/embedded/result,html,js/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

Now, what about one of the most important features of maps: layers.  We certainly want to be able to give the ability to add layers, and something like this isn't a terrible idea:

{% codeblock lang:html %}
<ol3map
  center="[-79, 40]"
  rotation="90"
  layers="[
    { type: 'Tile', title: 'Base Layer', source: 'OSM' },
    { type: 'Vector', title: 'Feature Layer', source: 'myLayers' }
  ]"
></ol3map>
{% endcodeblock %}

But I think it makes a lot of sense to have these layers be represented visually in the markup by full sub-components under the main map, for example:

{% codeblock lang:html %}
<ol3map center="[-79, 40]" rotation="90">
  <ol3layer type="Tile" title="Base Layer" source="OSM"></ol3layer>
  <ol3layer type="Tile" title="Feature Layer" source="myLayers"></ol3layer>
</ol3map>
{% endcodeblock %}

This makes our code a little bit more complicated though now as we are going to have to figure out how to handle bindings on the child elements.  The simple part, however, is registering the component - we can do that the same way we made the `<ol3map>` element.  Let's take a look at the `componentInfo.templateNodes` property, this is just DOM nodes that Knockout found between our custom component tags which it passes to us, all we have to do is place them back into the DOM and they'll be bound by Knockout.  Which means that if we have a custom component inside of our custom component we can get it to bind as well.  The simplest way to do this, I've found, is to just use the template nodes directly in the binding handler.  For example:

## Create Custom Binding (Root)
{% codeblock lang:js %}
ko.bindingHandlers.__map = {
    init: function(element, valueAccessor, allBindings, vm, context) {
        $(element).html(vm.templateNodes);
        vm.viewModel.map.setTarget(element);
    }
};
{% endcodeblock %}

When we get passed the template nodes, just fill the html back in with the nodes, this will make Knockout rebind them.  We can then do any additional work here.

## Create Custom Component (Root)
{% codeblock lang:js %}
ko.components.register('ol3map', {
  viewModel: {
    createViewModel: function (params, componentInfo) {
      return {
        viewModel: new MapViewModel(params, componentInfo.element),
        templateNodes: componentInfo.templateNodes
      };
    }
  },
  template: '<div data-bind="__map"></div>'
});
{% endcodeblock %}

Return an object containing the `viewmodel` and the `templateNodes`, then make sure the template has custom binding handler being called.

Now, assuming we have some child nodes containing custom components, we'll want to do the same thing for each of those:

## Create Custom Binding (Child)
{% codeblock lang:js %}
ko.bindingHandlers.__layer = {
    init: function(element, valueAccessor, allBindings, vm, context) {
        $(element).html(vm.templateNodes);
        var map = context.$parent.viewModel.map;
        map.addLayer(vm.viewModel);
    }
};
{% endcodeblock %}

Same thing as the `root`, the only difference is that if you need access to the parent (`root`) viewmodel, we can do that through the `context.$parent` property.  Then it's pretty simple really to do our hookup, also notice we're passing our `templateNodes` through in case we have more child components.

## Create Custom Component (Child)
{% codeblock lang:js %}
ko.components.register('ol3layer', {
  viewModel: {
    createViewModel: function (params, componentInfo) {
      return {
        viewModel: new LayerViewModel(params, componentInfo.element),
        templateNodes: componentInfo.templateNodes
      };
    }
  },
  template: '<div data-bind="__layer"></div>'
});
{% endcodeblock %}

Now we should be able to do something like this:

{% codeblock lang:html %}
<ol3map center="[-79, 40]" zoom="12">
  <ol3layer title="Base Layer" source="OSM"></ol3layer>
</ol3map>
{% endcodeblock %}

That's great, but what about the case where we would like to pass options to that source.  Could use something like `sourceOptions`, but that would break up the clean look of these attributes, so let's keep digging deeper.

{% codeblock lang:html %}
<ol3map center="[-79, 40]" zoom="12">
  <ol3layer title="Base Layer">
     <ol3source source="OSM"></ol3source>
  </ol3layer>
</ol3map>
{% endcodeblock %}

I think this is a clean way of visually representing our map.  By the way, the viewmodels for each of these are relatively simple, for this demo I'm really just passing things right through to OL3.  For an example of this, check out the `SourceViewModel`:

{% codeblock lang:js %}
function SourceViewModel (options, element) {
  options = ko.utils.extend({
    source: 'OSM'
  }, parseOptions(options, element));
  
  var source = options.source;
  delete options.source;
  
  return new ol.source[source](options);
}
{% endcodeblock %}

That `parseOptions` part takes that `options` object and adds in any of the attributes it found on the element passed in.  Then we just create a new source object, passing in options - pretty simple. 

The final thing to worry about is `Features`, these are part of `source`, and more specifically, `ol.source.Vector`.  So we are looking for something like this:

{% codeblock lang:html %}
<ol3layer title="feature layer" type="Vector">
  <ol3source source="Vector">
    <ol3feature geometry="Point" location="[-79, 40]"></ol3feature>
    <ol3feature geometry="Point" location="[-79, 40.009]"></ol3feature>
    <ol3feature geometry="Point" location="[-79, 40.005]"></ol3feature>
  </ol3source>
</ol3layer>
{% endcodeblock %}

This is done the same way as the others, just create a custom component for the new `ol3feature` tag:

{% codeblock lang:js %}
ko.components.register('ol3feature', {
  viewModel: {
    createViewModel: function(params, componentInfo) {
      return new FeatureViewModel(params, componentInfo.element);
    }
  },
  template: '<div data-bind="__feature"></div>'
});
{% endcodeblock %}

Add the feature to the parent source from within the binding handler and we're good: 

{% codeblock lang:js %}
ko.bindingHandlers.__feature = {
    init: function(element, valueAccessor, allBindings, vm, context) {
        var source = context.$parent.viewModel;
        source.addFeature(vm);
    }
};
{% endcodeblock %}

Cool! now the final markup looks like this: 

{% codeblock lang:html %}
<ol3map center="[-79, 40]" zoom="12">
  <ol3layer title="Base Layer">
     <ol3source source="OSM"></ol3source>
  </ol3layer>
  <ol3layer title="feature layer" type="Vector">
    <ol3source source="Vector">
      <ol3feature geometry="Point" location="[-79, 40]"></ol3feature>
      <ol3feature geometry="Point" location="[-79, 40.009]"></ol3feature>
      <ol3feature geometry="Point" location="[-79, 40.005]"></ol3feature>
    </ol3source>
  </ol3layer>
</ol3map>
{% endcodeblock %}

I think this is a nice clear concise way of seeing the structure of the map, the layers will be added top to bottom and all the options are right there in the attributes.  Obviously this is incomplete, we don't have anyway to really pass styling to the features or layer currently.  But this works great when you want to quickly make some maps with basic settings.

Here is a working example:

<iframe width="100%" height="600" src="//jsfiddle.net/timh06/ma5j990n/embedded/result,js,html/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>