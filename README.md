# Refactoring Our Code Into A Library

## Objectives

In this lesson, we will learn how to turn our code into a library that can be
used across JavaScript applications. By the end of the lesson you will be able
to:

* Understand which part of our codebase can be used across applications.
* Understand how to encapsulate the functions we built.
* Learn about the `getState` method and how it works.

Use `js/createStore.js` to follow along. Open `index.html` to try out the code.

## Encapsulate our application's state by wrapping our code in a function

Let's look at the code that we wrote in the last section.

```javascript
let state;

function reducer(state = { count: 0 }, action) {
  switch (action.type) {
    case 'INCREASE_COUNT':
      return { count: state.count + 1 };

    default:
      return state;
  }
};

function dispatch(action){
  state = reducer(state, action);
  render();
};

function render() {
  let container = document.getElementById('container');
  container.textContent = state.count;
};

dispatch({ type: '@@INIT' })
let button = document.getElementById('button');

button.addEventListener('click', () => {
    dispatch({ type: 'INCREASE_COUNT' });
})
```

See that `state` variable state all the way at the top of our code? Remember,
that variable holds a representation of all of our data we need to display. So
it's not very good if this variable is global, and we can accidentally overwrite
simply by writing `state = 'bad news bears'` somewhere else in our codebase.
Goodbye state.

We can solve this by wrapping our state in a function.

```javascript
function() {
  let state;
}
// ...

function dispatch(action) {
  state = reducer(state, action);
  render();
};

function render() {
  let container = document.getElementById('container');
  container.textContent = state.count;
};
```

Now if we dispatch our initial action by calling `dispatch({ type: '@@INIT' })`
our code breaks because the `dispatch` function does not have access to that
declared state. Notice that `render` won't have access to our state either. At
this point, we're tempted to move everything inside of our new function. Let's
try to figure out exactly what we should move inside the function in the next
section.

## Move Code Common to Every JavaScript Application Inside Our New Function

We ultimately want our new function to become a function that all of our
applications following the __Redux__ pattern can use. To decide what our new
function should be able to do, let's go back to our __Redux__ fundamentals.

`Action -> Reducer -> New State.`

The function that goes through this flow for us is the `dispatch` function. We
call `dispatch` with an action, and it calls our reducer and returns to us a new
state. So we move dispatch inside of our new method that now both encapsulates
the state and holds `dispatch`.

```javascript
function() {
  let state;
  // state is now accessible to dispatch

  function dispatch(action) {
    state = reducer(state, action);
    render();
  }
}
```

> Note: You may notice that in the above code we made a *closure*. As you surely
remember a JavaScript function has access to all the variables that were in
scope at the time of its definition. This feature is called a closure since a
function encloses or draws a protective bubble around the variables in its scope
and carries those with it when invoked later.

As you see above, `dispatch` is now private to our new function. But we'll need
to call the function when certain events happen in our application (eg. when a
user clicks on a button, call dispatch). So we expose the method by having our
function return a JavaScript object that has a `dispatch` method. We'll call
this returned JavaScript object our **store**, and, therefore, we'll call the
method `createStore`, because that's what it does.

```javascript
function createStore() {
  let state;

  function dispatch(action) {
    state = reducer(state, action);
    render();
  };

  return { dispatch };
};
```

This code almost works. Call `createStore`, and set the returned store equal to
a variable. Then change our remaining code to make sure we are properly
referencing the `dispatch` method. We can use the almost-working code in the
following manner.

```javascript
let store = createStore();
store.dispatch({ type: '@@INIT' });
```

So we have this object called a store which contains all of our application's
state. Right now we can dispatch actions that modify that state, but we need
some way to retrieve data from the store. To do this, our store should respond
to one other method, `getState`. This method simply returns the state, which we
can use elsewhere in our application.

```javascript
function createStore() {
  let state;

  function dispatch(action) {
    state = reducer(state, action);
    render();
  }

  function getState() {
    return state;
  }

  return {
    dispatch,
    getState
  };
};
```

Now we can get our code working by changing render to the following:

```javascript
function render() {
  let container = document.getElementById('container');
  container.textContent = store.getState().count;
};
```

...and then updating our button event listener to use `store.dispatch`:

```js
let button = document.getElementById('button');

button.addEventListener('click', () => {
    store.dispatch({ type: 'INCREASE_COUNT' });
})
```

All in all, with these changes, the code should look like the following:

```js
function createStore() {
  let state;

  function dispatch(action) {
    state = reducer(state, action);
    render();
  }

  function getState() {
    return state;
  }

  return {
    dispatch,
    getState
  };
};

function reducer(state = { count: 0 }, action) {
  switch (action.type) {
    case 'INCREASE_COUNT':
      return { count: state.count + 1 };

    default:
      return state;
  }
};

function render() {
  let container = document.getElementById('container');
  container.textContent = store.getState().count;
};

let store = createStore();
store.dispatch({ type: '@@INIT' });
let button = document.getElementById('button');

button.addEventListener('click', () => {
    store.dispatch({ type: 'INCREASE_COUNT' });
})
```

Our code is back to working. And it looks like we have a function called
`createStore` which can work with any JavaScript application...almost.

## Abstract away the reducer

We know that __Redux__ works by having an action dispatched, which calls a
reducer, and then renders the view. Our `createStore`'s dispatch method does
that.

```javascript
function dispatch(action) {
  state = reducer(state, action);
  render();
};
```

Notice, however, that we did not move the `reducer` function into the
`createStore` function. Take a look at it. _This code_ is particular to our
application.

```javascript
function reducer(state = { count: 0 }, action) {
  switch (action.type) {
    case 'INCREASE_COUNT':
      return { count: state.count + 1 };

    default:
      return state;
  }
};
```

We happen to have an application that increases a count. But we can imagine
applications that manage people's songs, their GitHub repositories, or their
contacts. So we want our `dispatch` method to call a reducer every time an
action is dispatched. However, we don't want the `createStore` function to
specify what that reducer is, or what it does. We want `createStore` to be
generic enough for _any_ JavaScript application. Instead, we should make the
reducer an argument to our `createStore` function. Then we pass through our
reducer function when invoking the `createStore` method.

```javascript
function createStore(reducer) {
  let state;

  function dispatch(action) {
    state = reducer(state, action);
    render();
  }

  function getState() {
    return state;
  };

  return {
    dispatch,
    getState
  };
};

function reducer(state = { count: 0 }, action) {
  switch (action.type) {
    case 'INCREASE_COUNT':
      return { count: state.count + 1 };

    default:
      return state;
  }
}


function render() {
  let container = document.getElementById('container');
  container.textContent = store.getState().count;
};

let store = createStore(reducer) // createStore takes the reducer reducer as an argument
store.dispatch({ type: '@@INIT' });
let button = document.getElementById('button');

button.addEventListener('click', () => {
  store.dispatch({ type: 'INCREASE_COUNT' });
});
```

As you see above, `createStore` takes the reducer as the argument. This sets the
new store's reducer as `reducer`. When an action is dispatched, it calls the
reducer that we passed through when creating the store.

## Summary

With this set up, we've got a fully functional `store`, that encapsulates our
state and provides a controlled way to write (`dispatch`) and retrieve
(`getState`) information.  

Every piece of code that would be common to any JavaScript application following
this pattern is wrapped inside of the `createStore` function. Any code that is
particular to our application is outside that function.

What's particular to a specific application?

* How the DOM is updated in our `render` function
* What events trigger a dispatch method
* How our state should change in response to different actions being dispatched.  

These are all implemented outside of our `createStore method`. What is generic
to each application following this pattern?

* That a call to `dispatch` should call a reducer, reassign the state, and render a change.

This is implemented inside the `createStore` method.

<p class='util--hide'>View <a href='https://learn.co/lessons/redux-create-store'>Redux Create Store</a> on Learn.co and start learning to code for free.</p>
