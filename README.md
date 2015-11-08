# flux-and-beyond

This article is for you if:
- you heard/uses flux to organise your project
- you heard funcational reactive programming(FRP) around the same time but not crystally clear what it is
- you want to learn or find out what it is

Also, you think
- softwere craftmanship is important
- deliver code effciently(good quality/time) is more important than deliver code fast 
- you embrace change

## The oneliner
In a nutshell, FRP is a level of abstraction of time done in funcational way.
If this is still too long, just remember two words: abstraction and time.

### 1. the right mindset about abstraction
Understanding any form of abstraction takes time, you need to be patient like you first start learning how to program.
Remember:
``` 
Empty your mind, be formless, shapeless - Bruce Lee
Stay hungry stay foolish - Steve Jobs
```

### 2. Analogy
Personally, I find analogy is the best place to start with when it comes to abstractions especially new ones.

The abstraction FRP provides is like the abstraction vector provides when dealing with pixel. Vector 'dettaches' pixel from the real world(screen). Thus, screen size becomes irrelevant or a separated concern until vectors get printed onto a screen. This is why 3d modeling is done is vector and bootstrap 4 is using em/rem.

### 3. Flux, Time and website.
Flux's essence is an uni-direction data flow. Data always flows from 'Action' to 'Render'. This is a very sound concept and it captures exactly how state should be managed at a point of time. 
Now, let's extend the single point of time to a time line and this is what we got:
```
────E──────────E──────────E───────────────>
    |          |          |
    T          T          T
    |          |          |
    ▼          ▼          ▼
────UI─────────UI─────────UI──────────────>
```
The horizonal lines represents time. 
`E` is event such as a mouse click, a key strock ... happens at a particular time.
`UI` on the bottom line is a particular UI state at a particular time.
`T` in the middle is how you manage state, it transforms an `E` to an `UI`. It can be Flux or any other form you like. But essentially, it's just a function or a bundle of functions to transform an input value to an output value that UI can understand.
Notice all the vertical lines are straight from `E` to `T` to `UI`? This means we are dealing with synchronous operations.
In practice, the graph is more like the one below:
```
────E1─────────E2────────────E3──────────────>
    |          |             |
    |          |             |--A2-|
    |          |-------A2---------------|
    |-A1-|                         |    |
    T    T                         T    X
    |    |                         |    
    ▼    ▼                         ▼    
────UI1──UI2───────────────────────UI3───────>
```
The situation for `E1` is very common. i.e. click a button(`E1`) -> display a spinner(`UI1`) and send an ajax request(`A1`) immediately -> when ajax request comes back with response -> display the result(`UI2`)
Let's put this in (pseudo) code.
```
E1.listen(() => {
  render(T({showSpinner: true}))                                  // this is synchronous
  A1(E1).then(data => render(T(data.assign(showSpinner: false)))) // this is async
})                        
```
Take a note that the T is the same but to deal with async operation we need to add some sugar(`A1`, which produce a promise) on top. There is nothing specially. Let's move on to `E2` and `E3`.

The combination of `E2` and `E3` are very common as well. For instance, user types a character in a search bar(`E2`) -> an ajax request is sent(`A2`) -> before the response is back from server -> user types another character(`E3`) -> another ajax request is sent(`A2`) -> the later request arrives first and the response is transformed(`T`) to update UI(`UI3`) -> the earlier request arrives later -> there is no need to process it(`X`)
The code for it will be
```
function A2(event) {
  storeEventAsTheLatestEventSomeWhere()
  return ajax(getUrlForThisEvent(event))
}

function handleReponse(data) {
  isTheResponseComingFromTheLatestRequest() ? render(T(data)) : doNothing())
}

searchBar.listen('keypress', (e) => {
  A2(e.target.value).then(handleReponse)
})
```
The implementation of `storeEventAsTheLatestEventSomeWhere` and `isTheResponseComingFromTheLatestRequest` is omitted but they access or mutate/replace a variable somewhere in order to know if a response coming back from a latest request.

When we look at these two examples, they have a common ground. They deal with asynchronous operations. In other words, they deal with events happen at not just one but multiple different points of time.

If there is a common ground, there is an abstraction. So, how do we abstruct the concept of multiple different points in time into something like `vector` vs `pixel`.

Let's start with changing the code for `E2` and `E3` to see if we can come up with some abstraction then see if it can be applied widely.
```
stream(keyboardEvent) // wrap keyboardEvent into a context called stream

  .map(e => e.target.value) // for every key press all I care is the value of the search box - search term
  
  .flatMapLatest(searchTerm => ajax(url + `?searchTerm=${searchTerm}`) // for every search term generate an ajax
  request and when reponse is back, get the latest data - latest search result
  
  .map(searchResults => T) // format the data into UI markup - [1, 2, 3] => <li>1</li> <li>2</li> <li>3</li>
  
  .do(uiMarkUp => render(uiMarkUp)) // redner UI
```
For `E1`:
```
const clickButtonStream = stream(clickButtonEvent) // wrap click event into a context called stream

const clickButtonWhenSpinnerIsOffStream = clickButtonStream

  .map(() => ({showSpinner: true})) // for every click generate an object indicating spinner needs to be shown
  
  .distinctUntilChanged() // for every object above take those distinct ones that their value changed, so that clicks on a button when spiner is shown is filtered out.
  

clickButtonWhenSpinnerIsOffStream  

  .map(T) // transform the showSpinner flag into UI markup
  
  .do(render) // redner UI
  
  
clickButtonWhenSpinnerIsOffStream

  .flatMap(() => ajax(url())) // for every click on the button generate a ajax request and when response back pass data downwrads
  
  .map(data => data.assign({showSpinner: false})) // since reponse is back, set the flag for showing spinner to false

  .map(T) // turn data into UI markup

  .do(render) // redner UI
```

The common part of the examples above is that all the code dealing with events happening in different point of time has been abstracted away by methods like `map`, `flatMap`, `flatMapLatest` and `distinctUntilChanged`. This makes the difference of separating code dealing with event at a particular time(make a request to server when user enter a character in search box) and the code dealing with related events along time(which request to take to display the latest search results).

In javascript, there are many libs out there to provide funcations like `map`, `flatMap` ...
Examples are Rxjs, Baconjs, Kefir etc...
Now, start googling them and look into these libs if you think this level of abstraction is what we should strive for.

You might wonder what the hell is `flatMap` when reading through the examples. I'll explain it in my next post, if you still like to read it then. So stay tuned for a "`flatMap` and beyond" (https://github.com/shineli1984/flatMap-and-beyond/blob/master/README.md)
