# redux-refiner
redux-refiner take heavy inspiration from the two most famous and used redux middleware that solve the thorny problem of side effects in redux: [redux-thunk](https://github.com/gaearon/redux-thunk) and [redux-saga](https://github.com/redux-saga/redux-saga).
Both are definitely good, but neither suited completely my needs and thats how I came up with this simple yet effective solution.

I liked very much the simplicity and flexibility of redux-thunk, but not so much the fact that a good amount of logic will be sprinkled on the action creators.

redux-saga instead has a much more centric approach and even if I like the idea of using generators in this clever way, it felt too inflexible for me, it wasn't straightforward to abstract patterns, like the common dispatching of `LOADING` followed by either `SUCCESS` or `ERROR`.

## Install
```sh
npm install --save redux-refiner
```

## Perks in a glance
* Centric approach for all business logic
* Separation of concerns
* Easy to abstract patterns, the `refiner` is just a function
* Easy to integrate with other libraries (eg: manage your async flow with Rx.js)
* Impure actions are just action objects, hence loggable and provide a valuable insight
* Easy to test, but require some effort depending on the side effect
* Adapts to your coding style
* Nearly the same logic as the reducer and not much to learn
* Crazy small

## Why *refiner*?
As the `reducer` reduce a new state from an action, the `refiner` refine (lift up) all *impurities* (side effects) from a given action and *produce* (dispatch) one or more *pure actions*.

```
Impure Action with side effects -> refiner -> Pure Action -> reducer -> Clean State update
```

## Show me some code!
The library itself is crazy small, so lets dive directly in an example: the famous Counter with async actions!

**redux-refiner** is render-agnostic, so we will skip the Counter component itself and how to render it.

Let's start with what we know, the `reducer`:
```javascript
import * as A from './actions';

export function reducer(state, action) {
  switch(action.type) {
    case A.INCR:
      return state + 1;

    case A.DECR:
      return state - 1;
  }
)
```
As you can see, the `reducer` will handle only the *pure actions* and it's as you would expect.

Now lets introduce the `refiner`, which instead will handle the *impurities*, and put it in the same dir of the reducer (or even same file):
```javascript
import * as A from './actions';

export function refiner(dispatch, action, state) {
  switch(action.type) {
    case A.ASYNC_INCR:
      setTimeout(() => dispatch(A.increase),
                 1000);
      return;

    case A.ASYNC_DECR:
      setTimeout(() => dispatch(A.decrease),
                 1000);
      return;
  }
}
```
The structure is identical to a reducer, with the only difference that it also receive a `dispatch` and cannot alter the state.

### Compare with saga & refactoring
Follow a solution provided by [redux-saga](https://github.com/redux-saga/redux-saga/blob/master/examples/counter-vanilla/index.html):
```javascript
function* incrementAsync() {
  yield effects.call(delay, 1000);
  yield effects.put({type: 'INCREMENT'});
}
// I've introduced the decrement for demonstration purpose
function* decrementAsync() {
  yield effects.call(delay, 1000);
  yield effects.put({type: 'DECREMENT'});
}
function* counterSaga() {
  yield effects.takeEvery('INCREMENT_ASYNC', incrementAsync);
  yield effects.takeEvery('DECREMENT_ASYNC', decrementAsync);
}
```

Lets be honest and say that this feels cleaner than the `refiner` version, however the careful reader would have probably spotted the code repetition in both version.  
Suppose that we can't modify the actions nor the flow, how would you refactor the saga version?  
I don't know a clean way to do it (and if you do, please point me to the solution, thanks!), but refactoring the `refiner` is pretty easy:

```javascript
function delayAction(dispatch, makeAction) {
  setTimeout(() => dispatch(makeAction()),
             1000)
}

export function refiner(dispatch, action) {
  switch(action.type) {
    case A.ASYNC_INCR:
      delayAction(dispatch, A.increment);
      return;

    case A.ASYNC_DECR:
      delayAction(dispatch, A.decrement);
      return;
  }
}
```

### Higher order functions and abstraction
Because the `refiner` is just a function, we can both test and compose it as we do with all the other piece of code, enabling a comfortable and powerful degree of abstraction.  
For example, suppose we have an higher order component `Pair` which render the same component twice.  
Suppose also that all actions from the child component will be wrapped in an action like:
```javascript
{
  type: PAIR_LEFT | PAIR_RIGHT
  payload: childAction
}
```

Creating the `refiner` is as easy as the reducer:
```javascript
export const makePairRefiner = (childRefiner) => (dispatch, action, state) {
  if(action.type !== A.PAIR_LEFT && action.type !== A.PAIR_RIGHT)
    return;

  const childState = action.type === A.PAIR_LEFT ? state.left : state.right;
  childRefiner(dispatch, action.payload, childState);
}
```

Then we can just call `makePairRefiner(counter.refiner)` and voilà!

### Plumbing
Being this a middleware, the drill is the usual one:
```javascript
import { createStore, applyMiddleware } from 'redux';
import { refinerMiddleware } from 'redux-refiner';
import { rootReducer } from './reducers';
import { rootRefiner } from './refiners';

const store = createStore(rootReducer, applyMiddleware(
  refinerMiddleware(rootRefiner)
));
```

### Impure action
The last thing that you need to know is that `refinerMiddleware` listen for any **impure action**, i.e. an action with `meta.impure === true`.  
You can easily create your own impure actions, but we ship with an handy impure action maker:
```javascript
import { impureAction } from 'redux-refiner';

export const ASYNC_INCR = 'ASYNC_INCR';
export const asyncIncrement = (payload) => impureAction(ASYNC_INCR, payload);
```

The result of `asyncIncrement(2)` will be:
```
{
  type: 'ASYNC_INCR',
  payload: 2,
  meta: {
    impure: true
  }
}
```
