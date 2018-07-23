# @ngx-patterns/store-service

Adds an abstraction layer between Angular components and the [@ngrx](https://github.com/ngrx/platform) store and effects. This decouples the components from the store, selectors, actions and effects and makes it easier to test components.

# Install

Get the latest version from NPM

```sh
npm install @ngx-patterns/store-service
```

# Comparison

## Before

> Component

```ts
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { Book } from 'src/app/shared/books/book.model';
// Tight coupling to ngrx, state model, selectors and actions
import { Store } from '@ngrx/store'; 
import { AppState } from 'src/app/store/appstate.model';
import { getAllBooks } from 'src/app/store/books/books.selectors'; 
import { AddBookAction } from 'src/app/store/books/books.actions'; 
 
@Component({
    selector: 'nss-book-list',
    templateUrl: './book-list.component.html',
    styleUrls: ['./book-list.component.scss']
})
export class BookListComponent {

    books$: Observable<Book[]>;

    constructor(
        private store: Store<AppState>
    ) {
        this.books$ = this.store.select(getAllBooks());
    }

    addBook(book: Book) {
        this.store.dispatch(new AddBookAction(book));
    }
}
```

## After

> Component

```ts
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { Book } from 'src/app/shared/books/book.model';
import { BookStoreService } from 'src/app/shared/books/book-store.service'; // <- Reduced to just one dependency. Loose coupling

@Component({
    selector: 'nss-book-list',
    templateUrl: './book-list.component.html',
    styleUrls: ['./book-list.component.scss']
})
export class BookListComponent {

    books$: Observable<Book[]>;

    constructor(
        private bookStore: BookStoreService // <- StoreService
    ) {
        this.books$ = this.bookStore.getAllBooks(); // <- Selector
    }

    addBook(book: Book) {
        this.bookStore.addBook(book); // <- Action
    }
}
```

> BookStoreService

```ts
import { Injectable } from '@angular/core';
import { Select, StoreService, Dispatch } from '@ngx-patterns/store-service';
import { Observable } from 'rxjs';
import { Book } from 'src/app/shared/books/book.model';
import { getBooks } from 'src/app/store/books/books.selectors';
import { State } from 'src/app/store/store.model';
import { AddBookAction } from 'src/app/store/books/books.actions';

@Injectable()
export class BookStoreService extends StoreService<State> {

    @Select(getBooks) // <- Selector
    getAllBooks: () => Observable<Book[]>;

    @Dispatch(AddBookAction) // <- Action
    addBook: (book: Book) => void;

    @Observe(Actiontypes.BooksLoaded)
    booksLoaded$: Observable<Book[]>; // <- Action stream / effects
}
```

# How to use

## StoreService

The `BookStoreService` Injectable class should extend the `StoreService<State>` class where `State` is your ngrx state model.

```ts
import { StoreService } from '@ngx-patterns/store-service';
import { AppState } from 'app/store/state.model';

@Injectable()
export class BookStoreService extends StoreService<AppState> {
    ...
}
```

## Selectors

To use selectors you have to use the `@Select(...)` decorator inside the `StoreService`. Add the selector function inside the `@Select(...)` annotation:

```ts
// Define the selector function
export function selectAllBooks() {
    return state => state.books;
}

...

// Use the selector function inside the @Select(...) annotation
@Select(selectAllBooks)
allBooks: () => Observable<Book[]>;
```
The selector needs to be a __function__.

Be sure to use correct typing for the property inside the `StoreService`. If a parameter is required inside the selector function it also has to be required in the property typing.

```ts
export function selectBookById(id: number) {
                              ^^^^^^^^^^^^
    return state => state.books[id];
}

...

@Select(selectBookById)
getBook: (id: number) => Observable<Book>;
         ^^^^^^^^^^^^
// The typing of the selector function and the property have to match!
```

## Actions

To dispatch actions a similar approach as mentioned in the selectors is used. Add a property with the `@Dispatch(...)` annotation.

```ts
// Defined the Action as a class
export class LoadBooksAction implements Action {
    public type = '[Books] Load books';
}

...
// Use the Action class inside the @Action(...) annotation
@Dispatch(LoadBooksAction)
loadBooks: () => void;
```

If the Action class expects parameters, the typings on the property inside the `StoreService` have to match the class constructor.

```ts
export class AddBookAction implements Action {
    public type = '[Books] Add book';
    constructor(
        public payload: Book
               ^^^^^^^^^^^^^
    ) {}
}

...
@Dispatch(AddBookAction)
addBook: (book: Book) => void;
         ^^^^^^^^^^^^
// The typing of the action constructor and the property have to match!
```

## Observers (from @ngrx/effects)

Observers are a way to listen for specific action types on the `Actions` stream from [@ngrx/effects](https://github.com/ngrx/platform/blob/master/docs/effects/README.md).

```ts
@Observe(Actiontypes.BooksLoaded)
booksLoaded$: Observable<Book[]>;
```

It automatically maps the action to it's payload. The `@Observe(...)` decorator wraps the following functionality:

```ts
this.actions.pipe(
    ofType(ActionTypes.BooksLoaded),
    map(action => action.payload)
)
```

You can also provide multiple types, just like on the `ofType(...)` pipe.

```ts
@Observe(Actiontypes.BooksLoaded, Actiontypes.BookLoadFailed)
booksLoaded$: Observable<Book[] | string>;
```

## Complete BookStoreService
The finished `BookStoreService` looks like this:
```ts
import { Dispatch, Select, StoreService } from '@ngx-patterns/store-service';
import { AppState } from 'app/store/state.model';
import { selectAllBooks, selectBookById } from 'app/store/books/books.selectors';
import { LoadBooksAction, AddBookAction } from 'app/store/books/books.actions';

@Injectable()
export class BookStoreService extends StoreService<AppState> {

    @Select(selectAllBooks)
    allBooks: () => Observable<Book[]>;

    @Select(selectBookById)
    getBook: (id: number) => Observable<Book>;

    @Dispatch(LoadBooksAction)
    loadBooks: () => void;

    @Dispatch(AddBookAction)
    addBook: (book: Book) => void;

    @Observe(Actiontypes.BooksLoaded, Actiontypes.BookLoadFailed)
    booksLoaded$: Observable<Book[] | string>;
}
```  

# Prerequisites

## Selectors should be functions

```ts
// This will not work
const selector = state => state.property;
```

```ts
// This works
function selector() {
    return state => state.property;
}
```

If the selector is not a function, the typing of the StoreService Class won't work: `() => Observable<any>`

## Actions should be classes

```ts
export class LoadAction implements Action {
    public type: 'Load action';
    constructor(
        public payload: any
    ) { }
}
```
This is mandatory because the actions are instantiated using the `new` keyword.

## Actions should have a `payload` property

To work with the `@Observe(...)` decorator all actions should have a `payload` property. The `@Observe(...)` decorator automatically maps to this property so you don't have to extract it yourself. Other property names will not work.


# Testing
Testing your components and the StoreService is made easy. The `@ngx-patterns/store-service/testing` package provides helpful test-helpers to reduce testing friction.

## Selectors

To test selectors you provide the `StoreService` using the `provideStoreServiceMock` method in the testing module of your component. Then cast the store service instance using the `StoreServiceMock<T>` class to get the correct typings.


```ts
import { provideStoreServiceMock, StoreServiceMock } from '@ngx-patterns/store-service/testing';
...
let bookStoreService: StoreServiceMock<BookStoreService>;
...
TestBed.configureTestingModule({
    imports: [AppModule],
    providers: [
        provideStoreServiceMock(BookStoreService)
    ]
})
...
bookStoreService = TestBed.get(BookStoreService);
```

The `StoreServiceMock` class replaces all selector functions on the store service class with a `BehaviourSubject`. So now you can do the following to emit new values to the observables:

```ts
bookStoreService.getAllBooks().next(newBooks);
```

The `BehaviourSubject` is initialized with the value being `undefined`. If you want a custom initial value, the `provideStoreServiceMock` method offers an optional parameter. This is an object of key value pairs where the key is the name of the selector function, e.g. `getAllBooks`.

```ts
import { provideStoreServiceMock, StoreServiceMock } from '@ngx-patterns/store-service/testing';
...
let bookStoreService: StoreServiceMock<BookStoreService>;
...
TestBed.configureTestingModule({
    imports: [AppModule],
    providers: [
        provideStoreServiceMock(BookStoreService, {
            getAllBooks: []
        })
    ]
})
...
bookStoreService = TestBed.get(BookStoreService);
```

The `BehaviourSubject` for `getAllBooks` is now initialized with an empty array instead of `undefined`.

## Actions

To test if a component dispatches actions, you import the `NgrxStoreServiceTestingModule` inside the testing module.

To get the injected Store instance use the `MockStore` class for proper typings.

```ts
import { NgrxStoreServiceTestingModule, MockStore } from '@ngx-patterns/store-service/testing';
...
let mockStore: MockStore;
...
TestBed.configureTestingModule({
    imports: [
        NgrxStoreServiceTestingModule
    ]
})
...
mockStore = TestBed.get(Store);
```

Optionally use the `withState(...)` function on the `NgrxStoreServiceTestingModule` to provide an object that should be used as the state.

```ts
import { NgrxStoreServiceTestingModule} from '@ngx-patterns/store-service/testing';
...
const state = {
    books: []
}
...
TestBed.configureTestingModule({
    imports: [
        NgrxStoreServiceTestingModule.withState(state)
    ]
})
```

The `MockStore` class has a `dispatchedActions` property which is an array of all dispatched actions. The last dispatched action is appended at the end.

```ts
const lastDispatchedAction = mockStore.dispatchedActions[mockStore.dispatchedActions.length - 1];

// Or with lodash

const lastDispatchedAction = last(mockStore.dispatchedActions);
```

## Observers / Actions stream

To test the actions stream, you import the `NgrxStoreServiceTestingModule` inside the testing module.

Get the `MockActions` instance from the `TestBed`

```ts
import { NgrxStoreServiceTestingModule, MockActions } from '@ngx-patterns/store-service/testing';
...
let mockActions: MockActions;
...
TestBed.configureTestingModule({
    imports: [
        NgrxStoreServiceTestingModule
    ]
})
...
mockActions = TestBed.get(MockActions);
```

The `MockActions` class provides a `next(...)` function which will emit a new value to the `Actions` stream from @ngrx/effects.
This way you can emit new actions to the stream.

Here is an example on how to test this using the `MockActions` class.

```ts
import { NgrxStoreServiceTestingModule, MockActions } from '@ngx-patterns/store-service/testing';
...
let mockActions: MockActions;
...
TestBed.configureTestingModule({
    imports: [
        NgrxStoreServiceTestingModule
    ]
})
...
mockActions = TestBed.get(MockActions);
...
it('test', () => {
    const expectedValue = [ { author: 'Author', title: 'Title', year: 2018 } ];
    storeService.booksLoaded$.subscribe(
        books => {
            expect(books).toBe(expectedValue);
        }
    );
    const action = new BooksLoadedAction(expectedValue);
    mockActions.next(action)
})
```

# Examples

For detailed examples of all this have a look at the Angular Project in [the src/app folder](src/app).

## Store Service

Have a look at the [BookStoreService](src/app/shared/books/book-store.service.ts)

## Testing

For examples on Component Tests please have look at the test for the [BookListComponent](src/app/components/book-list/book-list.component.spec.ts) and the [NewBookComponent](src/app/components/new-book/new-book.component.spec.ts)

Testing the `StoreService` is also very easy. For an example have a look at the [BookStoreService](src/app/shared/books/book-store.service.spec.ts)
