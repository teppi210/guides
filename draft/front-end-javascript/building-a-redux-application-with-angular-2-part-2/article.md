# What we did last time

# Restructuring
  In this part, the application is going to be accomodated in a way that it can support having multiple states in its Store. To do so, the architecture of the application needs to be altered.
  
  In its current state, the application does not have a Meta-Reducer. Meta-Reducers are a map of all the reducer functions in the state tree. It contains a reference for each of the state slices and their reducers. When an action gets dispatched, the Meta-Reducer goes through the map of all reducers, looking for the one that matches the action type and calls the function.
  
 
  Another issue that causes concern is  the actions and the reducer for `operations` are staying together, but as the applicaiton grows and the actions and the reducer are becoming more complex, there's going to be a lot of code in a single file. Thus, the actions and the reducers have to be divided in different files.
  
 
  Having all these concerns in mind, here is how the directory structure will look in the new application:
  
  
  ```
+-- app
    |
    +-- reducers
    |   |
    |   +--index.ts
    +-- actions
    |
    +-- models
```

  Let's get started. First, move `operation.model.ts` to the `models` folder. This is where the models for all entities of the application will be stored from now on.
  
### "Classifying" acitons


 Next, the actions need to be overhauled so that it is known to which reducer function they belong to.
 
 In `operations.ts`, delete the action constants and create a new file in `actions/operations.ts'`
  
 Instead of making a new constant for each action, the actions will be grouped into enums.
 ```
 export const ActionTypes = {
  ADD_OPERATION: 'Add an operation',
  REMOVE_OPERATION:'Remove an operation',
  INCREMENT_OPERATION:'Increment an operation',
  DECREMENT_OPERATION: 'Decrement an operation',
};
```


Because actions will not be dispatched to a reducer directly, but instead this will be done through a Meta Reducer, @ngrx/store has provdied  the `Action` class, which lets us define custom actions and create actions as new class instances when dispatching. Expressing actions as classes also enables type checking in reducer functions.

Here's how an ``ADD_OPERATION`` action would look like:

```
export class AddOperationAction implements Action {
  type = ActionTypes.ADD_OPERATION;

  constructor(public payload:Operation) { }
}

```

The action `type` is given by the `ActionTypes` enum that was previously defined and the payload of the action in the constructor fo the class.


Lastly, a type alias for all actions in order to be later used in teh reducers.

```
export type Actions
  = AddOperationAction |
  RemoveOperationAction |
  IncrementOperationAction |
  DecrementOperationAction
  
```

 
Here is how the full code for `operations` actions:


 ```
 // app/actions/operations.ts
 
import { Action } from '@ngrx/store';
import {Operation} from "../models/operation.model";


export const ActionTypes = {
  ADD_OPERATION: 'Add an operation',
  REMOVE_OPERATION:'Remove an operation',
  INCREMENT_OPERATION:'Increment an operation',
  DECREMENT_OPERATION: 'Decrement an operation',
};




export class AddOperationAction implements Action {
  type = ActionTypes.ADD_OPERATION;

  constructor(public payload:Operation) { }
}

export class RemoveOperationAction implements Action {
  type = ActionTypes.REMOVE_OPERATION;

  constructor(public payload:Operation) { }
}

export class IncrementOperationAction implements Action {
  type = ActionTypes.INCREMENT_OPERATION;

  constructor(public payload:Operation) { }
}

export class DecrementOperationAction implements Action {
  type = ActionTypes.DECREMENT_OPERATION;

  constructor(public payload:Operation) { }
}


export type Actions
  = AddOperationAction |
  RemoveOperationAction |
  IncrementOperationAction |
  DecrementOperationAction
```
#### The new way of dispatching actions
 With the new implementations of actions, comes a new way of dispatching them. 
 
 **Before**
 ```
 //app.component.ts
 
     this._store.dispatch({type: ADD_OPERATION , payload: {
      id: ++ this.id,//simulating ID increments
      reason: operation.reason,
      amount: operation.amount
    }});
```
 
 Instead of directly sending an action with its type and payload, now this will be handled by the operation's class. Now that the action is a class, the dispatching is done by creating a new class member and putting the payload in the class constructor:
 
**After**
```
//Import the action classes for operations
import * as operations from "./common/actions/operations"


this._store.dispatch(new operations.AddOperationAction({
    id: ++ this.id,//simulating ID increments
  reason: operation.reason,
  amount: operation.amount
})
```

### Adapting the reducer function
 There are three problems with the current reducer: 
 
**1.**The type of the state (array) is too simplistic. Instead, it will be replaced with an object which contains the array and add it as an interface. This allows for the state to be more extendable in the future, when new features are implemented and there  is more data that has to be tracked.
 
```
export interface State {
  entities:Array<Operation>
};

```
 
**2.**The `operationsReducer` is not a function, but a constant of type `ActionReducer` to which has a reducer function as a value. This was done because the constant was then directly passed to `provideStore`, without being referenced in a Meta reducer.
Now that a Meta Reducer going to be implemented,  `operationsReducer` will be renamed simply to `reducer` and converted to a function with a return type of `State`.
 
**Before**

```
export const operationsReducer: ActionReducer = (state = initialState, action: operations.Actions) => { ... }
```
**After**

```
export function reducer(state = initialState, action: operations.Actions): State  {...} 
```

**3.**The new action types are not implemented. To implement them, we'll import `actions/operations` and use `ActionTypes` from it in the case statements.
 
**Before**
```
  switch (action.type) {
    case ADD_OPERATION:
      const operation:Operation = action.payload;
      return [ ...state, operation ];
  }
```
**After**
```
  switch (action.type) {
    case operations.ActionTypes.ADD_OPERATION: {
      const operation: Operation = action.payload;
      return {
        entities: [...state.entities, operation]
      };

    }
    //... rest of the cases
```   
 
 
 Here is how the new `operations` reducer looks like after all the changes have been applied:
 
 ```
import '@ngrx/core/add/operator/select';
import 'rxjs/add/operator/map';
import 'rxjs/add/operator/let';
import { Observable } from 'rxjs/Observable';
import * as operations from '../actions/operations';
import {Operation} from "../models/operation.model";


/*
  From a simple array ( [] ),
  the state becomes a object where the array is contained
  withing the entities property
 */
export interface State {
  entities:Array<Operation>
};

const initialState: State = {  entities: []};

/* 
 Instead of using a constant of type ActionReducer, the
 function is directly exported 
 */
export function reducer(state = initialState, action: operations.Actions): State {
  switch (action.type) {
    case operations.ActionTypes.ADD_OPERATION: {
      const operation: Operation = action.payload;
      /*
      Because the state is now an object instead of an array,
      the return statements of the reducer have to be adapted.
      */
      return {
        entities: [...state.entities, operation]
      };

    }

    case operations.ActionTypes.INCREMENT_OPERATION: {
      const operation = ++action.payload.amount;
      return Object.assign({}, state, {
        entities: state.entities.map(item => item.id === action.payload.id ? Object.assign({}, item, operation) : item)
      });
    }

    case operations.ActionTypes.DECREMENT_OPERATION: {
      const operation = --action.payload.amount;
      return Object.assign({}, state, {
         entities: state.entities.map(item => item.id === action.payload.id ? Object.assign({}, item, operation) : item)
      });
    }

    case operations.ActionTypes.REMOVE_OPERATION: {

      return Object.assign({}, state, {
        entities: state.entities.filter(operation => operation.id !== action.payload.id)
      })

    }
    default:
      return state;
  }

};

 ```
 
From the code we have we can conclude that a module of the state has to export:
 1. A reducer function 
 2. An interface that describes how the state looks like.
 


### Creating a Meta Reducer

With the actions and the reducer adapted to the new standards, it is time to implement the Meta reducer. If each of the reducer modules represent table in a database, the meta reducer `State` interface represents the database schema and the meta `reducer` function itself represents the database itself

. To have a clearer idea what happens behind the scenes in a Meta Reducer, we need to first look into the implementation of `combineReducers`.

 
#### combineReducers 
```
const combineReducers = reducers => (state = {}, action) => {
  return Object.keys(reducers).reduce((nextState, key) => {
    nextState[key] = reducers[key](state[key], action);
    return nextState;
  }, {});
};
```
`combineReducers` is a function that takes an object with all the reducer functions as property values and extracts its keys. Then it uses [Array.reduce()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce) to accumulate the return value of each of the reducer functions into a state tree and heassign it to the key the reducer corresponds to. In the end, the state tree (Store) is returned - an object which contains the key-value pairs of the reducers and the states they returned.
 
#### Implementation

 In your `app/reducers` folder, create a new file named `index.ts`. In it, paste the following snippet:
 
```
import {combineReducers, ActionReducer} from '@ngrx/store';
import {Observable} from "rxjs";
import {compose} from "@ngrx/core";


/*
 Import each module of your state. This way, you can access
 its reducer function and state interface as a property.
*/
import * as fromOperations from '../reducers/operations';



/*
 The top-level interface of the state is simply a map of all the inner states.
 */
export interface State {
  operations: fromOperations.State;
}

/* The reducers variable represents the map of all the reducer function that is used in the Meta Reducer */
const reducers = {
  operations: fromOperations.reducer,
};


/* Using combineReducers to create the Meta Reducer and export it from the module. The exported Meta Reducer will be used as an argument in provideStore() in the application's root module. 
*/

const combinedReducer: ActionReducer<State> = combineReducers(reducers);

export function reducer(state: any, action: any) {
    return combinedReducer(state, action);
}
```
The final step of the implementation is to put the  Meta `reducer` as an argument in `provideStore()`
```
import {reducer} from "./common/reducers/index";


@NgModule({
  bootstrap: [ AppComponent ],
  declarations: [
   //...
  ],
  imports: [ 
    /*
     Put the reducer as an argument in provideStore 
    */
    StoreModule.provideStore(reducer),

  ],
})
export class AppModule {
  constructor() {}
}
```

# Getting state slices
 The new architecture of the application requires a different way for accessing the state slices. 
 
 Previously, we could simply access `operations` using the following snippet:
 
 ```
 //app.component.ts 
  import {State, Store} from "@ngrx/store";
  
  export class AppComponent {

  //....

  constructor(private _store: Store<State>) {
    this.operations = _store.select('operations')
  }

 ```
 
**However, this isn't a good idea now that there's a state tree and the `operations` state is an object that could have multiple properties.** Additionally, this doesn't allow combining states in a particlar manner. There needs to be a set of designated functions whose role is to return certain parts of the state tree.

#### Accessing parts of the state tree.
Accessing the state slices is not going to be done directly from the application's components. Instead, the `select`-ing of the state will be done in the `app/reducers` modules. 

To To illustrate how it works,let's first select `operations` from the state tree:


```
 //app/reducers/index.ts
 
 export function getOperations(state$: Observable<State>) {
   return state$.select(state => state.operations);
}

```
  
  
#### Accessing the state properties

Using the last function, only the `operations` state of the application is accessed, and it contains this:
```js
 {
    entities: [ //...an array of operations
 }
```

 That's a problem, because we need a particlar property of the `operations` state - the `entities` array. Here is how it's done:
 
 
 **First**, in `oprations.ts`, export a function which accesses the `entities` property of the `operations` state:
 
```
  
// app/reducers/operations.ts

/*

 Get the entities of the operations state object. This function will be
 imported into the file for the Meta Reducer, where it will
 be composed together with a function that gets the state of 
 the  operations state object out of the application state.
*/

export function getEntities(state$: Observable<State>) {
return state$.select(s => s.entities);
}

```

**Second**, compose `getOperations` and `getEntities`. *Function composition* is one of the building blocks of functional programming. It executes a set of functions, putting the returned value of the first function an an argument for the second function. *Remember that compose applies the result from right to left*.

```
// app/reducers/index.ts

export const getEntities = compose(fromOperations.getEntities, getOperations);
```

In this case, `getOperations` first accesses the `operations` state from the state tree and then `getEntities` gets the `entities` property of  the operations state.

Let's apply this functionality in `app.component.ts`:

```js
//app.component.ts

/*
 In order to access the application state, reference the reducers folder again,
 accessing all the exported members from it though index.ts
 */
import * as fromRoot from './common/reducers';

export class AppComponent {

  //...
  
  constructor(private _store: Store<fromRoot.State>) {
    this.operations = _store.let(fromRoot.getEntities)
  }

```

`_store.let` executes `getEntities` and returns its value.



Here is how the new `AppComponent` looks like with the new state selection and action dispatching implemented:

```js
import { Component } from '@angular/core';
import {State, Store} from "@ngrx/store";
import {Operation} from "./common/models/operation.model";
import * as operations from "./common/actions/operations"
import * as fromRoot from './common/reducers';

@Component({
  selector: 'app-root',
  template: `
      <div class="container">
           
            <new-operation (addOperation)="addOperation($event)"></new-operation>
            <operations-list [operations]="operations| async"  
            (deleteOperation)="deleteOperation($event)"
            (incrementOperation)="incrementOperation($event)"
            (decrementOperation)="decrementOperation($event)"></operations-list>
      </div>
`
})
export class AppComponent {

  public id:number = 0 ; //simulating IDs
  public operations:Observable<Operation[]>;


  constructor(private _store: Store<fromRoot.State>) {
    this.operations = this._store.let(fromRoot.getEntities)

  }

  addOperation(operation) {
    this._store.dispatch(new operations.AddOperationAction({
        id: ++ this.id,//simulating ID increments
      reason: operation.reason,
      amount: operation.amount
    })
    );
  }

  incrementOperation(operation){
    this._store.dispatch(new operations.IncrementOperationAction(operation))
  }

  decrementOperation(operation) {
    this._store.dispatch(new operations.DecrementOperationAction(operation))
  }


  deleteOperation(operation) {
    this._store.dispatch(new operations.RemoveOperationAction(operation))
  }

}

```
# Adding a new state
 It's time to take the example application to the next level by adding multi-currency support. Being able to see the operations in different currencies comes with certain requirements:
 
  1. **Reducer** which stores the available currencies and the currently selected currency.
  2. **JSON API** which provides up-to-date currency rates.
  3. **Technique** for reactively changing amounts in the operations.

Let's tackle task #1 and add the `currency` actions and reducer. With the architecture already laid, adding new functionality is much simpler.


### Actions
Let's start with the actions first. Create a new file in `actions` named `currencies.ts`.
The first action will be when the user attempts to change the currency. Here's how the code for the action looks:

```
// app/actions/currencies.ts

import { Action } from '@ngrx/store';


export const ActionTypes = {
  CHANGE_CURRENCY: 'Change currency',
};

export class ChangeCurrencyAction implements Action {
  type = ActionTypes.CHANGE_CURRENCY;
  constructor(public payload:string) { }
}


export type Actions =
  ChangeCurrencyAction 
``` 
### Reducer

Next, create a file `currencies.ts` in `app/reducers`. The `currencies` .

```
// app/reducers/currencies.ts

import '@ngrx/core/add/operator/select';
import { Observable } from 'rxjs/Observable';
import * as currencies from '../actions/currencies';

/*
The sate object needs to have three properties:
1. A property for keeping a list of the available currencies
2. A property for keeping the selected currency
3. A property for keeping the list of exchange rates
*/
export interface State {
  entities:Array<string>
  selectedCurrency: string | null;
  rates: Array<Object>,
};

const initialState: State = {
    entities: ['GBP', 'EUR'],
    selectedCurrency: null,
    rates: [] ,
};


export function reducer(state = initialState, action: currencies.Actions): State {
  switch (action.type) {

    case currencies.ActionTypes.CHANGE_CURRENCY: {
        return {
          entities: state.entities,
          selectedCurrency: action.payload,
          rates: state.rates
        };
    }

    default:
      return state;
  }

}

/*
 These selector functions provide access to certain slices of the currency state object.
*/

export function getCurrenciesEntities(state$: Observable<State>) {
  return state$.select(s => s.entities);
}


export function getSelectedCurrency(state$: Observable<State>) {
  return state$.select(s => s.selectedCurrency);
}

export function getRates(state$: Observable<State>) {
  return state$.select(s => s.rates);
}
```

The final step is to include the `currencies` state in the store:

```
// app/reducers/index.ts

// Import app/reducers/currencies.ts

import * as fromCurrencies from '../reducers/currencies';
//...


export interface State {
  operations: fromOperations.State;
  //Add the currencies state interface
  currencies: fromCurrencies.State;
}


const reducers = {
  operations: fromOperations.reducer,

  // Add the currency reducer
  currencies: fromCurrencies.reducer

};

//...


//Access the 'currencies' state in the application store
export function getCurrencies(state$: Observable<State>) {
  return state$.select(state => state.currencies);
}

// Access 'entities' from the 'currencies' state in the application state.
export const getCurrencyEntities = compose(fromCurrencies.getCurrenciesEntities , getCurrencies);

// Access 'selectedCurrency' from the 'currencies' state in the application state.
export const getSelectedCurrency = compose(fromCurrencies.getSelectedCurrency , getCurrencies);

// Access 'rates' from the 'currencies' state in the application state.
export const getCurrencyRates = compose(fromCurrencies.getRates , getCurrencies);






```

### Accessing the new state
Before creating a 'dumb' component to display the currency options and the selected currency, the `currencies` have to be retrieved in the  smart component (in this case `AppComponent`).

```
// app/app.component.ts


export class AppComponent {
  //...
  public currencies:Observable<string[]>;
  public selectedCurrency: Observable<string>;


  constructor(private _store: Store<fromRoot.State>) {
    //...
    this.currencies = this._store.let(fromRoot.getCurrencyEntities);
    this.selectedCurrency =this._store.let(fromRoot.getSelectedCurrency);
  }
  //...
    onCurrencySelected(currency:string) {
    this._store.dispatch(new currencies.ChangeCurrencyAction(currency))
  }
  
 }
```
 Apart from making component class variables for accessing the `currencies` in the store, there is also a function for dispatching the `ChangeCurencyAction()` when the user selects another currency.
 
Next, install [ng-bootstrap](https://ng-bootstrap.github.io/#/getting-started) in order to add good-looking, interactive radio buttons. 
```bash
 $ npm install --save @ng-bootstrap/ng-bootstrap
```

And import the `NgbModule` in the `AppModule`:
```
// app.module.ts
import {NgbModule} from '@ng-bootstrap/ng-bootstrap';

//...
@NgModule({
  declarations: [AppComponent, ...],
  imports: [NgbModule.forRoot(), ...],
  bootstrap: [AppComponent]
})
export class AppModule {
}
```

Next, create a `Currencie`component. 

```
// currencies.component.ts

import {Component, Input, ChangeDetectionStrategy, Output, EventEmitter} from '@angular/core';

@Component({
    selector: 'currencies',
    template: `
    '<div [ngModel]="selectedCurrency" (ngModelChange)="currencySelected.emit($event)" ngbRadioGroup name="radioBasic">
    <label *ngFor="let currency of currencies" class="btn btn-primary">
    <input type="radio" [value]="currency"> {{currency}}
    </label>
    </div>`,
     changeDetection: ChangeDetectionStrategy.OnPush
})
export class Currencies  {
    @Input() currencies:Array<string>;
    @Input() selectedCurrency:string;
    @Output() currencySelected = new EventEmitter();

    constructor() { }

}
```
The `Currencies` component is a 'dumb' component (has no logic) that uses `ngbRadioGroup` to display the available currencies. `(ngModelChange)` is used to emit an event to `AppComponent` every time the user clicks on the radio buttons.

Lastly, add `Currencies` to `AppModule`:

```
// app.module.ts
import {Currencies} from './currencies.component';

//...
@NgModule({
  declarations: [Currencies, ...],
  //...
})
export class AppModule {
}
```

# Effects

If you go to [http://localhost:4200/](http://localhost:4200/) now and play around with the currency buttons, you'll see that nothing special happens yet. Looking at the current state of the application, there needs to be a way to load the currency rates from a dedicated currency data API, such as [fixer.io](http://fixer.io/). 

**[Redux has its own convention for handling server-side requests](http://redux.js.org/docs/advanced/AsyncActions.html)**. It does it through using a middleware - a piece of logic that stays between the server and the reducer functions. Actions that trigger server-side requests are considered as *impure* actions, since they cannot be completely handled by the reducer.

In redux, andling server-side requests requires the implementation of three actions:

   1. **Action that indicates the start of the server-side request**:
   This action is dispatched just before the request is made. In the example application, the name of this action will be named `LOAD_CURRENCIES`. A reducer handling such action would change a dedicated state property for indicating a server-side location such as `loadingCurrencies`, which can be used to implement a loading spinner, for example.
   2. **Action that indicates a successfull request**:
   This action is dispatched when te request is done. Its payload contains the request response. The reducer simply adds the response to its corresponding state property. In the example, this action name will be called `LOAD_CURRENCIES_SUCCESS` and its payload will fill the `rates` state property with the most recent information about currency rates.
   3. **Action that indicates a failed request**
   This action is dispatched if the request fails. The action payload may contain the error reason or simply return nothing. In the example application, this action would be normally called `LOAD_CURRENCIES_FAIL`
   

#### What happens between the *Load* action and the *Load Success/Failure* action? 

This is where the middleware comes into play and more particularly, the middleware for handling side effects. In Redux, server-side requests are regarded as side effects from actions that cannot be handled through a reducer (that's why they're called *impure*. The server-side calls themselves will be handled by an Angular 2 service which will be called within the effect.

In Angular 2, there is a special package for handling side effects - [ngrx/effects](https://github.com/ngrx/effects)

Open your terminal and type:
```
npm install @ngrx/effects --save
```
### make an effects directory
### make a service
### install money.js
### add a currency pipe
