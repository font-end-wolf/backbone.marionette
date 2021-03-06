# Marionette.View

A `View` is a view that represents an item to be displayed with a template.
This is typically a `Backbone.Model`, `Backbone.Collection`, or nothing at all.

Views are also used to build up your application hierarchy - you can easily nest
multiple views through the `regions` attribute.

**_Note: From Marionette v3.x, `Marionette.View` replaces
`Marionette.LayoutView` and `Marionette.ItemView`._**

## Documentation Index

* [Rendering a Template](#rendering-a-template)
* [Managing an Existing Page](#managing-an-existing-page)
* [Laying out Views - Regions](#laying-out-views-regions)
  * [Class Definition](#class-definition)
    * [Specifying 'regions' as function](#specifying-regions-as-function)
    * [Regions on View Instantiation](#regions-on-view-instantiation)
  * [Managing Sub-views](#managing-sub-views)
    * [Showing a view](#showing-a-view)
    * [Accessing a child view](#accessing-a-child-view)
* [Organizing your View](#organizing-your-view)
* [Events](#events)
  * [onEvent Listeners](#onevent-listeners)
  * [Lifecycle Events](#lifecycle-events)
    * [View Creation Lifecycle](#view-creation-lifecycle)
    * [View Destruction Lifecycle](#view-destruction-lifecycle)
    * [View Creation Events](#view-creation-events)
    * [View Destruction Events](#view-destruction-events)
    * [Other View Events](#other-view-events)
  * [Binding To User Input](#binding-custom-events)
    * [Event and Trigger Mapping](#event-and-trigger-mapping)
    * [View `events`](#view-events)
    * [View `triggers`](#view-triggers)
* [Model and Collection Events](#model-and-collection-events)
  * [Model Events](#model-events)
  * [Collection Events](#collection-events)
* [Advanced View Topics](#advanced-view-topics)

## Rendering a Template

The Marionette View implements a powerful render method which, given a template,
will build your HTML from that template, mixing in model information and any
extra template context.

**Overriding `render`** If you want to add extra behavior to your view's render,
you would be best off doing your logic in the
[`onBeforeRender` or `onRender` handlers](#lifecycle-events).

To render a template, set the `template` attribute on your view:

```javascript
var Mn = require('backbone.marionette');
var _ = require('underscore');

var MyView = Mn.View.extend({
  tagName: 'h1',
  template: _.template('Contents')
});

var myView = new MyView();
myView.render();
```

[Live example](https://jsfiddle.net/marionettejs/dhsjcka4/)

For more detail on how to render templates, see the
[Template documentation](./template.md).

## Managing an Existing Page

Marionette is able to manage pre-generated pages - either static or
server-generated - and treat them as though they were generated from Marionette.

To use the existing page, set the `template` attribute to `false`:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View({
  el: '#base-element',
  template: false
});
```

[Live example](https://jsfiddle.net/marionettejs/b2yz38gj/)

Marionette will then attach all the usual
[`event`](#events-and-callback-methods) and [`ui`](#organizing-your-view)
handlers to the view using the existing HTML. Though the View has no template,
you can still listen to the `before:render` and `render` events that will fire
as usual when `render` is called - or when you execute `region.show(view)`.

### Setting a falsy `template` value

When using an existing page, Marionette explicitly looks for `false` - any other
falsy value will cause Marionette to raise an error when attempting to render
the template.

## Laying out Views - Regions

The `Marionette.View` class lets us manage a hierarchy of views using `regions`.
Regions are a hook point that lets us show views inside views, manage the
show/hide lifecycles, and act on events inside the children.

**This Section only covers the basics. For more information on regions, see the
[Regions Documentation.](./marionette.region.md)**

Regions are ideal for rendering application layouts by isolating concerns inside
another view. This is especially useful for independently re-rendering chunks
of your application without having to completely re-draw the entire screen every
time some data is updated.

Regions can be added to a View at class definition, with `regions`, or at
runtime using  `addRegion`.

When you extend `View`, we use the `regions` attribute to point to the  selector
where the new view will be displayed:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
	template: '#tpl-view-with-regions',

	regions: {
		firstRegion: '#first-region',
		secondRegion: '#second-region'
	}
});
```

If we have the following template:

```html
<script type="x-template/underscore" id="tpl-view-with-regions">
  <div id="first-region"></div>
  <div id="second-region"></div>
  <div id="third-region"></div>
</script>
```

[Live example](https://jsfiddle.net/marionettejs/4e3qdgwr/)

When we show views in the region, the contents of `#first-region` and
`#second-region` will be replaced with the contents of the view we show. The
value in the `regions` hash is just a jQuery selector, and any valid jQuery
syntax will suffice.

### Managing Sub-views

`View` provides a simple interface for managing sub-views with `showChildView`
and `getChildView`:

#### Showing a view

To show a view inside a region, simply call `showChildView(region, view)`. This
will handle rendering the view's HTML and attaching it to the DOM for you:

```javascript
var Mn = require('backbone.marionette');
var SubView = require('./subview');

var MyView = Mn.View.extend({
  template: '#tpl-view-with-regions',

  regions: {
    firstRegion: '#first-region'
  },

  onRender: function() {
    this.showChildView('firstRegion', new SubView());
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/98u073m0/)

#### Accessing a child view

To access the child view of a `View` - use the `getChildView(region)` method.
This will return the view instance that is current being displayed at that
region, or `null`:

```javascript
var Mn = require('backbone.marionette');
var SubView = require('./subview');

var MyView = Mn.View.extend({
  template: '#tpl-view-with-regions',

  regions: {
    firstRegion: '#first-region'
  },

  onRender: function() {
    this.showChildView('firstRegion', new SubView());
  },

  onSomeEvent: function() {
    var first = this.getChildView('firstRegion');
    first.doSomething();
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/b12kgq3t/)

If no view is available, `getChildView` returns `null`.

### Region availability
Any defined regions within a `View` will be available to the `View` or any
calling code immediately after instantiating the `View`. This allows a View to
be attached to an existing DOM element in an HTML page, without the need to call
a render method or anything else, to create the regions.

However, a region will only be able to populate itself if the `View` has access
to the elements specified within the region definitions. That is, if your view
has not yet rendered, your regions may not be able to find the element that
you've specified for them to manage. In that scenario, using the region will
result in no changes to the DOM.

### Efficient nested view structures
When your views get some more regions, you may want to think of the most
efficient way to render your views. Since manipulating the DOM is performance
heavy, it's best practice to render most of your views at once.

Marionette provides a simple mechanism to infinitely nest views in a single
paint: just render all of the children in the onRender callback.

```javascript
var Mn = require('backbone.marionette');

var ParentView = Mn.View.extend({
  onRender: function() {
    this.showChildView('header', new HeaderView());
    this.showChildView('footer', new FooterView());
  }
});

myRegion.show(new ParentView(), options);
```

In this example, the doubly-nested view structure will be rendered in a single
paint.

This system is recursive, so it works for any deeply nested structure. The child
views you show can render their own child views within their onRender callbacks!

### Listening to events on children

Using regions lets you listen to the events that fire on child views - views
attached inside a region. This lets a parent view take action depending on what
is happening in views it directly owns.

**To see more information about events, see the [events](#events) section**

#### Defining `childViewEvents`

The `childViewEvents` hash defines the events to listen to on a view's children
mapped to the method to call. The method will receive a `child` object
referencing the view that triggered the event.

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  regions: {
    product: '.product-hook'
  },

  childViewEvents: {
    'add:item': 'updateBasketValue'
  },

  /** Assume item is the model belonging to the child */
  updateBasketValue: function(child, item) {
    var initialValue = this.model.get('value');
    this.model.set({
      value: item.get('value') * item.get('quantity')
    });
  }
});
```

You can also directly define a function to call in `childViewEvents` like so:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  regions: {
    product: '.product-hook'
  },

  childViewEvents: {
    'add:item': function(child, item) {
      var initialValue = this.model.get('value');
      this.model.set({
        value: item.get('value') * item.get('quantity')
      });
    }
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/z8079srg/)

## Organizing your View

The `View` provides a mechanism to name parts of your template to be used
throughout the view with the `ui` attribute. This provides a number of benefits:

1. Provide a reference to commonly used UI elements
2. Cache the jQuery selector
3. Change the selector later in only one place in your view

### Defining `ui`

To define your `ui` hash, just set an object of key to jQuery selectors to the
`ui` attribute of your View:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  template: '#my-template',
  ui: {
    save: '#save-button',
    close: '.close-button'
  }
});
```

Inside your view, the `save` and `close` references will point to the jQuery
selectors `#save-button` and `.close-button` respectively.

### Accessing UI elements

To get the handles to your UI elements, use the `getUI(ui)` method:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  template: '#my-template',
  ui: {
    save: '#save-button',
    close: '.close-button'
  },

  onDoSomething: function() {
    var saveButton = this.getUI('save');
    saveButton.addClass('disabled');
    saveButton.attr('disabled', 'disabled');
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/rpa58v0g/)

As `saveButton` here is a jQuery selector, you can call any jQuery methods on
it, according to the jQuery documentation.

#### Referencing UI in events

The UI attribute is especially useful when setting handlers in the
[`events`](#view-events) and [`triggers`](#view-triggers) objects - simply use
the `@ui.` prefix:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  template: '#my-template',
  ui: {
    save: '#save-button',
    close: '.close-button'
  },

  events: {
    'click @ui.save': 'handleSave'
  },

  triggers: {
    'click @ui.close': 'close:view'
  },

  handleSave: function() {
    this.model.save();
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/f2k0wu05/)

In this example, when the user clicks on `#save-button`, `handleSave` will be
called. If the user clicks on `.close-button`, then the event `close:view` will
be fired on `MyView`.

By prefixing with `@ui`, we can change the underlying template without having to
hunt through our view for every place where that selector is referenced - just
update the `ui` object.

## Events

Firing events on views allows you to communicate that something has happened
on that view and allowing it to decide whether to act on it or not.

During the create/destroy lifecycle for a `View`, Marionette will call a number
of events on the view being created and attached. You can listen to these events
and act on them in two ways:

  1. The typical Backbone manner: `view.on('render', function() {})`
  2. Overriding the onEvent listener methods: `onRender: function() {}`

### onEvent Listeners

Marionette creates onEvent listeners for all events fired using
`view.triggerMethod('event')` - if there is an `onEvent` method, Marionette will
call it for you. An example:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  onRender: function() {
    console.log("Fired whenever view.triggerMethod('render') is called.");
  },

  onOtherEvent: function(argument) {
    console.log("Fired other:event with '" + argument + "' as an argument");
  }
});

var view = new MyView();

view.triggerMethod('other:event', 'test argument');
```

[Live example](https://jsfiddle.net/marionettejs/wb95xd3m/)

This will display in the console:
`Fired other:event with 'test argument' as an argument`

To set up handlers for events, see the rules in the
[Documentation for Events](./events.md#magic-method-binding).

### Lifecycle Events

When rendering and showing a `View`, a number of events will be fired to denote
certain stages of the creation, or destruction, lifecycle have been reached.
For a full list of events, and how to use them, see the
[documentation for `View` lifecycle events](./viewlifecycle.md#view-lifecycle).

### Binding To User Input

Views can bind custom events whenever users perform some interaction with the
DOM. Using the view `events` and `triggers` handlers lets us either bind  user
input directly to an action or fire a generic trigger that may or may not be
handled.

#### Event and Trigger Mapping

The `events` and `triggers` attributes bind DOM events to actions to perform on
the view. They each take a DOM event key and a mapping to the handler.

We'll cover a simple example:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  events: {
    'click a': 'showModal',
    'click @ui.save': 'saveForm'
  },

  triggers: {
    'click @ui.close': 'cancel:entry'
  },

  ui: {
    save: '.btn-save',
    close: '.btn-cancel'
  },

  showModal: function() {
    console.log('Show the modal');
  },

  saveForm: function() {
    console.log('Save the form');
  }
});
```

Event listeners are constructed by:

```javascript
'<dom event> <dom node>': 'listener'
```

The `dom event` can be a jQuery DOM event - such as `click` - or another custom
event, such as Bootstrap's `show.bs.modal`.

The `dom node` represents a jQuery selector or a `ui` key prefixed by `@.`. This
must exist inside the view once it has completed rendering. For more information
about the `ui` object, and how it works, see 
[the documentation on ui](#organizing-your-view).

#### View `events`

The view `events` attribute binds DOM events to functions or methods on the
view. The simplest form is to reference a method on the view:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  events: {
    'click a': 'showModal'
  },

  showModal: function(event) {
    console.log('Show the modal');
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/jfxwtmxj/)

The DOM event gets passed in as the first argument, allowing you to see any
information passed as part of the event.

**When passing a method reference, the method must exist on the View.**

The `events` attribute can also directly bind functions:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  events: {
    'click a': function(event) {
      console.log('Show the modal');
    }
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/obt5vt09/)

As when passing a string reference to a view method, the `events` attribute
passes in the `event` as the argument to the function called.

#### View `triggers`

The view `triggers` attribute binds DOM events to Marionette View events that
can be responded to at the view or parent level. For more information on events,
see the [events documentation](./events.md). This section will just
cover how to bind these events to views.

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  triggers: {
    'click a': 'link:clicked'
  },

  onLinkClicked: function() {
    console.log('Show the modal');
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/exu2s3tL/)

When the `a` tag is clicked here, the `link:click` event is fired. This event
can be listened to using the
[Magic Method Binding](./events.md#magic-method-binding) technique
discussed in the [events documentation](./events.md).

The major benefit of the `triggers` attribute over `events` is that triggered
events can bubble up to any parent views. For a full explanation of bubbling
events and listening to child events, see the
[event bubbling documentation](./marionette.events#child-view-events).

## Model and Collection events

The Marionette View can bind to events that occur on attached models and
collections - this includes both [standard][backbone-events] and custom events.

### Model Events

For example, to listen to a model's events:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  modelEvents: {
    'change:attribute': 'actOnChange'
  },

  actOnChange: function(value, model) {
    console.log('New value: ' + value);
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/auvk4hps/)

The `modelEvents` attribute passes through all the arguments that are passed
to `model.trigger('event', arguments)`.

The `modelEvents` attribute can also take a
[function returning an object](basics.md#functions-returning-values).

#### Function callback

You can also bind a function callback directly in the `modelEvents` attribute:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  modelEvents: {
    'change:attribute': function() {
      console.log('attribute was changed');
    }
  }
})
```

[Live example](https://jsfiddle.net/marionettejs/zaxLe6au/)

### Collection Events

Collection events work exactly the same way as [`modelEvents`](#model-events)
with their own `collectionEvents` key:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  collectionEvents: {
    sync: 'actOnSync'
  },

  actOnSync: function(collection) {
    console.log('Collection was synchronised with the server');
  }
});
```

[Live example](https://jsfiddle.net/marionettejs/7qyfeh9r/)

The `collectionEvents` attribute can also take a
[function returning an object](basics.md#functions-returning-values).

Just as in `modelEvents`, you can bind function callbacks directly inside the
`collectionEvents` object:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  collectionEvents: {
    'update': function() {
      console.log('the collection was updated');
    }
  }
})
```

[Live example](https://jsfiddle.net/marionettejs/ze8po0x5/)

### Listening to both

If your view has a `model` and `collection` attached, it will listen for events
on both:

```javascript
var Mn = require('backbone.marionette');

var MyView = Mn.View.extend({
  modelEvents: {
    'change:someattribute': 'changeMyAttribute'
  },

  collectionEvents: {
    update: 'modelsChanged'
  },

  changeMyAttribute: function() {
    console.log('someattribute was changed');
  },

  modelsChanged: function() {
    console.log('models were added or removed in the collection');
  }
})
```

[Live example](https://jsfiddle.net/marionettejs/h9ub5hp3/)

In this case, Marionette will bind event handlers to both.

## Advanced View Topics

This method is used to convert a View's `model` or `collection`
into a usable form for a template.

Item Views are called such because they process only a single item
at a time. Consequently, only the `model` **or** the `collection` will
be serialized. If both exist, only the `model` will be serialized.

By default, models are serialized by cloning the attributes of the model.

Collections are serialized into an object of this form:

```javascript
{
  items: [modelOne, modelTwo]
}
```

where each model in the collection will have its attributes cloned.

The result of `serializeData` is included in the data passed to
the view's template.

Let's take a look at some examples of how serializing data works.

```javascript
var myModel = new MyModel({foo: "bar"});

new MyView({
  template: "#myItemTemplate",
  model: myModel
});

MyView.render();
```

```html
<script id="myItemTemplate" type="template">
  Foo is: <%= foo %>
</script>
```

[Live example](https://jsfiddle.net/marionettejs/brp0t7pq/)

If the serialization is a collection, the results are passed in as an
`items` array:

```javascript
var myCollection = new MyCollection([{foo: "bar"}, {foo: "baz"}]);

new MyView({
  template: "#myCollectionTemplate",
  collection: myCollection
});

MyView.render();
```

```html
<script id="myCollectionTemplate" type="template">
  <% _.each(items, function(item){ %>
    Foo is: <%= foo %>
  <% }); %>
</script>
```

[Live example](https://jsfiddle.net/marionettejs/yv3hrvkf/)

If you need to serialize the View's `model` or `collection` in a custom way,
then you should override either `serializeModel` or `serializeCollection`.

On the other hand, you should not use this method to add arbitrary extra data
to your template. Instead, use [View.templateContext](./template.md#templatecontext).
