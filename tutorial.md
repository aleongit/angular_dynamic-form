
## Enable reactive forms for your project

- Dynamic forms are based on reactive forms. To give the application access reactive forms directives, the `root module` imports `ReactiveFormsModule` from the `@angular/forms library`.

- The following code from the example shows the setup in the root module.

- **main.ts**
```ts
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

import { AppModule } from './app/app.module';

platformBrowserDynamic().bootstrapModule(AppModule)
  .catch(err => console.error(err));
```

- **app.module.ts**
```ts
import { BrowserModule } from '@angular/platform-browser';
import { ReactiveFormsModule } from '@angular/forms';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';
import { DynamicFormComponent } from './dynamic-form.component';
import { DynamicFormQuestionComponent } from './dynamic-form-question.component';

@NgModule({
  imports: [ BrowserModule, ReactiveFormsModule ],
  declarations: [ AppComponent, DynamicFormComponent, DynamicFormQuestionComponent ],
  bootstrap: [ AppComponent ]
})
export class AppModule {}
```



## Create a form object model

- A dynamic form requires an object model that can describe all scenarios needed by the form functionality. The example hero-application form is a set of questions —that is, each control in the form must ask a question and accept an answer.

- The data model for this type of form must represent a question. The example includes the `DynamicFormQuestionComponent`, which defines a question as the fundamental object in the model.

- The following `QuestionBase` is a base class for a set of controls that can represent the question and its answer in the form.

- **src/app/question-base.ts**
```ts
export class QuestionBase<T> {
  value: T|undefined;
  key: string;
  label: string;
  required: boolean;
  order: number;
  controlType: string;
  type: string;
  options: {key: string, value: string}[];

  constructor(options: {
      value?: T;
      key?: string;
      label?: string;
      required?: boolean;
      order?: number;
      controlType?: string;
      type?: string;
      options?: {key: string, value: string}[];
    } = {}) {
    this.value = options.value;
    this.key = options.key || '';
    this.label = options.label || '';
    this.required = !!options.required;
    this.order = options.order === undefined ? 1 : options.order;
    this.controlType = options.controlType || '';
    this.type = options.type || '';
    this.options = options.options || [];
  }
}
```


### Define control classes

- From this base, the example derives two new classes, `TextboxQuestion` and `DropdownQuestion`, that represent different control types. When you create the form template in the next step, you instantiate these specific question types in order to render the appropriate controls dynamically.



#### *TextboxQuestion* control type

- Presents a question and lets users enter input.

- **src/app/question-textbox.ts**
```ts
import { QuestionBase } from './question-base';

export class TextboxQuestion extends QuestionBase<string> {
  override controlType = 'textbox';
}
```

- The `TextboxQuestion` control type is represented in a form template using an `<input>` element. The `type` attribute of the element is defined based on the `type` field specified in the `options` argument (for example `text`, `email`, `url`).




#### *DropdownQuestion* control type

- Presents a list of choices in a select box.

- **src/app/question-dropdown.ts**
```ts
import { QuestionBase } from './question-base';

export class DropdownQuestion extends QuestionBase<string> {
  override controlType = 'dropdown';
}
```



### Compose form groups

- A dynamic form uses a service to create grouped sets of input controls, based on the form model. The following `QuestionControlService` collects a set of `FormGroup` instances that consume the metadata from the question model. You can specify default values and validation rules.

- **src/app/question-control.service.ts**
```ts
import { Injectable } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

import { QuestionBase } from './question-base';

@Injectable()
export class QuestionControlService {
  toFormGroup(questions: QuestionBase<string>[] ) {
    const group: any = {};

    questions.forEach(question => {
      group[question.key] = question.required ? new FormControl(question.value || '', Validators.required)
                                              : new FormControl(question.value || '');
    });
    return new FormGroup(group);
  }
}
```



## Compose dynamic form contents

- The dynamic form itself is represented by a container component, which you add in a later step. Each question is represented in the form component's template by an `<app-question>` tag, which matches an instance of `DynamicFormQuestionComponent`.

- The `DynamicFormQuestionComponent` is responsible for rendering the details of an individual question based on values in the data-bound question object. The form relies on a `[formGroup]` directive to connect the template HTML to the underlying control objects. The `DynamicFormQuestionComponent` creates form groups and populates them with controls defined in the question model, specifying display and validation rules.

- **dynamic-form-question.component.ts**
```ts
import { Component, Input } from '@angular/core';
import { FormGroup } from '@angular/forms';

import { QuestionBase } from './question-base';

@Component({
  selector: 'app-question',
  templateUrl: './dynamic-form-question.component.html'
})
export class DynamicFormQuestionComponent {
  @Input() question!: QuestionBase<string>;
  @Input() form!: FormGroup;
  get isValid() { return this.form.controls[this.question.key].valid; }
}
```

- **dynamic-form-question.component.html**
```html
<div [formGroup]="form">
  <label [attr.for]="question.key">{{question.label}}</label>

  <div [ngSwitch]="question.controlType">

    <input *ngSwitchCase="'textbox'" [formControlName]="question.key"
            [id]="question.key" [type]="question.type">

    <select [id]="question.key" *ngSwitchCase="'dropdown'" [formControlName]="question.key">
      <option *ngFor="let opt of question.options" [value]="opt.key">{{opt.value}}</option>
    </select>

  </div>

  <div class="errorMessage" *ngIf="!isValid">{{question.label}} is required</div>
</div>
```

- The goal of the `DynamicFormQuestionComponent` is to present question types defined in your model. You only have two types of questions at this point but you can imagine many more. The `ngSwitch` statement in the template determines which type of question to display. The switch uses directives with the `formControlName` and `formGroup` selectors. Both directives are defined in `ReactiveFormsModule`.


### Supply data

- Another service is needed to supply a specific set of questions from which to build an individual form. For this exercise you create the `QuestionService` to supply this array of questions from the hard-coded sample data. In a real-world app, the service might fetch data from a backend system. The key point, however, is that you control the hero job-application questions entirely through the objects returned from `QuestionService`. To maintain the questionnaire as requirements change, you only need to add, update, and remove objects from the `questions` array.

- The `QuestionService` supplies a set of questions in the form of an array bound to `@Input()` questions.

- **src/app/question.service.ts**
```ts
import { Injectable } from '@angular/core';

import { DropdownQuestion } from './question-dropdown';
import { QuestionBase } from './question-base';
import { TextboxQuestion } from './question-textbox';
import { of } from 'rxjs';

@Injectable()
export class QuestionService {

  // TODO: get from a remote source of question metadata
  getQuestions() {

    const questions: QuestionBase<string>[] = [

      new DropdownQuestion({
        key: 'brave',
        label: 'Bravery Rating',
        options: [
          {key: 'solid',  value: 'Solid'},
          {key: 'great',  value: 'Great'},
          {key: 'good',   value: 'Good'},
          {key: 'unproven', value: 'Unproven'}
        ],
        order: 3
      }),

      new TextboxQuestion({
        key: 'firstName',
        label: 'First name',
        value: 'Bombasto',
        required: true,
        order: 1
      }),

      new TextboxQuestion({
        key: 'emailAddress',
        label: 'Email',
        type: 'email',
        order: 2
      })
    ];

    return of(questions.sort((a, b) => a.order - b.order));
  }
}
```


## Create a dynamic form template