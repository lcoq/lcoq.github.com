---
layout: post
title: "Ember.js: Dig into the Container"
date: 2013-02-03 21:34
comments: true
categories: Ember.js
---

If you are a little curious, or have already took a look at the Ember.js source code, you
have probably seen then `container` package, or maybe the `__container__` property of an application.

This post tries to explain what is it, and how it works, so...here we are!

## What is a `Container` ?

As you can imagine, the purpose of a `Container` is to contain something, but what? Factories, 
ie. Ember classes, since a class is an instance factory.

Basically, we can do two things with a `Container`:

- Register factories. They are defined with a `fullName`: a type and a name, joined by a colon. 
For example, in Ember, a `PostsController` is registered in a container as `controller:posts`.

- Look up objects. If a factory was previously registered with both type and name matching
those passed to `lookup`, it returns an instance of this factory.

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

And then look up `controller:posts`. It will return a instance of the `App.PostsController` class:

```js
var postsController = container.lookup('controller:posts');
App.PostsController.detectInstance(postsController); // => true
```

In fact, factories instances are **singletons** by default, so calling two times `lookup` with 
the same name will always return the same instance:

```js
var anotherPostsController = container.lookup('controller:posts');
postsController === anotherPostsController // => true
```

Here is the most basic usage of the `Container`.
It's time to talk about relationship.

## Relationship

A container can have children: it has a method `child()` that generates a new child each time this 
method is called. But the powerful of this relationship is that a child has a parent, not children.

A container will try to find the factory in its parent if it does not register itself the factory
you're looking for.

Containers have other features like resolvers, cache and injections. They also works the same 
way with the parent container.

## Cache

Assuming a factory is registered in a container, when calling `lookup` with its name, it:

* Tries to find if an instance of this factory is stored in its registry or in its parent,
* Instantiate an object from this factory, unless instantiation is disabled (via an option).
* Store it in the cache, unless singleton option is disabled.

It implies that if you create an item with the parent container, and then try to get an item of the
factory with one of its child, you will get the object already created by its parent.

## Injections

Containers can define injections. They are a way to inject properties into factories instances.
There are two kinds of injections:

* By type - It injects properties to all factories that have the same type (e.g. for all 
controllers, those starting by `controller:`)
* By factory - In injects properties only in the factory that has the name specified

This is pretty easy to use, take a look with these factories:

```js
container.register('controller:post', App.PostController);
container.register('controller:comments, App.CommentsController');
container.register('store:main', App.Store);
```

We can define injections like this:

```js
container.typeInjection('controller', 'store', 'store:main');
container.injection('controller:post', comments, 'controller:comments');
```

The first (type) injection will set the `store` property as the `store:main` instance of each `controller` factories instances:

```js
var store1 = container.lookup('controller:post').get('store');
var store2 = container.lookup('controller:comments).get('store');
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
the containers search the factory in its registry. It will stop its search if the resolver found
something.

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

## What I did not say

### Options

Containers can register options by type, and by factory, just like injections. There are two options
at this moment:

* `singleton` - Each time you will call `lookup` for the factory which has this option set to 
`false`, a new instance is created instead of caching it.
* `instantiate` - When calling `lookup`, it returns the same object that was registered. Container
are really just containers when this option is set to `false`.

### Has

You can check if a factory exist by calling `has` on the container. Note that it also calls 
the `resolver`.

## In Ember
