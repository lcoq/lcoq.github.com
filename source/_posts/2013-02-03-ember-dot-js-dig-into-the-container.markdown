---
layout: post
title: "Ember.js: Dig into the Container"
date: 2013-02-03 21:34
comments: true
categories: Ember.js
---

If you are a little curious, or have already took a look at the Ember.js source code, you
have probably seen then `container` package, or maybe the `__container__` property of an 
application.

This post tries to explain what is it, and how it works, but its main purpose is to keep a trace of my research; you'll find these informations easily by reading its code.

## What is a `Container` ?

As you can imagine, the purpose of a `Container` is to contain something, but what? In fact, it 
could be anything, but most of the time, it will be **factories**, ie Ember classes, since a 
class is an instance factory.

Basically, we can do three things with a `Container`:

- Register factories. They are defined with a `fullName`: a type and a name, joined by a colon. 
For example, in Ember, a `PostsController` is registered in a container as `controller:posts`.

- Look up objects. If a factory was previously registered with both type and name matching
those passed to `lookup`, it returns an instance of this factory.

- Check if the factory is registered in the container.

*Note: You will see at the end of this post that containers can also just register objects instead
of factories, and return them as they were registered.*

### Example

Given a `PostsController` class, and a `container`,  defined as follows:

```js
App.PostsController = Ember.ArrayController.extend();
var container = new Container();
```

We can register the `PostsController` like this:

```js
container.register('controller:posts', App.PostsController);
```

We could verify the factory is well registered by calling the `has` method:

```js
container.has('controller:posts'); // => true
```

And then look up `controller:posts`. It will return a instance of the `App.PostsController` class:

```js
var postsController = container.lookup('controller:posts');
App.PostsController.detectInstance(postsController); // => true
```

Here is the most basic usage of the `Container`.
It's time to talk about relationship.

## Relationship

A container can have children: it has a method `child()` that generates a new child each time this 
method is called. But the powerful of this relationship is that a child has a parent, not children:

If a container does not register the factory you're looking for, it tries to **find it in its 
parent.**

Containers have other features like resolvers, singletons and injections. They also works the same 
way with the parent container.

## Singletons

Assuming a factory is registered in a container, when calling `lookup` with its name, it:

* Search in its cache or its parent cache if an instance of this factory already exist, and 
return then if so
* Else, instantiate an object from this factory, unless `instantiate` option is set to `false`
* Store it in the cache, unless `singleton` option is set to `false`

Given the `PostsController` registered in the example above, we have:

```js
container.lookup('controller:posts') ==== controller.lookup('controller:posts'); // => true
```

As the cache also works with the parent container, It implies that if you create an item with the 
parent container, and then try to get an item of the factory with one of its child, you will get 
the object already created by its parent.

## Injections

Containers can define injections. This is a way to inject properties into factories instances. 
There are two kinds of injection:

* By type - It injects properties to all factories that have the same type (e.g. for those starting
by `controller:`)
* By factory - In injects properties only in the factory that matches the name specified

This is pretty easy to use, take a look with these factories:

```js
container.register('controller:post', App.PostController);
container.register('controller:comments', App.CommentsController);
container.register('store:main', App.Store);
```

We can define injections like this:

```js
container.typeInjection('controller', 'store', 'store:main');
container.injection('controller:post', 'comments', 'controller:comments');
```

The first (type) injection will set the `store` property as the `store:main` instance of each `controller` factories instances:

```js
var store1 = container.lookup('controller:post').get('store');
var store2 = container.lookup('controller:comments').get('store');
store1 === store2 === container.lookup('store:main');
```

The second injection will set the `comments` property as the `controller:comments`
instance of the `controller:post` instances:

```js
var commentsController = container.lookup('controller:post').get('comments');
commentsController === container.lookup('controller:comments');
```

## Resolver

A container can have a `resolver`. This is basically a hook called when `lookup` is called, before
the container starts to search the factory in its registry. It will stop its search if the resolver
returned something.

It can be useful when the factory you are looking for is not defined in the container, so you can 
resolve factories lazily:

```js
var MyPostsController = Ember.ArrayController.extend();
container.resolver = function(fullName) {
  if (fullName === 'controller:posts') {
    return MyPostsController;
  }
}
```

Note that resolvers are also inherited from parent containers.

## In Ember

When you create an `Ember.Application`, it creates a container that will store all controllers, 
views, templates, routes, the router and the application itself.

### Options and injections

Some ovious options are defined when the application build this container:

* `view` factories are not singletons
* `template` cannot be instantiated
* `application:main` cannot be instantiated (this is already an instance of `Ember.Application`)

Then, some injections are defined:

* The `router:main` is injected in controllers as `target`, and in routes as `router`
* The `application:main` is injected in controllers as `namespace`

*Note: Ember-Data add other injections in an application initializer: the `store` is injected in 
controllers and routes `store`.*

### Resolver

The `Ember.Application` container resolver does two things:

* Search templates in `Ember.TEMPLATES`
* Lookup in the Application the factory that matches the fullname, eg for `type:factory`, it will
looks up `App.FactoryType`. That's how Ember conventions works!

### Add some extra

An `Ember.Application` has two public methods that are defined for interacting with the container:
`register` and `inject`. They just forward the call to the same method of the container, but they
are **public**, so they can be used by us :)

## Limitations

As controllers are singleton in Ember, some features does not work well at this time. This is the
case of the new handlebar helpers `render` and `control`.

Basically, the `control` helper render the named template in the current context using an 
instance of the same-named controller. A model can be passed to this helper.

When a `control` helper is computed, it:

* Generates a `child` of the current controllers container
* Lookup the controller and the template that matches the name passed to control in this child.

Normally, `control` could be used multiple times with the same name.

In fact, it cannot work because when the `control` is rendered a second time, lookup the
controller will return the controller which was instantiated on the first control. So the 
first model will be destroyed in favor of the last control model.

That's why there could be something new in containers: **scopes**. I suggest you to take a look at
*@ghempton* comments in [emberjs/emberjs#1968](https://github.com/emberjs/ember.js/pull/1968#issuecomment-13137494) if you want to know more.
