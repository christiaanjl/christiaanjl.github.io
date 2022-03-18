---
layout: post
title:  "Taming global state management"
date:   2022-03-12 16:15:10 +0000
categories: react context redux
---

Much of the complexity in modern React applications is in managing state. [Managing state] has plagued the software industry for at least a few decades, 
so there is no surprise that we're still talking about it today and inventing new ways to manage state in 2022. In this article, I will focus on React [Component State] and walk
through how the JS community has chosen to manage state, what some of the trade-offs are and propose a few remediating actions.

Familiarity with React Hooks, ES6, Context and Redux is needed to make this an interesting read.

&nbsp;
## Vanilla React
Plain-old-basic-vanilla-nothing-fancy React with state and handlers passed as props. This will be our starting point with the other sections based on the code below.

{% highlight jsx %}
const INITIAL_STATE = {
    name: "Ben",
    surname: "Franklin"
}

function SomeComponent () {
    const [state, setState] = useState(INITIAL_STATE);
    return (
        <>
            <Header title={state.name}/>
            <button onClick={() => setState({
                name: 'Sally',
                surname: 'Franklin'
            })}>
                Make Sally
            </button>
        </>
    )
}
{% endhighlight %}

Component `SomeComponent` consists of a `<Header>` and a `<button>`. When the button is clicked we update the `name` passed to the header. That's it.

Non-exhaustive summary of this approach:

**Advantages**
* Easy to understand

**Disadvantages**
* Refactoring overhead when moving state within the component tree
* Intermediary components do not necessarily care about state

&nbsp;
## Context

To remedy some of the issues with the above approach in larger applications, Context was introduced; 
"Context provides a way to share values like these between components without having to explicitly pass a prop through every level of the tree"[[1]]

{% highlight jsx %}
// index.jsx
const INITIAL_STATE = {
    name: "Ben",
    surname: "Franklin"
}
const personState = useState(INITIAL_STATE);

const PersonContext = React.createContext(undefined);

<PersonContext.Provider value={personState}>
    <App />
</PersonContext.Provider>
{% endhighlight %}

{% highlight jsx %}
// SomeComponent.jsx
function SomeComponent() {
    const [person, setPerson] = useContext(PersonContext);
    return (
        <>
            <Header title={person.name}/>
            <button onClick={() => setPerson({
                name: 'Sally',
                surname: 'Jones'
            })}>
                Make Sally
            </button>
        </>
    )
}
{% endhighlight %}

Person's state is provided somewhere higher up the component stack (e.g. index.jsx) and 
subcomponents (e.g. SomeComponent.jsx) are able to access this state and state modifiers through the `useContext` Hook without 
higher level components passing them using props through the entire component stack. 
Pretty standard, nothing new so far.

There are some questions to this approach we might ask.
1. Given multiple states, should we provide a context for each state or merge them into one state? 
How will this impact code quality and performance respectively?
2.  Is state now essentially a global object with all fields public? 
If so, should we constrain access and updates through a defined interface/API/methods?

**What have we gained?**
* Any subcomponent can directly access any state it becomes interested in, provided it's a subcomponent of the context Provider.

**What has it cost us?**
* 'Global' state blob that anyone can change in any way
* Subcomponent rerenders if anything in blob changes.

&nbsp;
## Context+

Enter [`useReducer`]. The opportunity to constrain updates to state.

{% highlight jsx %}
// index.jsx
const INITIAL_STATE = {
    name: "Ben",
    surname: "Franklin"
};

function reducer(state, action) {
    switch (action.type) {
        case 'UPDATE_NAME':
            return {...state, name: action};
        case 'UPDATE_SURNAME':
            return {...state, surname: action};
        default:
            throw new Error();
    }
}

const [state, dispatch] = useReducer(reducer, INITIAL_STATE);

const PersonState = React.createContext(null);
const PersonDispatch = React.createContext(null);

<PersonState.Provider value={state}>
    <PersonDispatch.Provider value={dispatch}>
        <App />
    </PersonDispatch.Provider>
</PersonState.Provider>
{% endhighlight %}

{% highlight jsx %}
// SomeComponent.jsx
function SomeComponent() {
    const person = useContext(PersonState);
    const dispatch = useContext(PersonDispatch);

    return (
        <>
            <button onClick={() => dispatch({
                type: 'UPDATE_NAME',
                payload: 'Sally'
            })}>
                Make Sally
            </button>
            <Header title={person.name}/>
        </>
    )
}
{% endhighlight %}

Person's state is provided through the `PersonState` Provider and updates to that state through the `PersonDispatch` Provider. 
Subcomponents can now choose whether they are interested in the state or updating the state (or both) and are constrained to only the updates 
provided by the reducer (through `dispatch`), namely `UPDATE_NAME` and `UPDATE_SURNAME`. 

**What have we gained?**
* State updates are controlled/specified
* Components interested in `dispatch` only are not re-rendered when state changes 

**What has it cost us?**
* Doubling the number of providers overall. Building the `<App />` wrapper pyramid or needing helper code.
* Many of the Redux costs but few of the benefits. Redux is generally considered verbose, to the extent that 
Redux Toolkit was introduced to standardise the reducers above and the often needed extra helper code like Action Creators, constant/fixed `action.type` values, etc, etc. 

So, are there other ways to constrain updates to state with a bit less ceremony? Yes, we do it every day... methods on objects!

&nbsp;
## Context++
To manage and constrain updates to state, a utility function `createContextStateProvider` was created. 
This function takes an initial state `INITIAL_STATE` and functions to update that state `personActions` as arguments and returns 
a hook to access the state, a hook to access the actions and a provider.


{% highlight jsx %}
// PersonContext.js
const INITIAL_STATE = {
    name: "Ben",
    surname: "Franklin"
};

const personActions = ({setState}) => ({
    setName: (name) => {
        setState(state => ({...state, name}))
    },
    setSurname: (surname) => {
        setState(state => ({...state, surname}))
    },
});

export const {
    Provider,
    useState,
    useActions
} = createContextStateProvider(personActions, INITIAL_STATE);
{% endhighlight %}

{% highlight jsx %}
// index.js
import {Provider} from './PersonContext';

<Provider>
  <App />
</Provider>
{% endhighlight %}

{% highlight jsx %}
// SomeComponent.jsx
import * as PersonContext from 'PersonContext';

function SomeComponent() {
    const person = PersonContext.useState();
    const {setName} = PersonContext.useActions();
    // const [person, {setName}] = PersonContext.useAll();

    return (
        <>
            <button onClick={() => setName('Sally')}>
                Make Sally
            </button>
            <Header title={person.name}/>
        </>
    )
}
{% endhighlight %}

Our subcomponent `SomeComponent.tsx` can now call `PersonContext.useActions()` for the available actions and simply `setName()` to update the state! 

I've written a simple [todo list fetcher application] to show it in action. `createContextStateProvider` takes care of the things 
we have to do but don't really care about; like calling `useState`, `createContext`, splitting state and action providers and lets 
us focus on what we do care about, namely defining our state and valid actions. If you're comfortable with TypeScript, generics and lodash feel free to [take a gander] at the implementation. 
In addition to the standard state setter returned from `[_, setState]  = useState()`, it also provides `mergeState` and `deepMerge` for a shallow and deep merge respectively to avoid painful [nested updates] and 
so `personActions` above can be simplified to:
{% highlight jsx %}
const personActions = ({mergeState}) => ({
    setName: (name) => mergeState({name}),
    setSurname: (surname) => mergeState({surname})
});
{% endhighlight %}


**What have we gained?**
* Control over state updates with minimal ceremony

**What has it cost us?**
* `createContextStateProvider` is not an external dependency and needs to be understood and maintained by you.

I believe we've at least somewhat "Tamed global state management" and could still improve on our `createContextStateProvider` utility. 
Perhaps change `PersonContext.useActions()` to  `usePersonActions()` or consider using [Immer] instead of [lodash] as [Redux Toolkit does]. 
All solutions have their costs and I'll leave their exploration for another time.

Having come this far it seems a good idea to go a little further and show how a Redux and Redux Toolkit approach compares with what we have so far.

&nbsp;
## Redux
The [Redux] code below looks very similar to our [Context+](#context-1) `useReducer` solution.

{% highlight jsx %}
// PersonReducer.js
const INITIAL_STATE = {
    name: "Ben",
    surname: "Franklin"
}

export const PersonReducer = (state = INITIAL_STATE, action) => {
    switch (action.type) {
        case 'UPDATE_NAME':
            return {...state, name: action};
        case 'UPDATE_SURNAME':
            return {...state, surname: action};
        default:
            return state;
    }
}
{% endhighlight %}


{% highlight jsx %}
// index.js
export const store = {
    person: PersonReducer
}

<Provider store={store}>
    <App/>
</Provider>
{% endhighlight %}

{% highlight jsx %}
// SomeComponent.jsx
function SomeComponent({person, dispatch}) {
        return (
        <>
            <button onClick={() => dispatch({
                type: 'UPDATE_NAME',
                payload: 'Sally'
            })}>
                Make Sally
            </button>
            <Header title={person.name}/>
        </>
    )
}

export default connect(
    (store) => ({ person: store.person }))
    (SomeComponent);

{% endhighlight %}

Compared to [Context+](#context-1):

**What have we gained?**
* Observable store (state, actions, ...) and middleware:
    * Redux Debug - visibility as to how, when and why state changed
    * Redux Thunk - function dispatch
    * Redux Saga
        * A way to handle long running transactions with side-effects or potential failures
        * Throttling & Debouncing
        * Retry async calls
        * ...
    * ...
* Components only rerender when state they are `connect`ed to change & other performance improvements.
* Single application Provider at a known location (benefit?)

**What has it cost us?**
* an additional dependency?
* package size?

Redux is still considered quite verbose and many have attempted to create a sensible set of conventions for larger projects. 
My efforts are [here]. Let's see how Redux's own official attempt [Redux Toolkit] compares.

&nbsp;
## Redux Toolkit

{% highlight jsx %}
// PersonReducer.js
const INITIAL_STATE = {
    name: "Ben",
    surname: "Franklin"
}

export const personSlice = createSlice({
    name: 'person',
    INITIAL_STATE,
    reducers: {
        setName: (state, action) => {
            state.name = action.payload;
        },
        setSurname: (state, action) => {
            state.surname = action.payload;
        },
    },
})

export const { setName, setSurname } = personSlice.actions
export default personSlice.reducer
{% endhighlight %}

{% highlight jsx %}
// index.js
export const store = configureStore({
    person: personReducer,
});

<Provider store={store}>
    <App/>
</Provider>

{% endhighlight %}

{% highlight jsx %}
// SomeComponent.jsx
function SomeComponent(props) {
    const person = useSelector((state) => state.person)
    const dispatch = useDispatch()

    return (
        <>
            <button onClick={() => dispatch(setName('Sally'))}>
                Make Sally
            </button>
            <Header title={person.name}/>
        </>
    )
}
{% endhighlight %}

Looks very similar to [Redux](#redux) above! Likely due to the fact that this is a very simple example and much of the Redux helper code 
around Action Creators, etc is not shown.

We could compare Redux Toolkit with [Context+](#context-1) as we did with Redux above but the outcome would be similar. 
Let's compare it to [Context++](#context-2) instead which does away with the `useReducer` Hook in favour of simple object update methods. 

**What have we gained?**
* Observable store (state, actions, ...) & middleware
* Single application Provider at a known location (benefit?)

**What has it cost us?**
* ?

At first glance [Context++](#context-2) and [Redux Toolkit](#redux-toolkit) look quite similar; 
`personActions` and `personSlice.reducer` look very similar (even more so if we use Immer instead of lodash) and
in `SomeComponent.tsx` we use the state and update hooks to get the state and update functions 
and call `setName('Sally')` and `dispatch(setName('Sally'))` respectively. 

So... given they look similar in managing changes to state, do we go for [Redux Toolkit](#redux-toolkit) with its bells and whistles (almost all of which not covered in this article) 
 or [Context++](#context-2) with a comparatively simple utility function we need to maintain ourselves? 

&nbsp;
## Conclusion

Having taken a tour through state management in React, it's clear that there are costs and benefits to all solutions considered. 
For my part, the observable Redux store and it's accompanying browser dev tools seems like a lot to give up. Those who find Redux 
too verbose or arduous to work with, give [Context++](#context-2) another look to manage state in slightly larger projects.


[1]:https://reactjs.org/docs/context.html
[Component state]: https://reactjs.org/docs/state-and-lifecycle.html
[Managing state]: https://github.com/papers-we-love/papers-we-love/blob/master/design/out-of-the-tar-pit.pdf
[`useReducer`]: https://reactjs.org/docs/hooks-reference.html#usereducer
[todo list fetcher application]: https://github.com/christiaanjl/react-context-template
[take a gander]: https://github.com/christiaanjl/react-context-template/tree/main/src/context
[nested updates]: https://alexsidorenko.com/blog/react-update-nested-state/
[Immer]: https://immerjs.github.io/immer/
[Redux Toolkit does]: https://redux-toolkit.js.org/usage/immer-reducers#redux-toolkit-and-immer
[Redux Toolkit]: https://redux-toolkit.js.org/
[Redux]: https://redux.js.org/
[lodash]: https://lodash.com/
[here]: https://github.com/christiaanjl/react-redux-template
