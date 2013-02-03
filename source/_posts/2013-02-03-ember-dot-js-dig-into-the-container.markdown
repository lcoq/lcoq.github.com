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
For example, in Ember, a `PostsController` is registered as `controller:posts`.

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
Let's go deeper!

## Inheritance
