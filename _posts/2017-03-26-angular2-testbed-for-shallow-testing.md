---
layout: post
title: Using TestBeds for Shallow Testing Angular2 Components
tags:
- typescript
- angular
- TDD
---

Angular 2 now *supports shallow* testing with *TestBeds*. So we can create a *Test Module* just the same way we create a normal module, inject *Test Doubles* as services, or *Spy* on certain service functions to test Component + HTML both.

This post is to solve a few confusions around getting the *Angular 2 TestBed* to work. It should also show a complete example that works!

### expenses.component.ts
A simple *Component* that shows a list of expenses in a table. Uses `ExpensesService` to fetch the *expenses*.

This is the component I want to shallow test.

```typescript
import { Component, OnInit } from '@angular/core';
import { ExpensesService } from './expenses.service';
import { Expense } from './shared/expense.model';

@Component({
    selector: 'selector',
    templateUrl: './expenses.component.html',
    styleUrls: ['./expenses.component.css'],
    providers: [ ExpensesService ]
})
export class ExpensesComponent implements OnInit {
    public expenses: Expense[];

    constructor(private expensesService: ExpensesService) { }

    public ngOnInit() {
        this.expenses = this.expensesService.getExpenses();
    }
}
```

### expenses.component.html

```html
<table>
    <thead>
        <tr>
            <th>Id</th>
            <th>Name</th>
            <th>amount</th>
        </tr>
    </thead>

    <tbody>
        <tr *ngFor="let expense of expenses">
            <td>{% raw %}{{expense.id}}}{% endraw %}</td>
            <td>{% raw %}{{expense.name}}{% endraw %}</td>
            <td>{% raw %}{{expense.amount}}{% endraw %}</td>
        </tr>
    </tbody>
</table>
```

### expense.model.ts

```typescript
export interface Expense {
    id: Number;
    name: string;
    amount: Number;
}
```
### ExpensesService

The service returns a list of *expenses* to *ExpensesComponent*.

```typescript
import { Injectable } from '@angular/core';
import { Expense } from './shared/expense.model';

@Injectable()
export class ExpensesService {
    private expenses: Expense[];

    constructor() {
        this.expenses = [
            { id: 1, name: 'Breakfast', amount: 9.0 },
            { id: 2, name: 'Lunch', amount: 19.0 },
            { id: 3, name: 'Coffee', amount: 3.0 },
            { id: 4, name: 'Drinks', amount: 11.0 }
        ];
    }

    public getExpenses() {
        return this.expenses;
    }
}
```

### expenses.component.spec.ts

The test is almost straight foward. The target is to shallow-test the `ExpensesComponent`.

Shallow Tests are preferable over Isolated unit tests because they let us test the HTML output generated also.

> `TestBed.configureTestingModule()` should not be run on `beforeAll` - although you'd prefer doing that. Tests in each `it` *resets the `TestBed`*. That makes the properties undefined.

> Whenever the component template is in a separate file, TestBed takes time to put the component together. `beforeEach(async() => {})` is used with `TestBed.configureTestingModule().compileComponents()` to wait for that.

```typescript
import { TestBed, ComponentFixture } from '@angular/core/testing';
import { ExpensesComponent } from './expenses.component';
import { ExpensesService } from './expenses.service';
import { DebugElement } from '@angular/core';
import { Expense } from './shared/expense.model';
import { By } from '@angular/platform-browser';

describe('Component: Expenses-Shallow', () => {
    let fixture: ComponentFixture<ExpensesComponent>;
    let component: ExpensesComponent;
    let de: DebugElement;
    let expensesService: ExpensesService;
    let expenses: Expense[] = [{ id: 1, name: 'test', amount: 10 }];

    // Using beforeAll will break the tests.
    beforeEach(async() => {
        TestBed.configureTestingModule({
                declarations: [ ExpensesComponent ],
                providers: [ ExpensesService ]
        }).compileComponents();

        fixture = TestBed.createComponent(ExpensesComponent);
        component = fixture.componentInstance;
        de = fixture.debugElement;
        expensesService = TestBed.get(ExpensesService);
    });

    it('should show expense when served from service', () => {
        spyOn(expensesService, 'getExpenses')
            .and.returnValue(expenses);

        fixture.detectChanges();
        // Just to show passing tests.
        const el = de.query(By.css('td')).nativeElement;
        expect(el.textContent).toEqual('1');
    });
});
```

### Output
```text
Component: Expenses-Shallow
    ✔ should show expense when served from service
```

### Some of the issues I met while getting this to work

* *spyOn could not find an object to spy upon for foo()* when `beforeAll` was used.
* *undefined fixture*

Good luck & happy testing! :)