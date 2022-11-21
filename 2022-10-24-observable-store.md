# State Management Is Easy Now!? An Introduction to Observable Store

![Surprised man](https://res.cloudinary.com/practicaldev/image/fetch/s--QNHg0iNG--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pmt8893tgs2jmpq0n3bm.jpg)

Oh application state. The source of so many bugs. React has [Redux](https://redux.js.org/), Vue has [Pinia](https://pinia.vuejs.org/), and Angular has [NgRx](https://ngrx.io) as their most used state management libraries.

NgRx is a great solution for folks well versed in Angular-ese and can make tackling problems with enterprise-sized applications way easier. There's a very big application I work on where we as a team started using NgRx to help us out for this very reason.

I work on a team where most of the developers are hardened Java veterans and don't spend too much time on the frontend.  NgRx helped a lot but we still had to deal with reducers, actions, effects, registries, and getting it all wired up properly. As a team we had to understand new concepts and basically adopt an additional framework on top of the framework many of us were already still struggling to learn.

To their credit, the folks behind NgRx have simplified it and have removed a lot of the boilerplate code (there used to be A LOT of boilerplate).

It's much better than trying to manage state without a library by passing around properties from child to parent/parent to child or by using services. I can say from experience that doing so gets very messy very quickly if you have even a moderately complicated application.

In 2020, Dan Wahlin (a big contributor to the Angular community), did a [talk](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwjBz97yvfr6AhVpg4kEHXPbDKcQwqsBegQIDBAB&url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3Djn4AH5pGWhA&usg=AOvVaw28BB5BrnAvi0ZWEQVh4Ufw) about a new library he came up with called [Observable Store](https://github.com/DanWahlin/Observable-Store).

This was *game changing* and something I don't think got enough attention. Why? Because it's just so dang simple to use and easy to understand.

We just need a basic understanding of RxJS. 

That's it! 

No, really. Read on and I'll prove it.

To be clear, even though this post is tailored to Angular developers, this is not an Angular-only thing. It can be used with any JS framework - or without, as long as you have RxJS installed.

Even though the [documentation](https://github.com/DanWahlin/Observable-Store/blob/main/README.md) is excellent I thought I'd share a [starter project](https://github.com/stevewhitmore/observable-store-poc) and a quick rundown of how it works.

## The Rundown

Assuming you have an Angular app set up (and therefore RxJS installed), run the following command:

```bash
npm i @codewithdan/observable-store
```

Create a new class. It'll be treated as a service so it'll have that `@Injectable` decorator. Extend the `ObservableStore` class and type it with a model:

<p class="caption">customer-store.ts</p>

```typescript
export interface StoreStateModel {
  customers: CustomerModel[],
  addMode: boolean,
}

@Injectable()
export class CustomersStore extends ObservableStore<StateStoreModel> {
  initialState = {
    customers: [],
    addMode: false,
  };

  constructor() {
    super({
        logStateChanges: true,
    })
    this.setState(this.initialState, 'INIT_STATE');
  }

  get(): Observable<StoreStateModel> {
    const state = this.getState();
    return of(state);
  }
}
```

Let's pump the breaks for a sec and see where we're at.

We have a class property `initialState` that gets passed in the constructor to `setState()` along with the `INIT_STATE` action. Actions are more for the developers' convenience here and can be any arbitrary string.

We're calling `super()` since we're extending the `ObservableStore` class and passing it `logStateChanges: true`. This prints out the state in the console like so:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9i5pt2rtm8mk3c87r085.png)

That's right. No additional libraries needed. No browser extensions needed. Just a property passed to `super()`. That's pretty darn super! *(Sorry)*

Next, we have the `get()` method that gets that state and wraps it up nice in an Observable.

Now we can inject the CustomersStore into our component and capture the state changes in an Observable class property:

<p class="caption">app.component.ts</p>

```typescript
export class AppComponent {
  state$ = this.customersStore.stateChanged;

  constructor(private customersStore: CustomersStore) {}
```

So far, so good. But how do we handle getting data? There's a couple different approaches we can take. One would be to add our data service to our CustomersStore, keeping it as the single source of truth and our app nice and clean.

<p class="caption">customer-store.ts</p>

```typescript
export interface StoreStateModel {
  customers: CustomerModel[],
  addMode: boolean,
}

@Injectable()
export class CustomersStore extends ObservableStore<StateStoreModel> {
  initialState = {
    customers: [],
    addMode: false,
  };

  constructor(private dataService: DataService) {
    super({
        logStateChanges: true,
    })
    this.setState(this.initialState, 'INIT_STATE');
  }

  get(): Observable<StoreStateModel> {
    const state = this.getState();
    return of(state);
  }

  fetchData(): void {
    this.fetchSub = this.dataService.getData()
      .subscribe({
        next: (customers: CustomerModel[]) => {
          const updatedState = {
            ...this.initialState,
            customers,
          }
          this.setState(updatedState, 'FETCHED_DATA');
        },
        error: response => // do stuff to handle error
      });
  }
}
```

Back in our `AppComponent` we can fetch it in the `ngOnInit` hook.

<p class="caption">app.component.ts</p>

```typescript
export class AppComponent implements OnInit {
  state$ = this.customersStore.stateChanged;

  constructor(private customersStore: CustomersStore) {}

  ngOnInit(): void {
    this.getCustomerData();
  }

  getCustomerData(): void {
    this.customersStore.fetchData();
  }
```
> *I try to avoid putting logic directly in lifecycle hooks. I think it makes unit tests easier to write and the code easier to read.*

Now our console output looks like this:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c41u8gqz9gj5i0x0329d.png)

Great, but what about updating the state with new customers?

Let's take a look!

We should probably display our customers in its own component to keep our components nice and single-minded.

Let's update the `AppComponent`'s html to pass the state to our new component:

<p class="caption">app.component.html</p>
```html
<div *ngIf="state$ | async as state">
    <app-customers [state]="state"></app-customers>
</div>
```

<p class="caption">customers.component.ts</p>

```typescript
export class CustomersComponent implements OnChanges {
  @Input() state: StoreStateModel = { customers: [], addMode: false };
  customersData: CustomerModel[] = [];
  customersStore: CustomersStore;
  displayedColumns: string[] = ['id', 'firstName', 'lastName', 'phone', 'email', 'memberSince', 'delete'];

  constructor(customersStore: CustomersStore) {
    this.customersStore = customersStore;
  }

  ngOnChanges(changes: SimpleChanges): void {
    const stateChanges = changes['state'];
    if (stateChanges && stateChanges.currentValue) {
      this.customersData = stateChanges.currentValue.customers;
    }
  }
```

Our `customersStore` was injected as a public property so we can directly call `initAddMode()` in the html.

<p class="caption">customers.component.html</p>

```html
<!-- a table or something to display the data -->

<button
    class="addBtn"
    (click)="customersStore.initAddMode()"
    [disabled]="state.addMode">
    Add
</button>
```

<p class="caption">customers-store.ts</p>

```typescript
initAddMode(): void {
  const state = this.getState();

  const updatedState = {
    ...state,
    addMode: true
  };

  this.setState(updatedState, 'INIT_ADD_MODE');
}
```

Let's assume you have another component `AddCustomerComponent` and you have its selector in the `CustomersComponent`'s html:

```html
<div *ngIf="state.addMode">
    <app-add-customer></app-add-customer>
</div>
```

`AddCustomerComponent` would look something like this:

<p class="caption">add-customer.component.ts</p>

```typescript
export class AddCustomerComponent {
  customersStore: CustomersStore;
  addCustomerForm = this.fb.group({
    firstName: ['', Validators.required],
    lastName: ['', Validators.required],
    phone: ['', Validators.required],
    email: ['', Validators.required],
  })

  constructor(
    private fb: FormBuilder,
    customersStore: CustomersStore,
  ) {
    this.customersStore = customersStore;
  }
}
```

Like with the `CustomersComponent` we injected `customersStore` as a public property so we can directly call `addCustomer()` and `resetView()` in the html.

```html
<!-- form stuff -->
<div class="button-group">
  <button
    type="submit"
    [disabled]="!addCustomerForm.valid"
    (click)="customersStore.addCustomer(addCustomerForm.value)">
    Save
  </button>
  <button
    type="button"
    (click)="customersStore.resetView()">
    Cancel
  </button>
```

<p class="caption">customers-store.ts</p>

```typescript
addCustomer(customer: CustomerModel): void {
  const state = this.getState();
  const ids = state.customers.map(c => c.id)
  const newId = ids.length ? Math.max(...ids) + 1 : 1;
  customer.id =  newId;
  customer.memberSince = new Date();

  this.addSub = this.dataService.addCustomer(customer)
    .subscribe({
      next: (customers: CustomerModel[]) => {
      const updatedState = {
        customers: [ ...state.customers, customer ],
        addMode: false,
      };

      this.setState(updatedState, 'ADDED_CUSTOMER');
    }, 
    error: response => // do stuff to handle error
  });

}

resetView(): void {
  const state = this.getState();

  const updatedState = {
    ...state,
    addMode: false,
  };

  this.setState(updatedState, 'RESET_VIEW');
}
```

By now you should be getting the idea. For a more in-depth example feel free to take a peek at that [example project](https://github.com/stevewhitmore/observable-store-poc) I mentioned earlier.

## Bonus! Unit Tests

Unit testing Observable Store is pretty straight forward. After all, we're just dealing with RxJS and class methods.

Yes, I write my tests after I write my application code. Yes, I know it's better to write tests first and then write application code. I fully embrace unit tests (as should you as a professional) but I haven't quite made the mental switch to TDD. I'll get there, promise.

Let's start off with creating some mock data and a stub for our DataService:

<p class="caption">customers-store.spec.ts</p>

```typescript
const mockData = require('../../test-data/customers.json');
const dataServiceStub = {
    getData: () => of([]),
}
```

Next, the setup. Just like with any other service we'll want to inject this class into the TestBed:

<p class="caption">customers-store.spec.ts</p>

```typescript
describe('CustomersStore', () => {
  let customersStore: CustomersStore;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [ 
        CustomersStore,
        { provide: DataService, useValue: dataServiceStub }
       ],
    });

    customersStore = TestBed.inject(CustomersStore);
  });
```

Now we test the class like we would any other component:

```typescript
describe('addCustomer()', () => {
  it('should set the ID to 1 if no customers are present', fakeAsync(() => {
    customersStore.addCustomer(mockData[0]);

    customersStore.get()
      .subscribe({
        next: data => {
          expect(data.customers[0].id).toBe(1);
        }
       });
  }));

  it('should set the ID to 1 more than the last one', fakeAsync(() => {
    customersStore.addCustomer(mockData[0]);
    customersStore.addCustomer(mockData[1]);

    customersStore.get()
      .subscribe({
        next: data => {
          expect(data.customers[1].id).toBe(2);
        }
    });
  }));

  it('should set the "memberSince" property to today\'s date', fakeAsync(() => {
    customersStore.addCustomer(mockData[0]);

    customersStore.get()
      .subscribe({
        next: data => {
          expect(data.customers[0].memberSince).toEqual(new Date());
        }
      });
  }));
});
```

That's all folks! I'm curious to hear your thoughts or experiences with Observable Store. Let me know!

