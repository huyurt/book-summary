### [NgModules](https://angular.io/guide/ngmodules)

**NgModules** configure the injector and the compiler and help organize related things together.

An NgModule is a class marked by the `@NgModule` decorator. `@NgModule` takes a metadata object that describes how to compile a component's template and how to create an injector at runtime. It identifies the module's own components, directives, and pipes, making some of them public, through the `exports` property, so that external components can use them. `@NgModule` can also add service providers to the application dependency injectors.

Modules can be loaded eagerly when the application starts or lazy loaded asynchronously by the router.

NgModule metadata does the following:

- Declares which components, directives, and pipes belong to the module.
- Makes some of those components, directives, and pipes public so that other module's component templates can use them.
- Imports other modules with the components, directives, and pipes that components in the current module need.
- Provides services that other application components can use.

```typescript
// imports
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';

// @NgModule decorator with its metadata
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}
```



### Component Selector

````typescript
// server.component.ts
@Component({
  selector: '[app-server]',
  template: `
	<p>server works!</p>
  `
})

// app.component.ts
@Component({
  selector: 'app-root',
  template: `
	<h3>I'm in the AppComponent!</h3>
    <div app-server></div>
  `
})
````



### [Data Binding](https://angular.io/guide/binding-syntax)

Data binding automatically keeps your page up-to-date based on your application's state. The target of a data binding can be a property, an event, or an attribute name.

Angular provides three categories of data binding according to the direction of data flow:

- From the source to view
- From view to source
- In a two way sequence of view to source to view

| Type                                         | Syntax                                                       | Category                                |
| :------------------------------------------- | :----------------------------------------------------------- | :-------------------------------------- |
| Interpolation Property Attribute Class Style | `{{expression}} [target]="expression" bind-target="expression"` | One-way from data source to view target |
| Event                                        | `(target)="statement" on-target="statement"`                 | One-way from view target to data source |
| Two-way                                      | `[(target)]="expression" bindon-target="expression"`         | Two-way                                 |



| Type      | Target                                                 | Examples                                                     |
| :-------- | :----------------------------------------------------- | :----------------------------------------------------------- |
| Property  | Element property Component property Directive property | `src`, `hero`, and `ngClass` in the following:`<img [src]="heroImageUrl"> <app-hero-detail [hero]="currentHero"></app-hero-detail> <div [ngClass]="{'special': isSpecial}"></div>` |
| Event     | Element event Component event Directive event          | `click`, `deleteRequest`, and `myClick` in the following:`<button (click)="onSave()">Save</button> <app-hero-detail (deleteRequest)="deleteHero()"></app-hero-detail> <div (myClick)="clicked=$event" clickable>click me</div>` |
| Two-way   | Event and property                                     | `<input [(ngModel)]="name">`                                 |
| Attribute | Attribute (the exception)                              | `<button [attr.aria-label]="help">help</button>`             |
| Class     | `class` property                                       | `<div [class.special]="isSpecial">Special</div>`             |
| Style     | `style` property                                       | <button [style.color]="isSpecial ? 'red' : 'green'">`        |



#### Property Binding

````typescript
@Component({
  template: `
	<button class="btn btn-primary" [disabled]="!allowNewServer">Add Server</button>
 	<p [innerText]="allowNewServer"></p>
 `,
})
export class ServerComponent implements OnInit {
  allowNewServer = false;

  constructor() {
    setTimeout(() => {
      this.allowNewServer = true;
    }, 2000);
  }
}
````

<img src="https://1.bp.blogspot.com/-EpG34MTC6uA/YTE2yHTJxKI/AAAAAAAADRY/2Q4sq5dbYV0h6g2u9RJaUQdeV6MpZ2UjQCLcBGAsYHQ/s0/20210902233943224.gif">



##### Custom property binding

###### Input

The `@Input()` decorator in a child component or directive signifies that the property can receive its value from its parent component.

````typescript
@Component({
  template: `
	<app-server [element]="serverElement" [srvElement]="serverElement"></app-server>
 `,
})
export class ServerComponent {
  @Input() element: {type: string, name: string, content: string};
  @Input('srvElement') element2: {type: string, name: string, content: string};
}
````



#### [Event Binding](https://angular.io/guide/event-binding-concepts)

````typescript
@Component({
  template: `
	<button class="btn btn-primary" (click)="onCreateServer()">Add Server</button>
	<p>{{serverCreationStatus}}</p>
 `,
})
export class ServerComponent {
  serverCreationStatus = 'No server was created!';

  onCreateServer() {
    this.serverCreationStatus = 'Server was created.';
  }
}
````

<img src="https://1.bp.blogspot.com/-PMmtkwBDUi4/YTE7gZAzYHI/AAAAAAAADRg/s1e-aXt_gOUAC8g61RpRKaezFzCQwTf4ACLcBGAsYHQ/s0/20210902235943770.gif">



##### Handling events

A common way to handle events is to pass the event object, `$event`, to the method handling the event. The `$event` object often contains information the method needs, such as a user's name or an image URL.

The target event determines the shape of the `$event` object. If the target event is a native DOM element event, then `$event` is a [DOM event object](https://developer.mozilla.org/en-US/docs/Web/Events), with properties such as `target` and `target.value`.

In the following example the code sets the `<input>` `value` property by binding to the `name` property.

````typescript
@Component({
  template: `
	<input [value]="currentItem.name"
       (input)="currentItem.name=getValue($event)">
 `,
})
export class ServersComponent {
  getValue(event: Event): string {
    return (event.target as HTMLInputElement).value;
  }
}
````



````typescript
@Component({
  template: `
	<label>Server Name</label>
	<input type="text" class="form-control" (input)="onUpdateServerName($event)">
	<p>{{serverName}}</p>
 `,
})
export class ServersComponent {
  serverName = '';

  onUpdateServerName(event: Event) {
    this.serverName = (<HTMLInputElement>event.target).value;
  }
}
````

<img src="https://1.bp.blogspot.com/-bSrVXr7S8CE/YTE_qURRqwI/AAAAAAAADRo/kz6V9pHzolMY1N5oVH5qzHzw0LG99QIhgCLcBGAsYHQ/s0/20210903001720112.gif">



##### Custom event binding

###### Output

The `@Output()` decorator in a child component or directive lets data flow from the child to the parent.

````typescript
// parent.component
@Component({
  template: `
	<app-child (serverCreated)="onServerAdded($event)" (bpCreated)="onBlueprintAdded($event)"></app-child>
 `,
})
export class ParentComponent {
  serverElements = [{type: 'server', name: 'Testserver', content: 'Just a test!'}];

  onServerAdded(serverData: {serverName: string, serverContent: string}) {
    this.serverElements.push({
        type: 'server',
        name: serverData.serverName,
        content: serverData.serverContent
    });
  }
    
  onBlueprintAdded(blueprintData: {serverName: string, serverContent: string}){
    this.serverElements.push({
        type: 'blueprint',
        name: blueprintData.serverName,
        content: blueprintData.serverContent
    });
  }
}

// child.component
@Component({
  selector: 'app-child'
})
export class ChildComponent {
  @Output() serverCreated = new EventEmitter<{serverName: string, serverContent: string}>();
  @Output('bpCreated') blueprintCreated = new EventEmitter<{serverName: string, serverContent: string}>();
  newServerName = '';
  newServerContent = '';

  onAddServer() {
    this.serverCreated.emit({
        serverName: this.newServerName,
        serverContent: this.newServerContent,
    });
  }
  
  onAddBlueprint(){
    this.blueprintCreated.emit({
        serverName: this.newServerName,
        serverContent: this.newServerContent,
    });
  }
}
````



#### Two Way Data Binding

##### [[(ngModel)]](https://angular.io/api/forms/NgModel)

Creates a `FormControl` instance from a domain model and binds it to a form control element.

The `FormControl` instance tracks the value, user interaction, and validation status of the control and keeps the view synced with the model. If used within a parent form, the directive also registers itself with the form as a child control.

This directive is used by itself or as part of a larger form. Use the `ngModel` selector to activate it.

It accepts a domain model as an optional `Input`. If you have a one-way binding to `ngModel` with `[]` syntax, changing the domain model's value in the component class sets the value in the view. If you have a two-way binding with `[()]` syntax, the value in the UI always syncs back to the domain model in your class.


```typescript
@Component({
  template: `
    <input type="text" [(ngModel)]="name" #ctrl="ngModel" required>
	<input type="text" [ngModel]="name">
	
	<p>{{name}}</p>
	<p>Valid: {{ ctrl.valid }}</p>
	
	<button (click)="setValue()">Set value</button>
  `,
})
export class ServerComponent {
  name: string = '';

  setValue() {
    this.name = 'Hudayfe';
  }
}
```

<img src="https://1.bp.blogspot.com/-KpF5TLlMaU0/YTEd5qg8MMI/AAAAAAAADRQ/UTyV0TCWmpAQNHRKOc2iQ_ngufpJVNoKwCLcBGAsYHQ/s0/20210902215012325.gif">



### Directives

##### [NgIf](https://angular.io/api/common/NgIf)

````typescript
@Component({
  template: `
	<p *ngIf="serverCreated; else noServer">Server was created, server name is {{serverName}}</p>
	<ng-template #noServer>
  		<p>No server was created!</p>
	</ng-template>
 	
	<button class="btn btn-primary" (click)="onCreateServer()">Add Server</button>
 `,
})
export class ServerComponent {
  serverName = '';
  serverCreated = false;

  onCreateServer() {
    this.serverCreated = true;
    this.serverCreationStatus = 'Server was created.';
  }
}
````



````typescript
<div *ngIf="condition; then thenBlock else elseBlock"></div>
<ng-template #thenBlock>Content to render when condition is true.</ng-template>
<ng-template #elseBlock>Content to render when condition is false.</ng-template>


<ng-template [ngIf]="heroes" [ngIfElse]="loading">
 <div class="hero-list">
  ...
 </div>
</ng-template>

<ng-template #loading>
 <div>Loading...</div>
</ng-template>
````



##### [NgStyle](https://angular.io/api/common/NgStyle)

Set the font of the containing element to the result of an expression.

```
<some-element [ngStyle]="{'font-style': styleExp}">...</some-element>
```

Set the width of the containing element to a pixel value returned by an expression.

```
<some-element [ngStyle]="{'max-width.px': widthExp}">...</some-element>
```

Set a collection of style values using an expression that returns key-value pairs.

```
<some-element [ngStyle]="objExp">...</some-element>
```



````typescript
@Component({
  template: `
	<p [ngStyle]="{backgroundColor: getColor()}">Server is {{getServerStatus()}}</p>
 `,
})
export class ServerComponent {
  serverStatus: string = 'offline';

  constructor() {
    this.serverStatus = Math.random() > 0.5 ? 'online' : 'offline';
  }

  getServerStatus() {
    return this.serverStatus;
  }

  getColor() {
    return this.serverStatus === 'online' ? 'green' : 'red';
  }
}
````



##### [NgClass](https://angular.io/api/common/NgClass)

The CSS classes are updated as follows, depending on the type of the expression evaluation:

- `string` - the CSS classes listed in the string (space delimited) are added,
- `Array` - the CSS classes declared as Array elements are added,
- `Object` - keys are CSS classes that get added when the expression given in the value evaluates to a truthy value, otherwise they are removed.

```typescript
<some-element [ngClass]="'first second'">...</some-element>

<some-element [ngClass]="['first', 'second']">...</some-element>

<some-element [ngClass]="{'first': true, 'second': true, 'third': false}">...</some-element>

<some-element [ngClass]="stringExp|arrayExp|objExp">...</some-element>

<some-element [ngClass]="{'class1 class2 class3' : true}">...</some-element>
```



````typescript
@Component({
  template: `
	<p [ngStyle]="{backgroundColor: getColor()}" [ngClass]="{online: serverStatus === 'online'}">Server is {{getServerStatus()}}</p>
 `,
  styles: [`
    .online {
      color: white;
    }
  `]
})
export class ServerComponent {
  serverStatus: string = 'offline';

  constructor() {
    this.serverStatus = Math.random() > 0.5 ? 'online' : 'offline';
  }

  getServerStatus() {
    return this.serverStatus;
  }

  getColor() {
    return this.serverStatus === 'online' ? 'green' : 'red';
  }
}
````



##### [NgForOf](https://angular.io/api/common/NgForOf)

````typescript
@Component({
  template: `
	<p *ngFor="let server of servers">{{server}}</p>
 `
})
export class ServerComponent {
  servers = ['TestServer', 'TestServer 2'];
}
````



##### [NgSwitch](https://angular.io/api/common/NgSwitch)

````typescript
@Component({
  template: `
	<div [ngSwitch]="value">
		<p *ngSwitchCase="5">Value is {{value}}</p>
		<p *ngSwitchCase="10">Value is {{value}}</p>
		<p *ngSwitchCase="100">Value is {{value}}</p>
		<p *ngSwitchDefault>Value is Default</p>
	</div>
 `
})
export class ServerComponent {
  value = 5;
}
````



##### [NgClass](https://angular.io/api/common/NgClass)

**NgClass,** adds and removes CSS classes on an HTML element.

````typescript
<some-element [ngClass]="'first second'">...</some-element>

<some-element [ngClass]="['first', 'second']">...</some-element>

<some-element [ngClass]="{'first': true, 'second': true, 'third': false}">...</some-element>

<some-element [ngClass]="stringExp|arrayExp|objExp">...</some-element>

<some-element [ngClass]="{'class1 class2 class3' : true}">...</some-element>


@Component({
  template: `
	<li [ngClass]="{odd: odd % 2 === 0}"></li>
 `,
  styles: [`
    .odd {
      color: red;
    }
  `]
})
````



##### [NgStyle](https://angular.io/api/common/NgStyle)

**NgStyle,** an attribute directive that updates styles for the containing HTML element. Sets one or more style properties, specified as colon-separated key-value pairs. The key is a style name, with an optional `.<unit>` suffix (such as 'top.px', 'font-style.em'). The value is an expression to be evaluated. The resulting non-null value, expressed in the given unit, is assigned to the given style property. If the result of evaluation is null, the corresponding style is removed.

````typescript
<some-element [ngStyle]="{'font-style': styleExp}">...</some-element>

<some-element [ngStyle]="{'max-width.px': widthExp}">...</some-element>

<some-element [ngStyle]="objExp">...</some-element>


@Component({
  template: `
	<li [ngStyle]="{backgroundColor: odd % 2 === 0 ? 'yellow' : 'transparent'}"></li>
 `
})
````



#### [Create Attribute Directive](https://angular.io/guide/attribute-directives)

````typescript
@Directive({
  selector: '[appHighlight]'
})
export class HighlightDirective implements OnInit {
  constructor(private elementRef: ElementRef, private renderer: Renderer2) {
    el.nativeElement.style.backgroundColor = 'yellow';
  }
    
  ngOnInit(): void {
    this.renderer.setStyle(this.elementRef.nativeElement, 'background-color', 'orange');
  }
}


@Component({
  template: `
	<p appHighlight>Style me with basic directive!</p>
 `
})
````

![](https://1.bp.blogspot.com/-iWTNVxzBWZw/YTfE_4MRrCI/AAAAAAAADR4/JG8uoyompc0r09uwG76Kiz1pbXqF1eebwCLcBGAsYHQ/s0/20210907230000212.png)



#### [Create Structural Directive](https://angular.io/guide/structural-directives)

````typescript
@Directive({
  selector: '[appUnless]'
})
export class UnlessDirective {
  @Input() set appUnless(condition: boolean){
    if(!condition) {
      this.vcRef.createEmbeddedView(this.templateRef);
    } else {
      this.vcRef.clear();
    }
  }
  
  constructor(private templateRef: TemplateRef<any>, private vcRef: ViewContainerRef) {
  }
}


@Component({
  template: `
	<div *appUnless="onlyOdd">
		...
	</div>
 `
})
````



### View Encapsulation

In Angular, component CSS styles are encapsulated into the component's view and don't affect the rest of the application.

To control how this encapsulation happens on a *per component* basis, you can set the *view encapsulation mode* in the component metadata.

- `ShadowDom` view encapsulation uses the browser's native shadow DOM implementation to attach a shadow DOM to the component's host element, and then puts the component view inside that shadow DOM. The component's styles are included within the shadow DOM.
- `Emulated` view encapsulation (the default) emulates the behavior of shadow DOM by preprocessing (and renaming) the CSS code to effectively scope the CSS to the component's view.
- `None` means that Angular does no view encapsulation. Angular adds the CSS to the global styles. The scoping rules, isolations, and protections discussed earlier don't apply. This mode is essentially the same as pasting the component's styles into the HTML.

````typescript
@Component({
  selector: 'app-shadow-dom-encapsulation',
  template: `
    <h2>ShadowDom</h2>
    <div class="shadow-message">Shadow DOM encapsulation</div>
    <app-emulated-encapsulation></app-emulated-encapsulation>
    <app-no-encapsulation></app-no-encapsulation>
  `,
  styles: ['h2, .shadow-message { color: blue; }'],
  encapsulation: ViewEncapsulation.ShadowDom,
})
export class ShadowDomEncapsulationComponent { }
````



### Access Local Reference

Template variables help you use data from one part of a template in another part of the template. Use template variables to perform tasks such as respond to user input or finely tune your application's forms.

````typescript
@Component({
  template: `
	<input type="text" #serverNameInput>
	<button (click)="onAddServer(serverNameInput)">Add Server</button>
 `
})
export class ChildComponent {
  @Output() serverCreated = new EventEmitter<{serverName: string, serverContent: string}>();
  newServerContent = '';

  onAddServer(nameInput: HTMLElementInput) {
    this.serverCreated.emit({
        serverName: nameInput.value,
        serverContent: this.newServerContent,
    });
  }
}
````



### Access Using ViewChild

Property decorator that configures a view query. The change detector looks for the first element or the directive matching the selector in the view DOM. If the view DOM changes, and a new child matches the selector, the property is updated.

View queries are set before the `ngAfterViewInit` callback is called.

**Metadata Properties**:

- **selector** - The directive type or the name used for querying.
- **read** - Used to read a different token from the queried elements.
- **static** - True to resolve query results before change detection runs, false to resolve after change detection. Defaults to false.

With Angular 8, that should be `@ViewChild('...', {static: true})` since we'll also use the selected element in `ngOnInit`.

````typescript
@Component({
  template: `
	<input type="text" #serverNameInput>
	<input #serverContentInput type="text">
	<button (click)="onAddServer(serverNameInput)">Add Server</button>
 `
})
export class ChildComponent {
  @Output() serverCreated = new EventEmitter<{serverName: string, serverContent: string}>();
  @ViewChild('serverContentInput', { static: true }) serverName: ElementRef;

  onAddServer(nameInput: HTMLElementInput) {
    this.serverCreated.emit({
        serverName: nameInput.value,
        serverContent: this.serverName.nativeElement.value,
    });
  }
}
````



### Access Using ContentChild

Use to get the first element or the directive matching the selector from the content DOM. If the content DOM changes, and a new child matches the selector, the property will be updated.

Content queries are set before the `ngAfterContentInit` callback is called.

Does not retrieve elements or directives that are in other components' templates, since a component's template is always a black box to its ancestors.

**Metadata Properties**:

- **selector** - The directive type or the name used for querying.
- **read** - Used to read a different token from the queried element.
- **static** - True to resolve query results before change detection runs, false to resolve after change detection. Defaults to false.

With Angular 8, that should be `@ViewChild('...', {static: true})` since we'll also use the selected element in `ngOnInit`.

````typescript
@Component({
  template: `
	<p #contentParagraph>
		<strong *ngIf="serverElement.type === 'server'">{{serverElement.content}}</strong>
		<em *ngIf="serverElement.type === 'blueprint'">{{serverElement.content}}</em>
	</p>
`
})
export class ChildComponent implements OnInit {
  @ContentChild('contentParagraph', { static: true }) paragraph: ElementRef;

  ngOnInit() {
      console.log('Text Content of paragraph: ' + this.paragraph.nativeElement.textContent);
  }
}
````



### [Lifecycle Hooks](https://angular.io/guide/lifecycle-hooks)

A component instance has a lifecycle that starts when Angular instantiates the component class and renders the component view along with its child views. The lifecycle continues with change detection, as Angular checks to see when data-bound properties change, and updates both the view and the component instance as needed. The lifecycle ends when Angular destroys the component instance and removes its rendered template from the DOM. Directives have a similar lifecycle, as Angular creates, updates, and destroys instances in the course of execution.

| Hook method               | Purpose                                                      | Timing                                                       |
| :------------------------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `ngOnChanges()`           | Respond when Angular sets or resets data-bound input properties. The method receives a `SimpleChanges` object of current and previous property values. Note that this happens very frequently, so any operation you perform here impacts performance significantly. | Called after a bound input property changes. Called before `ngOnInit()` (if the component has bound inputs) and whenever one or more data-bound input properties change.Note that if your component has no inputs or you use it without providing any inputs, the framework will not call `ngOnChanges()`. |
| `ngOnInit()`              | Initialize the directive or component after Angular first displays the data-bound properties and sets the directive or component's input properties. | Called once the component is initialized. Called once, after the first `ngOnChanges()`. `ngOnInit()` is still called even when `ngOnChanges()` is not (which is the case when there are no template-bound inputs). |
| `ngDoCheck()`             | Detect and act upon changes that Angular can't or won't detect on its own. | Called during every change detection run. Called immediately after `ngOnChanges()` on every change detection run, and immediately after `ngOnInit()` on the first run. |
| `ngAfterContentInit()`    | Respond after Angular projects external content into the component's view, or into the view that a directive is in. | Called after content (ng-content) has been projected into view. Called *once* after the first `ngDoCheck()`. |
| `ngAfterContentChecked()` | Respond after Angular checks the content projected into the directive or component. | Called every time the projected content has been checked. Called after `ngAfterContentInit()` and every subsequent `ngDoCheck()`. |
| `ngAfterViewInit()`       | Respond after Angular initializes the component's views and child views, or the view that contains the directive. | Called after the component’s view (and child views) has been initialized. Called *once* after the first `ngAfterContentChecked()`. |
| `ngAfterViewChecked()`    | Respond after Angular checks the component's views and child views, or the view that contains the directive. | Called every time the view (and child views) have been checked. Called after the `ngAfterViewInit()` and every subsequent `ngAfterContentChecked()`. |
| `ngOnDestroy()`           | Cleanup just before Angular destroys the directive or component. Unsubscribe Observables and detach event handlers to avoid memory leaks. | Called once the component is about to be destroyed. Called immediately before Angular destroys the directive or component. |

#### Cleaning up on instance destruction

Put cleanup logic in `ngOnDestroy()`, the logic that must run before Angular destroys the directive.

This is the place to free resources that won't be garbage-collected automatically. You risk memory leaks if you neglect to do so.

- Unsubscribe from Observables and DOM events.
- Stop interval timers.
- Unregister all callbacks that the directive registered with global or application services.

The `ngOnDestroy()` method is also the time to notify another part of the application that the component is going away.



### [Renderer2](https://angular.io/api/core/Renderer2)

Extend this base class to implement custom rendering. By default, Angular renders a template into DOM. You can use custom rendering to intercept rendering calls, or to render to something other than DOM.



### [HostListener](https://angular.io/api/core/HostListener)

Decorator that declares a DOM event to listen for, and provides a handler method to run when that event occurs.

Angular invokes the supplied handler method when the host element emits the specified event, and updates the bound element with the result.

If the handler method returns false, applies `preventDefault` on the bound element.

| Option                                                       | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`eventName?`](https://angular.io/api/core/HostListener#eventName) | The DOM event to listen for.                                 |
| [`args?`](https://angular.io/api/core/HostListener#args)     | A set of arguments to pass to the handler method when the event occurs. |

````typescript
@Component({
  template: `
	<h1>{{counter}} number of times!</h1> Press any key to increment the counter.
`
})
export class AppComponent {
  counter = 0;
  @HostListener('window:keydown', ['$event'])
  handleKeyDown(event: KeyboardEvent) {
    this.counter++;
  }
}
````



### [HostBinding](https://angular.io/api/core/HostBinding)

Decorator that marks a DOM property as a host-binding property and supplies configuration metadata. Angular automatically checks host property bindings during change detection, and if a binding changes it updates the host element of the directive.

````typescript
@Component({
  template: `
	<h1>{{counter}} number of times!</h1> Press any key to increment the counter.
`
})
export class AppComponent {
  @HostBinding('style.backgroundColor') backgroundColor: string = 'transparent';
    
  //constructor(private elementRef: ElementRef, private renderer: Renderer2) {
  //}
    
  @HostListener('mouseenter') mouseover(eventData: Event){
    //this.renderer.setStyle(this.elementRef.nativeElement, 'background-color', 'blue');
    this.backgroundColor = 'blue';
  }
    
  @HostListener('mouseleave') mouseleave(eventData: Event){
    this.backgroundColor = 'transparent';
  }
}
````

