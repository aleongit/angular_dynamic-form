
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