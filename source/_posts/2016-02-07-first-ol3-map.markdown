---
layout: post
title: "OL3 Element Using KnockoutJs"
date: 2016-02-07 23:25:10 -0500
comments: true
categories: OL3, KnockoutJs
---

<iframe
  width="100%"
  height="300"
  src="//jsfiddle.net/timh06/8030x8ms/embedded/result,js,html/"
  allowfullscreen="allowfullscreen"
  frameborder="0"
></iframe>

If you've used something like polymer or knockout before, you've seen HTML5
custom components, they are really cool.  The goal here is to create something
like this in your markup, for this we're going to use Knockout custom
components:

{% codeblock lang:html %}
<ol3map></ol3map>
{% endcodeblock %}

Then use the `KnockoutJs` simple custom component system to create the OL3
map.  If you haven't done so already, take a look over the custom component
section on [Knockout Components](http://knockoutjs.com/documentation
/component-overview.html).  Let's start by creating the component
registration:

{% codeblock lang:js %}
ko.components.register('ol3map', {
  viewModel: {
    createViewModel: function (params, componentInfo) {
      return new MapViewModel(params, componentInfo.element);
    }
  },
  template: '<!-- -->'
});
{% endcodeblock %}

We want to then create our new `MapViewModel` object, this will be the context
that will be bound to the template we provide.  This is important, since OL3
won't initialize on hidden elements, we'll have to do some additional work to
get it to show up.

{% codeblock lang:js %}
function MapViewModel (options, element) {
  this.map = new ol.Map({
    view: new ol.View({
      center: [0, 0],
      zoom: 2
    })
  });
}
{% endcodeblock %}

In the `MapViewModel` we want to create a new `ol.Map` object with *no* target
set yet.  We'll do that in a custom binding handler that will do the job of
connecting the two.

Now let's set up the custom binding handler:

{% codeblock lang:js %}
ko.bindingHandlers.__map = {
    init: function(element, valueAccessor, allBindings, viewModel, bindingContext) {
        viewModel.map.setTarget(element);
    }
};
{% endcodeblock %}

This binding handler will allow us to do something like
`<div data-bind="__map"></div>` and get back the element and the viewmodel.
Once it's called we know the element exists, so we can attach our map at that
point. Also, prepending with `__` helps to keep us from possibly overwriting
other handlers.  Finally, let's change our template to include the new
binding.

{% codeblock lang:js %}
ko.components.register('ol3map', {
  viewModel: {
    createViewModel: function (params, componentInfo) {
      return new MapViewModel(params, componentInfo.element);
    }
  },
  template: '<div data-bind="__map"></div>'
});
{% endcodeblock %}

That's it! now when we place a `<ol3map></ol3map>` tag somewhere in our page
Knockout will call our viewmodel code, which will create the object containing
that `map` property.  When the template is applied, it is bound to the new
viewmodel context which should be something like `{ map: ol.Map(...) }`, then
the `__map` binding handler is called with this object passed as the
`viewmodel` param.  Finally, we call `setTarget(element)` on the map object we
got from the passed in viewmodel and the map gets initialized.  The final code
looks like this:

{% codeblock lang:js %}
ko.bindingHandlers.__map = {
    init: function(element, valueAccessor, allBindings, viewModel, bindingContext) {
        viewModel.map.setTarget(element);
    }
};

function MapViewModel (options, element) {
  this.map = new ol.Map({
    view: new ol.View({
      center: [0, 0],
      zoom: 2
    }),
    layers: [
      new ol.layer.Tile({ source: new ol.source.OSM() })
    ]
  });
}

// Create the custom component
ko.components.register('ol3map', {
  viewModel: {
    createViewModel: function (params, componentInfo) {
      return new MapViewModel(params, componentInfo.element);
    }
  },
  template: '<div data-bind="__map"></div>'
});

ko.applyBindings({});
{% endcodeblock %}