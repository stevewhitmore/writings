# Angular Unit Tests: Common Pain Points

![Man about to rage quit](https://res.cloudinary.com/practicaldev/image/fetch/s--Ro6201-x--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/leik1y3s418tcc1lava4.jpg)

One of the biggest struggles I see when people are first learning Angular is dealing with unit tests. They can be painful when you're new to the framework or are used to dealing with backend code.

The biggest pain points I think people run into are:

- The setup
- The scope
- Testing Observables
- Dealing with lifecycle hooks

I wanted to post a quick blurb that may come in handy for those who are struggling today. Hopefully after reading this you'll feel a bit more comfortable when facing Angular unit tests in the future.

## The Setup

Keep it simple! This tends to be the first place people get tangled up. Only set up what you *absolutely need*.

Golden rules to follow for setting up your tests:

1. Keep it simple.
2. Don't import modules unless absolutely necessary (e.g. `ReactiveFormsModule` if you have form elements in the component or `RouterModule` if you have `routerLinks`).
3. Never use the `NO_ERRORS_SCHEMA`.
4. Never provide real services when testing components, even if they don't make http calls.

I can't overemphasize the first rule. Tests, and their setup, should be simple. Regardless of how complicated an application or its components may be.

Let's say we have a component that looks something like this:

```typescript
export class CustomerSummaryComponent implements OnInit, OnDestroy {
  updateListenerSub = new Subscription();
  customerData$: Observable<CustomerDataModel> | undefined;

  constructor(
    private activatedRoute: ActivatedRoute,
    private customerDataService: CustomerDataService,
    private mmmToastService: MmmToastService,
  ) {}

  ngOnInit(): void {
    this.getCustomerData();
    this.listenForUpdates();
  }

  getCustomerData() {
    const customerNumber = this.activatedRoute.snapshot.params['customerNumber'];
    const customerLastName = this.activatedRoute.snapshot.params['lastName'];

    this.customerData$ = this.customerDataService.getCustomerData(customerNumber, customerLastName);
  }

  listenForUpdates() {
    this.updateListenerSub = this.customerDataService.customerUpdated$
      .subscribe({
        next: (response: CustomerUpdateModel) => this.showToastMessage(response),
        error: (error: any) => this.showToastMessage(error),
      });
  }

  showToastMessage(response: CustomerUpdateModel) {
    const type = response.status === 200 ? 'success' : 'error';

    this.mmmToastService.addToast({ type, message: response.message });
  }

  ngOnDestroy(): void {
    this.updateListenerSub.unsubscribe();
  }
}
```

There are a few things going on here. We have a couple services, we're dealing with the ActivatedRoute, and we have a couple lifecycle hooks.

Here's what the setup should look like for our tests:

```typescript
describe('CustomerSummaryComponent', () => {
  let component: CustomerSummaryComponent;
  let fixture: ComponentFixture<CustomerSummaryComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [ CustomerSummaryComponent ],
      providers: [
        { provide: ActivatedRoute, useValue: activatedRouteStub },
        { provide: CustomerDataService, useValue: customerDataServiceStub },
        { provide: MmmToastServiceStub, useValue: mmmToastServiceStub },
      ],
    });

    fixture = TestBed.createComponent(CustomerSummaryComponent);
    component = fixture.componentInstance;
  });
```

That's it. We have stubs for our services and we're loading up our component into the test harness. We don't need to care about anything else at this point.

You should be mindful of importing modules because doing so can put way more into your test harness than what's needed and can ultimately lead to unintended side-effects.

The `NO_ERRORS_SCHEMA` is a big blanket that'll mask real errors in your application code. You can read more about what it does [here](https://angular.io/api/core/NO_ERRORS_SCHEMA), but I'd recommend just staying away from it altogether.

Services generally either have http calls or business logic in them. Your unit tests for this component shouldn't know or care about any of that. Which leads me to my next point...

## The Scope

Keep the scope of your tests as narrow as possible. The more narrow your test focus is, the bigger the return you'll get. They'll also be easier to write and manage.

Let's take another look at the `getCustomerData()` method from our component above.

```typescript
getCustomerData() {
    const customerNumber = this.activatedRoute.snapshot.params['customer-number'];
    const customerLastName = this.activatedRoute.snapshot.params['last-name'];

    this.customerData$ = this.customerDataService.getCustomerData(customerNumber, customerLastName);
}
```

In this method we're grabbing some data from the ActivatedRoute, passing it to our service, and storing the result in an Observable. 

Before we go any further let me outline some things that we're NOT trying to test:

- How ActivatedRoute behaves
- What happens in `customerDataService.getCustomerData()`

We only care that we're making a call to our service, what we're passing to it, and what we do in response to that call.

That's why we use stubs. Our stubs will just give us what we want so we can focus on what we *do* care about. 

Here's a good set of tests for our `getCustomersData()` method:

```typescript
beforeEach(() => {
    // ...
    activatedRouteStub.testParams = { customerNumber: '1234', customerLastName: 'Smith' };
});

describe('getCustomerData()', () => {
    it('should call "customerDataService.getCustomerData()" with the route params', () => {
        const spy = spyOn(customerDataServiceStub, 'getCustomerData');

        component.getCustomerData();

        expect(spy).toHaveBeenCalledOnceWith('1234', 'Smith');
    });

    it('should put the returned value in an observable', () => {
      spyOn(customerDataServiceStub, 'getCustomerData').and.returnValue(of());
      component.getCustomerData();

      expect(component.customerData$).toBeTruthy();
    });
});
```

Each test has a narrow focus and is following that AAA structure (Assemble, Act, Assert). Our `activatedRouteStub` and `customerDataServiceStub` are just bouncing back what we need to make our assertions. They would look something like this:

```typescript
export class ActivatedRouteStub {
    private subject = new BehaviorSubject(this.testParams);
    private _testParams: any;
  
    get testParams() {
        return this._testParams;
    }

    set testParams(queryParams: any) {
        this._testParams = queryParams;
        this.subject.next(queryParams);
    }
 
    get snapshot() {
        return {
          queryParams: this.testParams,
          params: this.testParams,
        };
      }    
}
```

```typescript
export class CustomerDataServiceStub {
    getCustomerData(customerNumber: string, customerLastName: string): any {}
}
```

Put the bare minimum in your stubs. They should be "stupid"!

## Testing Observables

This is a big pain point. When we're dealing in async space we risk subjecting ourselves to test bleedover and unintended side-effects. That's why it's important to wrap our tests in `fakeAsync` and to always `flush` when we're done :grin:.

Let's take a look at a couple scenarios where we're forced to deal with an Observable:

With that first method `getCustomerData()` we originally just asserted that our `customerData$` property wasn't falsy. That's fine, but it would be better if we could crack open that Observable and make sure what we expect is inside:

```typescript
const mockCustomer = {
    firstName: 'John',
    lastName: 'Doe',
    customerNumber: '12345',
    memberSince: new Date('11/11/2011'),
    email: 'johndoe@email.com',
    phone: '555-777-9999',
};

describe('getCustomerData()', () => {
    //...
    
    it('should put the returned value in an observable', fakeAsync(() => {
      spyOn(customerDataServiceStub, 'getCustomerData').and.returnValue(of(mockCustomer));
      
      component.getCustomerData();
      
      component.customerData$
        .subscribe({
          next: (response: CustomerDataModel) => {
            expect(response).toEqual(mockCustomer)
          }
        });
    
      flush();
    }));
});
```

By wrapping our test in `fakeAsync` we're able to test asynchronous code in a synchronous way. There are plenty of articles out there describing how this works in more detail and I encourage you to read up on it. In the meantime, just think of it as a nice tool to keep headaches in our tests to a minimum. 

Calling `flush()` either at the end of each test wrapped in `fakeAsync` or in a `afterEach()` method will ensure that anything going on in async space is wrapped up. This will help prevent test bleedover.

---

Our `listenForUpdates()` is another good scenario for testing in async space.

```typescript
listenForUpdates() {
    this.updateListenerSub = this.customerDataService.customerUpdated$
        .subscribe({
            next: (response: CustomerUpdateModel) => this.showToastMessage(response),
            error: (error: any) => this.showToastMessage(error),
        });
}
```

We can maintain that AAA structure if `customerUpdated$` is a BehaviorSubject.

```typescript
  describe('listenForUpdates()', () => {
    it('should call "showToastMessage()" with response', fakeAsync(() => {
      const spy = spyOn(component, 'showToastMessage');
      customerDataServiceStub.update(); // trigger ".next()" on the observable to make it emit

      component.listenForUpdates();

      expect(spy).toHaveBeenCalledTimes(1);
    }));
  });
```

*customer-data-service.stub.ts*

```typescript
export class CustomerDataServiceStub {
    customerUpdated$ = new BehaviorSubject({message: 'Yay!', status: 200});

    getCustomerData(customerNumber: string, customerLastName: string): any {}

    update() {
        this.customerUpdated$.next({message: 'updated', status: 200});
    }
}
```

This is because BehaviorSubjects immediately emit values when they're subscribed to. If `customerUpdated$` was a regular Subject we'd need to break that AAA structure a little bit:

```typescript
  describe('listenForUpdates()', () => {
    it('should call "showToastMessage()" with response', fakeAsync(() => {
      const spy = spyOn(component, 'showToastMessage');
      customerDataServiceStub.update();

      customerDataServiceStub.customerUpdated$
        .subscribe({
          next: () => expect(spy).toHaveBeenCalledTimes(1)
        });

      component.listenForUpdates();
    }));
  });
```

The sooner you can get comfortable with Observables, the better. They can feel daunting at first but once you get used to them they really are magical. Some good points to focus on would be:

- Learning the differences between the more widely used Subject types (Subject, BehaviorSubject, ReplaySubject)
- Learning the most commonly used operators (map, mergeMap/switchMap, combineLatest, takeUntil, to name a few)

## Dealing with lifecycle hooks

In short, don't test these. We want to test our code - not the framework we're using.

Let's change our CustomerSummary component a bit so that it's recieving data from a parent instead of fetching said data. We'll have it capture
the customer's first name into a class property as well:

```typescript
export class CustomerSummaryComponent implements OnInit, OnDestroy {
  updateListenerSub = new Subscription();
  @Input() customerData: CustomerDataModel;
  firstName: string;

  constructor(
    private activatedRoute: ActivatedRoute,
    private customerDataService: CustomerDataService,
    private mmmToastService: MmmToastService,
  ) {}

  ngOnChanges(changes: SimpleChanges): void {
    const customerChange = changes['customerData'];

    if (customerChange && customerChange.currentValue) {
        this.firstName = customerChange.currentValue.firstName;
    }
  }

  //...
}
```
I see a lot of tests like below:

```typescript
describe('ngOnChanges()', () => {
    it('should capture the first name of the customer', () => {
        const changes = { customerData: new SimpleChange(null, mockCustomer, true) };

        component.ngOnChanges(changes);

        expect(component.firstName).toBe('John');
    });
});
```

You're not really getting any real value out of a test like this. If there's logic in the lifecycle hook you should break the logic out into their own methods. It'll make the application code easier to test, manage, and read!

```typescript
ngOnChanges(changes: SimpleChanges): void {
    const customerChange = changes['customerData'];

    if (customerChange && customerChange.currentValue) {
        const currentCustomer = customerChange.currentValue;
        this.firstName = currentCustomer.firstName;

        this.updateListenerSub = this.customerDataService.customerUpdated$
            .subscribe({
                next: (response: CustomerUpdateModel) => this.showToastMessage(response),
                error: (error: any) => this.showToastMessage(error),
            });
    }
  }
```

becomes

```typescript
ngOnChanges(changes: SimpleChanges): void {
    const customerChange = changes['customerData'];

    if (customerChange && customerChange.currentValue) {
        this.captureFirstName(customerChange.currentValue);
        this.listenForupdates();
    }
}

captureFirstName(currentCustomer) {
    this.firstName = currentCustomer.firstName;
}

listenForUpdates() {
    this.updateListenerSub = this.customerDataService.customerUpdated$
        .subscribe({
            next: (response: CustomerUpdateModel) => this.showToastMessage(response),
            error: (error: any) => this.showToastMessage(error),
        });
}
```

## Summary 

Your key takeways from this post should be

- Keep your setup simple
- Keep your test scope narrow
- Observables are manageable with `fakeAsync` and `flush`
- Be mindful of *what* you're testing and why

Happy testing!
