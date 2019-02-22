<h1 class="post-heading">react-state-mutations</h1>
<em>February 22, 2019 - [@kddeisz](https://twitter.com/kddeisz)</em>

There are two types of objects that you can pass to the first argument to `setState` within React components. The first is an object, which will update the state to be equal to that value, as in:

```javascript
this.setState({ count: 0 });
```

The second is a function, which will be called with the current state and props, and should return an object that will then be used to set the state, as in:

```javascript
this.setState(() => ({ count: 0 }));
```

In this example, both are equivalent. However, things start to get interesting when you need access to the previous state to calculate the new state (e.g., if you were adding one to the previous count). Because React state updates [may be asynchronous](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous) and the [state updates are merged](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-are-merged), it's possible that you could end up with a race condition. For example:

```javascript
this.setState({ count: this.state.count + 1 });
this.setState({ count: this.state.count + 1 });
```

In the above example, because the updates can be asynchronous and they are merged, it's possible that the second update could be merged into the first and you would wind up with the count only being incremented by one. This doesn't happen with the function form of the argument, as in:

```javascript
this.setState(({ count }) => ({ count: count + 1 }));
this.setState(({ count }) => ({ count: count + 1 }));
```

These updates are performed in sequence, and so the `count` variable that you're pulling from the previous state is always guarunteed to be up to date.

## Mutations

An added benefit of using the function variant is that you can isolate the state mutation and reuse it in multiple places, as in:

```javascript
const incrementCount = ({ count }) => ({ count: count + 1 });

this.setState(incrementCount);
this.setState(incrementCount);
```

We can now use `incrementCount` anywhere we'd like in our application, without having to redeclare the function. This ensures a certain consistency within our application and a very tiny amount of memory reduction.

We can make these kinds of mutations even more generic by accepting the name of the field we're modifying so that it could be used for any field, as in:

```javascript
const increment = name => state => ({ [name]: state[name] + 1 });
const incrementCount = increment("count");
```

Now we can reuse `increment` throughout our application whenever anything needs incrementing, and it doesn't matter the name of the value in the state object.

## react-state-mutations

We've built a library called [react-state-mutations](https://github.com/CultureHQ/react-state-mutations) that encapsulates simple mutations like the one above into functions like `increment`. Examples for all of the below mutations can be found in the README of the repository.

### Standalone

There are "standalone" mutations like `increment` that function on just an initial value. These include:

* `decrement` - decrements a value by 1
* `increment` - increments a value by 1
* `toggle` - toggles a value between true and false

### Argument

There are also "argument" mutations that function on an initial value and an additional value for each mutation.

Within this category include mutations that work on adding and removing elements from lists:

* `append` - appends a value to a list
* `concat` - concatentates two lists
* `prepend` - prepends a value to a list

As well as mutations that modify lists:

* `filter` - filters a list
* `map` - maps over a list

Finally, there are two special mutations for specific use cases:

* `cycle` - cycles through a list of values, wrapping at the end back to the beginning
* `mutate` - mutates a nested object within the state object by using the previous value and merging it with the given value

## Combinations

One of the more powerful features of this library is that all of these mutations can be combined to perform multiple mutations using one function through the `combineMutations` function. For example, if you wanted to `toggle` a value and `increment` a count in the same mutation, you could:

```javascript
import React, { Component } from "react";
import { combineMutations, append, toggle } from "react-state-mutations";

const toggleFeature = combineMutations(
  append("eventLog"),
  toggle("featureEnabled")
);

class FeatureFlag extends Component {
  state = {
    eventLog: [],
    featureEnabled: false
  };

  handleClick = () => {
    this.setState(toggleFeature(new Date()));
  };

  render() {
    const { eventLog, featureEnabled } = this.state;

    return (
      <>
        <button type="button" onClick={this.handleClick}>
          {featureEnabled ? "Enabled" : "Disabled"}
        </button>
        <ul>
          {eventLog.map(event => {
            <li key={+event}>{event}</li>
          })}
        </ul>
      </>
    );
  }
}
```

The above component functions as a toggle and keeps track of the times that the button is clicked. `combineMutations` combines the functionality of each of the passes mutations, and passes the arguments on to the appropriate "argument" mutations.

## Hooks

Recently, we added support for React's [hooks](https://reactjs.org/docs/hooks-overview.html) by allowing our state mutations to be used as individual hooks. For example, the `increment` equivalent is `useIncrement`, as in:

```javascript
import { useIncrement } from "react-state-mutations";

const Counter = () => {
  const [count, onIncrement] = useIncrement();

  return <button type="button" onClick={onIncrement}>{count}</button>;
};
```

Each of the hooks returns a two-element array (mirroring the `useState` hook). The first element is the current value, and the second element is a function that can be called to mutate the state.

This functions similarly with "argument" mutations, as in:

```javascript
import { useAppend } from "react-state-mutations";

const ClickLog = () => {
  const [events, onAppend] = useAppend([]);

  return (
    <>
      <button type="button" onClick={() => onAppend(new Date())}>Click</button>
      <ul>
        {events.map(event => (
          <li key={+event}>{event}</li>
        ))}
      </ul>
    </>
  );
};
```

## tl;dr

We built a library called [react-state-mutations](https://github.com/CultureHQ/react-state-mutations) that handles mutating React state objects without race conditions. It leads to more code reuse and fewer bugs. We also now have support for hooks that further enhances this capability.

[← Back to home](/)
