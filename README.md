# RXJS Tricks

## Summary

* 5 helpful RxJS solutions


## 5 helpful RxJS solutions to everyday problems

> **From** : [Medium](https://medium.com/grensesnittet/5-helpful-rxjs-solutions-d34f7c2f1cd9)
> 
> **Author**: [John Melin](https://jme-76140.medium.com/)
>
> **Rewited**: Renaud Racinet

RxJS is a javascript library that allows us to code reactively using observable streams. With observables and functions provided by RxJS we can compose readable asynchronous code.

If you are using Angular you are already using it, since the framework is built upon it. Still many developers don’t use it much more than just calling subscribe on http requests.

### Start using RxJS with pipe-able operators!

A lot of the power behind RxJS comes from its pipe-able operators. What are those you ask? These are basically functions that can be “stacked” in order of execution in a pipe to compose new streams.” This will create better solutions with more readable and often less code.

There are numerous articles on best practises and explanations on what each of these operators do.

> You should definitely give a [few](https://blog.strongbrew.io/rxjs-best-practices-in-angular/#cleancode-practices) of these a read!

But while many of them may be great, I find the examples they contain often do not reflect the problems we find in everyday coding. So I decided to take another approach by giving you 5 RxJS solutions to everyday problems.

### 1. Simplifying handling of multiple HTTP requests

Let’s kick off the list with one of the most common scenarios in our daily life as web programmers, namely: handling multiple http requests. Many might already know about this, but as this is something I see all the time, I decided to include it.

#### Scenario 1: Sequential requests. Using data from previous requests in next requests.

We are often required to make requests based on the response from earlier requests. This is true in the scenario below.

 * We first retrieve a post based on an id.
 * Using the post response from the first call, we then want to retrieve the user of the post.
 * After we have the user we want to get all the posts of the retrieved user.

```Javascript
 this.postService.loadPost(postId).subscribe(post => {
   this.userService.loadUser(post.userId).subscribe(user => {
     this.postService.loadPosts(user.id).subscribe(posts => {
 ...
```

As we can see, this results in a nested mess. Imagine some other scenario where we might want to do some operations on the responses before passing it on to the next request. This could make the nesting even worse.

#### Solution: switchMap

This can be easily simplified with the operator [switchMap](https://www.learnrxjs.io/learn-rxjs/operators/transformation/switchmap). The functionality of switchMap lies in its name. After the first observable emits it then subscribes to an inner observable.

This is perfect for our scenario. We want to use our first observable of posts and map the result to a new observable that loads users.

The operator will take the outer observable of a post and when it emits, switch to the inner observable of ‘user’ and map the result of the outer observable to our inner observable.

```Javascript
   this.postService.loadPost(postId).pipe(
     // map our observable of a post to a new observable of user
     switchMap(post => this.userService.loadUser(post.userId)),

     // map our previous observable of a user to a new observable of all the users posts
     switchMap(user => this.userService.loadPosts(user.id)),
   )
   .subscribe(posts => console.log(posts));
```

> **Note:** Other features of switchMap worth noting, is the ability to cancel the previous emitted value of an observable if a new one is emitted before the previous has completed. This can be very helpful when creating autocomplete features.

### Scenario 2: Making parallel requests.

Sometimes we don’t care about the order in which multiple requests finish. We may just want to collect them once they are all done. Say we only have a userId. We want to get all the posts the user has made and metadata about the same user.

As each of these two requests are independent of each other the order in which they finish is not important. We do however want to wait until we have the data from both. Often when loading data we also want to show some sort of loading-indicator.

Below is a naive solution where we patch an object to indicate that we have the data.

```Javascript
let userWithPosts = {
  user: null,
  posts: null
};

this.postService.loadPosts(userId).subscribe(posts => {
  userWithPosts.posts = posts;
});

this.userService.loadUser(userId).subscribe(user => {
  userWithPosts.user = user;
});
```

```html
<p *ngIf=”!(userWithPosts.user || userWithPosts.posts)”>Loading...</p>
<div *ngIf=”userWithPosts.user && userWithPosts.posts”>... do stuff with the data</div>
```

You can agree that this doesn’t seem like a good solution. Imagine if we needed to make even more parallel requests. The code would quickly become long and unreadable.

#### Solution: forkJoin

With RxJs this can be solved in just a few lines of code with the operator [forkJoin](https://www.learnrxjs.io/learn-rxjs/operators/combination/forkjoin). For those that are used to promises, this operator works about the same as a Promise.all([..]). Inside the operator, all requests execute in parallel and will emit when all requests are resolved.

```Javascript
let userWithPosts;

forkJoin({
  posts: this.postService.loadPosts(userId)
  user: this.userService.loadUser(userId)
}).subscribe(response => {
  this.userWithPosts = response;
});
```

```Html
 <p *ngIf=”!userWithPosts”>Loading...</p>
 <div *ngIf=”userWithPosts”>... do stuff with the data</div>
```

> **Note:** Be aware that if any of the requests cast an error you will lose the value of the other requests even if they completed. If you care about them you will have to [catch](https://www.learnrxjs.io/learn-rxjs/operators/error_handling/catch) them.

#### 2. Combining multiple streams into one for a search feature

In one of my projects we implemented a smart search. Our search requests were composed of multiple sources such as

* A searchTerm$ coming from an input field
* A set of metadataFilters$ (ranges, checkboxes)
* A set of polygons$ that could be drawn on a map
* The currentResultPage$ coming from a result page with pagination buttons

The source data came from different components and so we needed an easy readable way of combining them in one handy stream.

#### Solution: combineLatest

The [combineLatest](https://www.learnrxjs.io/learn-rxjs/operators/combination/combinelatest) function combines an array of streams into one and emits them all whenever one of them changes.

```Javascript
combineLatest([
  this.searchTerm$,
  this.metadataFilters$,
  this.polygons$,
  this.currentResultPage$
]).subscribe(requestBody => this.search(requestBody))
```

Now we have all the sources we need to make a search request. Everytime any of these sources change we will make a new search. This means that everytime we click the next button or we type something in the input field we make a new request. But is this something we want? That would mean a lot of requests to the backend.

We can improve our solution further with a new operator [debounceTime](https://www.learnrxjs.io/learn-rxjs/operators/filtering/debouncetime) and our friend from our first scenario [switchMap](https://www.learnrxjs.io/learn-rxjs/operators/transformation/switchmap). Remember from before that another feature of switchMap is that it will cancel the previous request if a new one is made before the current one has completed. This means we always get the result from the request that was made last.

#### Improving our solution: Canceling quick requests

The new operator debounceTime also has cancelling capabilities, but this one cancels any new values that are emitted made within the provided dueTime. Say we set the dueTime to 1000ms. This means that if we click the next button in the pagination and clicks it again before 1 second has passed, no new request will be made.

```Javascript
combineLatest([
  this.searchTerm$,
  this.metadataFilters$,
  this.polygons$,
  this.currentResultPage$
]).pipe(
  debounceTime(<dueTime>),
  switchMap(requestBody =>  this.search(requestBody))
)
.subscribe(result => this.result = result);
```

When we use these two operators in tandem we can cancel any new requests that are made within the provided dueTime with the help of *debounceTime*. Our familiar *switchMap* will then cancel the previous request if it has not completed yet giving us the last result.

#### Improving our solution: Cancelling equal requests

We have now reduced unnecessary requests by only making requests when interaction is stable. But this means a new problem occurs. Say we check a checkbox in our filters and then immediately check it off again. This will trigger a request with the same response we just had.

Enter the [distinctUntilChanged](https://www.learnrxjs.io/learn-rxjs/operators/filtering/distinctuntilchanged) operator! This will stop any emits where the current value is the same as the previous. Awesome!

```Javascript
combineLatest([
  this.searchTerm$,
  this.metadataFilters$,
  this.polygons$,
  this.currentResultPage$
]).pipe(
  debounceTime(<dueTime>),
  distinctUntilChanged(),
  switchMap(requestBody =>  this.search(requestBody))
)
.subscribe(result => this.result = result)
```

Or is it? We will see that distinctUntilChanged does not work in our example. It will still make a new request event though our requests are the same. Why is that!?

This is because *distinctUntilChanged* uses ‘===’ comparison by default. This means that when using objects and arrays they are considered the same if they have the same reference. For it to work, we need to provide a comparison function that compares something unique, like an id.

#### More problems?! My source observables are no emitting

It’s important to note the *combineLatest* function will not emit until each of its source observables have emitted at least once.

Sometimes you may find that *combineLatest* is not emitting. This may be if any of the sources are of type [Subject](https://www.learnrxjs.io/learn-rxjs/subjects) or [ReplaySubject](https://www.learnrxjs.io/learn-rxjs/subjects/replaysubject). These observables do not require an initial value, and combineLatest will therefore not emit anything until you have called next on them at least once.

Often it’s preferable to use [BehaviourSubjects](https://www.learnrxjs.io/learn-rxjs/subjects/behaviorsubject) when creating observables, but if they do not fit your use case the problem can be mitigated using the [startWith](https://www.learnrxjs.io/learn-rxjs/operators/combination/startwith) operator.

Say our searchTerm$ observable is a ReplaySubject. It has not been provided a start value and *combineLatest* will not emit a value until you have typed something. You can then easily pipe on *startWith* and provide its start value an empty string.

```Javascript
combineLatest([
  // now searchTerm$ will always start with the value empty string
  this.searchTerm$.pipe(startWith("")),
  this.metadataFilters$,
  this.polygons$,
  this.currentResultPage$
]).pipe(
  debounceTime(<dueTime>),
  distinctUntilChanged(),
  switchMap(requestBody =>  this.search(requestBody))
)
.subscribe(result => this.result = result)
```

#### 3. Clicking a button to do a http request with data from a stream as parameter

Say you have a button that when clicked should download an excel sheet containing all posts for a user. The request requires a userId from the current selected user. But the user object is stored as an observable.

A naive solution might look something like this.

```Typescript
userId: string;
 
ngOnInit() {
  this.user$.subscribe(user => {
    this.userId = user.id;
  })
}

// Function called on click from HTML
downloadSpreadSheet() {
  this.http.getExcelDocument(this.userId).subscribe(result => {
    // Do something with the result…
  });
}
```

Now what is wrong with that? Well the biggest problem here is that we are not getting the userId at the same time as the click function is called. This means that on an off chance race conditions could occur. Also the code involved for downloading is spread out and not contained to one function.

#### Solution: withLatestFrom, fromEvent and switchMap

We start off by elevating the click event into a stream with [fromEvent](https://www.learnrxjs.io/learn-rxjs/operators/creation/fromevent) function. You may remember *combineLatest* from the last example. Could we do something like this?

```Typescript
@ViewChild(<some button id>) downloadButton;

combineLatest([
  fromEvent(this.downloadButton, 'click'),
  this.user$,
]) // call download
```

We could, but remember that combineLatest will emit any time its observables emits, so this would download every time the download button is pressed, but also every time the user changes.

[withLatestFrom](https://www.learnrxjs.io/learn-rxjs/operators/combination/withlatestfrom) will only add the last value from an observable at the time your source observable emits.

> *Note:* Be aware that the observable(s) passed to withLatestFrom need to have emitted once before the source observable will emit.

With this we can add the value of the user$ stream but only emit when the click event is triggered. Then we switch our stream to the download call using our familiar switchMap.

```Typescript
@ViewChild(<some button id>) downloadButton;

// Create a stream from button click
fromEvent(this.downloadButton, 'click').pipe(

  // Get the latest value of user at the moment of click
  withLatestFrom(this.user$), 

  // Switch the stream to the download request
  switchMap(([_, user]) => this.http.getExcelDocument(user.id)
)
.subscribe(results => {
  // Do something with the result…
});
```

#### 4. Using the filter operator instead of if/else

In our daily life as programmers when dealing with conditionals we often reach for if/else statements. The need for handling conditionals is no different when coding with streams. But there is a more *reactive* way to tackle conditionals.

#### Scenario: Making different requests based on type

Say we have two different types of user. A reader and writer. Readers can only read posts but not write them. Writers are of course allowed to both read and write posts. Now imagine when clicking on a user in a UI we want to load different things for readers and writers. For readers we might want to load all the articles the user has read, and for writers all the articles the user has created. A solution might look something like this.

```Javascript
this.user$.subscribe(user => {
  if (user.type === 'WRITER') {
    this.http.getUserWrittenArticles(user.id).subscribe(articles => {
      // Do stuff..
    })
  } else if (user.type === 'READER') {
    this.http.getUserReadArticles(user.id).subscribe(articles => {
      // Do stuff..
    })
  }
});
```

As we can see this quickly becomes unreadable. Then image adding even more conditions.

#### Solution: filter

The [filter](https://www.learnrxjs.io/learn-rxjs/operators/filtering/filter) operator will only allow values that pass the provided condition pass.

> **Note:** The RxJs filter is not to be confused with the filter function used with arrays. This one filters streams. If the emitted value is an array that you want to filter on, use the map operator to return a filtered array: map(array => array.filter(…))

With the filter operator we can split up our stream into one of each type. This scales a lot better than performing conditionals inside the stream.

```Javascript
const readerUser$ = this.user$.pipe(filter(user => user.type === 'READER'));
const writerUser$ = this.user$.pipe(filter(user => user.type === 'WRITER'));

readerUser$.pipe(
  switchMap(user => this.http.getUserReadArticles(user.id))
).subscribe(articles => {
    //Do stuff...
});
 
writerUser$.pipe(
  switchMap(user => this.http.getUserWrittenArticles(user.id))
).subscribe(articles => {
    //Do stuff...
});
```

#### 5. Making two different observables share the same subscribe.

Sometimes you may have two observables that emit the same type of value but call the same function. It could go something like this.

```Javascript
fromEvent(this.buttonA, 'click').pipe(
  map(() => 'A'),
  switchMap(value => this.callServer(value))
)
.subscribe(result => { 
    // do stuff.. 
}); 

fromEvent(this.buttonB, 'click').pipe(
  map(() => 'B'),
  switchMap(value => this.callServer(value))
)
.subscribe(result => { 
    // do stuff.. 
}); 
```

A good practice in programming is to keep it “DRY” (don’t repeat yourself). So subscribing to two equal streams feels highly unnecessary.

#### Solution: merge

To avoid to subscribe to each separate observable and call the same function in their subscribe block, this can easily be simplified by using the [merge](https://www.learnrxjs.io/learn-rxjs/operators/combination/merge) function. This will turn multiple observables into one observable.

```Javascript
const buttonAClick$ = fromEvent(this.buttonA, 'click').pipe(map(() => 'A'));
const buttonBClick$ = fromEvent(this.buttonB, 'click').pipe(map(() => 'B'));

merge(buttonAClick$, buttonBClick$).pipe(
  switchMap(value => this.callServer(value))
)
.subscribe(result => { 
    // do stuff..
}); 
```

### Wrapping up

That’s all peeps! Today we went through 5 RxJS recipes that improve on solutions to common problems. I hope you found any of them useful.

Do you know any other useful solutions? Please let me know in the comments below.

Now go out and make better cleaner solutions with RxJS!
