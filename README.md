# Using Firebase Triggerable Cloud Functions with Angular and AngularFire

This tutorial will make a simple Angular app that triggers Firebase Cloud Functions by writing to Firestore.

This project uses Angular 14, AngularFire 7, and Firestore Web version 9 (modular).

I assume that you know the basics of Angular (nothing advanced is required). No CSS or other styling is used, to make the code easier to understand.

### Triggerable vs. callable Cloud Functions

Firebase Cloud Functions can be executed in three ways:

* Call functions from your app, e.g., Angular or React
* Call functions via HTTP request, which is good for Express apps
* Trigger from a write to Firestore (other Firebase products such as Storage are in beta)

I wrote a tutorial for [calling Cloud Functions from an Angular App](https://github.com/tdkehoe/Firebase-Cloud-Functions-with-Angular). Calling functions via HTTP requests is for Express apps, not Angular apps. This tutorial is about triggering Cloud Functions from Firestore. This tutorial is seperate from the callable functions tutorial because the latter uses AngularFire 6 when this tutorial uses AngularFire 7.

## Create a new project

In your terminal:

```bash
npm install -g @angular/cli
ng new TriggerableFunctionsTutorial
```

The Angular CLI's `new` command will set up the latest Angular build in a new project structure. Accept the defaults (no routing, CSS). Start the server:

```bash
cd TriggerableFunctionsTutorial
ng serve -o
```

Your browser should open to `localhost:4200`. You should see the Angular default homepage.

## Install AngularFire and Firebase

Open another tab in your terminal and install AngularFire and Firebase from `npm`.

```bash
npm install firebase
ng add @angular/fire
```

Deselect `ng deploy -- hosting` and select `Firestore`. 

It will ask you for your email address associated with your Firebase account. Then it will ask you to associate a Firebase project. Select `[CREATE NEW PROJECT]`. Call it `triggerable-cloud-functions`. (Must be 6-30 characters, no spaces, all lower case. These naming requirements aren't the same as when you create a new project in the Firebase Console.)

You'll also be asked to create a new app. Call it `triggerable-functions-app`.

You should see:

```bash
UPDATE .gitignore (602 bytes)
UPDATE src/app/app.module.ts (627 bytes)
UPDATE src/environments/environment.ts (998 bytes)
UPDATE src/environments/environment.prod.ts (391 bytes)
```

If you look in `app.module.ts` you'll see some new imported modules. In `environment.ts` you'll see your new Firebase credentials.

## Create Firestore database
Open your Firebase console and open your project. Click on `Cloud Firestore`, `Create database`, and `Start in test mode`.

## Install Firebase
Open the official documentation for (Get started: write, test, and deploy your first functions)[https://firebase.google.com/docs/functions/get-started].

Install `firebase-tools` globally:

```bash
npm install -g firebase-tools
```

Install `firebase-admin` locally in your project directory.

```bash
npm install firebase-admin@latest
```

Run `firebase login` to log in via the browser and authenticate the firebase tool:

```bash
firebase login
```

## Initialize Firestore
Initialize the Firestore database:

```bash
firebase init firestore
```

Accept the default settings.

You may need to add your project:

```bash
firebase use --add triggerable-cloud-functions
```

## Install and initialize Functions

Install `firebase-functions` locally in your project directory.

```bash
npm install firebase-functions@latest
```

Initialize functions:

```bash
firebase init firestore
```

You'll be asked to select `JavaScript` or `TypeScript`. The TypeScript transpiler throws endless errors when I try to deploy cloud functions. I'll tell you how to fix some of these errors but you can avoid headaches by selecting JavaScript.

### TypeScript

If you chose TypeScript, don't use ESLint. This will cancel deployment because of endless style issues. I use Visual Studio Code to catch syntax errors.

Install the dependencies.

You should now have a subdirectory `functions`. This subdirectory has its own `package.json` and `tsconfig.json`.

Open `functions/package.json` and change:

```js
"main": "lib/index.js",
```

to

```js
"main": "src/index.ts",
```

In `tsconfig.json` add the properties `moduleResolution` and `noImplicitAny`:

```js
{
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "Node",
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "outDir": "lib",
    "sourceMap": true,
    "strict": true,
    "noImplicitAny": false,
    "target": "es2017"
  },
  "compileOnSave": true,
  "include": [
    "src"
  ]
}
```

## Initialize emulator

Firebase comes with an emulator. The emulator will simulate many Firebase services, including Firestore, Auth, etc. I've never found a need for the emulator with Firestore and Auth as these execute quickly in the cloud. 

Functions are different. Without the emulator developing code can be painfully slow. Deploying your code changes to the cloud takes about two minutes. Then I test my code changes and I have to wait for the console logs. This takes a few more minutes, with clicking various buttons in the console to get the logs to stream. I've seen a lag time between deploying functions to the cloud and the new version running so this can add a minute or two. All in all, waiting five minutes between writing new code and seeing the results feels painfully slow. With the emulator there's no waiting.

Another advantage of the emulator is that you screw up your code, such as writing an infinite loop, without affecting your Google Cloud Services bill. In other words, test your functions in the emulator before deploying them to the cloud.

Initiate the emulators:

```bash
firebase init emulators
npm run build
```

The latter command might ask you to update Java on your computer. 

In `src/environments.ts` add a property:

```ts
export const environment = {
  firebase: {
    projectId: '...,
    appId: '...,
    storageBucket: '...',
    locationId: 'us-central',
    apiKey: '...',
    authDomain: '...',
    messagingSenderId: '...',
  },
  production: false,
  useEmulators: true
};
```

Change `useEmulators` to `false` when you deploy to the cloud.

## Directory structure
Look at your directory and you should see, if you chose TypeScript:

```bash
myproject
 +- .firebaserc    # Hidden file that helps you quickly switch between
 |                 # projects with `firebase use`
 |
 +- firebase.json  # Describes properties for your project
 |
 +- functions/     # Directory containing all your functions code
      |
      +- node_modules/ # directory where your dependencies (declared in # package.json) are installed
      |
      +- package-lock.json
      |
      +- src/
          |
           +- index.ts  # main source file for your Cloud Functions code
      |
      +- tsconfig.json  # if you chose TypeScript
      |
      +- package.json  # npm package file describing your Cloud Functions code
```

JavaScript will be simpler:

```bash
myproject
 +- .firebaserc    # Hidden file that helps you quickly switch between
 |                 # projects with `firebase use`
 |
 +- firebase.json  # Describes properties for your project
 |
 +- functions/     # Directory containing all your functions code
      |
      +- node_modules/ # directory where your dependencies (declared in # package.json) are installed
      |
      +- package-lock.json
      |
      +- index.js  # main source file for your Cloud Functions code
      |
      +- package.json  # npm package file describing your Cloud Functions code
```

## Setup `@NgModule` for the `AngularFireModule` and `AngularFireFunctionsModule`

Open the [AngularFire documentation](https://github.com/angular/angularfire/blob/master/docs/functions/functions.md) for this section.

Now we can start writing Angular. Open `/src/app/app.module.ts` and import modules.

### AngularFire 6 vs 7

Here we hit a stumbling block. The official AngularFire documentation shows how to use callable functions with AngularFire 6. AngularFire 7 seems to be available for functions but there's no documentation. I asked on Stack Overflow and no one answered my question.

The problem is that you can't mix AngularFire 6 and 7. I use AngularFire 7 for Firestore and Auth. This means that I can't use callable functions with Firestore or Auth. The workaround is to trigger background functions instead of calling functions directly. We'll get to triggered background functions later. First this tutorial will teach callable functions, which you can't really use in a real Angular app now. I believe that we're very close to using AngularFire 7 with callable apps and only a few lines of code will need to change so let's learn to use callable functions with Angular 6.

```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';
import { environment } from '../environments/environment';

// AngularFire 7 -- comment out these line, we won't be using AngularFire 7
// import { initializeApp, provideFirebaseApp } from '@angular/fire/app';
// import { provideFirestore, getFirestore } from '@angular/fire/firestore';
// import { provideFunctions, getFunctions, connectFunctionsEmulator } from '@angular/fire/functions';

// AngularFire 6
import { AngularFireModule } from '@angular/fire/compat';
import { AngularFireFunctionsModule } from '@angular/fire/compat/functions';
import { USE_EMULATOR as USE_EMULATOR_FUNCTIONS } from '@angular/fire/compat/functions'; // comment out to run in the cloud

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,

    // AngularFire 7 -- comment out these line, we won't be using AngularFire 7
    // provideFirebaseApp(() => initializeApp(environment.firebase)),
    // provideFirestore(() => getFirestore()),
    // provideFunctions(() => getFunctions()),

    // AngularFire 6
    AngularFireModule.initializeApp(environment.firebase),
    AngularFireFunctionsModule
  ],
  providers: [
        { provide: USE_EMULATOR_FUNCTIONS, useValue: environment.useEmulators ? ['localhost', 5001] : undefined }
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

You can comment out the Angular 7 lines now. 

## Make the HTML view

Open `app.component.html`. Replace the placeholder view with:

```html
<div>
    <button mat-raised-button color="basic" (click)='callMe()'>Call me!</button>
</div>

{{ data$ | async }}
```

We made a button that calls a handler function in theb controller. There's also a line to display data returned from the callable function.

## Make the component controller

In `app.component.ts` 

```ts
import { Component } from '@angular/core';

// AngularFire 7
// import { getApp } from '@angular/fire/app';
// import { provideFunctions, getFunctions, connectFunctionsEmulator, httpsCallable } from '@angular/fire/functions';
// import { Firestore, doc, getDoc, getDocs, collection, updateDoc } from '@angular/fire/firestore';

// AngularFire 6
import { AngularFireFunctions } from '@angular/fire/compat/functions';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
})
export class AppComponent {
  data$: any;

  constructor(private functions: AngularFireFunctions) {
    const callable = this.functions.httpsCallable('executeOnPageLoad');
    this.data$ = callable({ name: 'Charles Babbage' });
  }

  callMe() {
    console.log("Calling...");
    const callable = this.functions.httpsCallable('callMe');
    this.data$ = callable({ name: 'Ada Lovelace' });
  };
}
```

Again, comment out the Angular 7 imports for now.

In `AppComponent` we make a single variable `data$: any`. This will handle the data returned from the callable functions and display it in the view.

In the `constructor` we make a local variable `functions` to alias `AngularFireFunctions`. This is AngularFire 6. `AngularFireFunctions` isn't on AngularFire 7. As soon as `AngularFireFunctions` (or an equivelant property) is available on AngularFire 7 we should be able to use AngularFire 7.

We have two lines of code that call a cloud function to execute on page load. These use the property `httpsCallable`. This take one parameter, the name of your cloud function.

To execute the callable function we use `this.data$` to handle the returned data and then call the function. Calling the function has a required parameter, which is an object holding the data you want to send to the cloud function.

Lastly, the component controller has a handler function for the button in the view. This executes similar code.

## Write your Firebase Cloud Functions

Open `functions/src/index.ts` or `functions/index.js`. Import two Firebase modules, initialize your app, and then write your callable functions.

```ts
// The Cloud Functions for Firebase SDK to create Cloud Functions and set up triggers.
const functions = require('firebase-functions');

// The Firebase Admin SDK to access Firestore.
const admin = require('firebase-admin');
admin.initializeApp();

// executes on page load
exports.executeOnPageLoad = functions.https.onCall((data, context) => {
    console.log("The page is loaded!")
    console.log(data);
    console.log(data.name);
    // console.log(context);
    return 22
});

// executes on user input
exports.callMe = functions.https.onCall((data, context) => {
    console.log("Thanks for calling!")
    console.log(data);
    console.log(data.name);
    // console.log(context);
    return 57
});
```

The function `executeOnPageLoad` executes when `ng serve` starts or restarts.

The functions `callMe` executes when you click the button in the view.

Each function is in the form 

```js
`functions.https.onCall((data, context) => {

});
```

`https.onCall` means that this functions can be called directly from Angular. `data` is the data sent from Angular. `context` is metadata about the function's execution. We won't be using this.

Each function sends a message to the console, then sends the data from Angular to the console. We've commented out displaying the `context` metadata in the console. This metadata goes on for pages and makes the logs hard to read. 

Finally, each callable function returns something. 

## Run emulator

Start the Firebase Emulator.

```bash
firebase emulators:start --only functions
```

Run the function `executeonPageLoad` by restarting `ng serve`. Run the function `callMe` by clicking the button in the view.

You should see the results in several places. In the view, you should see `22` as the result from `executeonPageLoad`. When you click the button this result changes to `57`. This is an observable so if the data changes in the cloud function it'll change in the view.

In the emulator logs, you should see the logs:

```
12:05:02  I function[us-central1-callMe]  Beginning execution of "callMe"
12:05:02  I function[us-central1-callMe]  Thanks for calling!
12:05:02  I function[us-central1-callMe]  { name: 'Ada Lovelace' }
12:05:02  I function[us-central1-callMe]  Ada Lovelace
12:05:02  I function[us-central1-callMe]  Finished "callMe" in 6.512202ms
```

You should see the same logs in the terminal emulator tab.

## TypeScript errors

If you chose to use TypeScript you may see some errors. The issue is the data type of the parameters in your functions:

```ts
exports.callMe = functions.https.onCall((data, context) => {}
```

This will throw any error `Parameter 'data' implicitly has an 'any' type.` This is because TypeScript is running in `strict` mode by default.

```ts
exports.callMe = functions.https.onCall((data: any, context: any) => {}
```

This will throw this error:

```
SyntaxError: Unexpected token ':'
```

The latter appears to be an issue with the transpiler. The transpiler apparently failed to remove the types when it transpiled TypeScript into JavaScript.

Both errors can be ignored. You can get rid of the former error by opening `functions/tsconfig.json` and either change `strict` to `false` or to add this line:

```js
"noImplicitAny": false,
```

It doesn't seem to matter if you set this to `true` or `false`.

## Deploy to Firebase


```bash
 firebase deploy --only functions
 ```
 
 In `src/environments.ts` change `useEmulators` to `false`:
 
 ```js
 useEmulators: false
 ```
 
Check the logs in your Firebase Console to see if your functions run.
 
## Calling functions via HTTP requests

Firebase cloud functions can also be [called via HTTP requests](https://firebase.google.com/docs/functions/http-events?hl=en&authuser=0). This is useful for Express apps but not for Angular apps.

## Triggering Firebase Cloud Functions from FireStore

You can trigger a Firebase Cloud Function by writing data to Firestore. This doesn't use AngularFire for functions, i.e., only uses AngularFire for Firestore, so it doesn't matter whether you use AngularFire 6 or 7 for functions.

Make a new cloud function:

```ts
exports.makeUppercase = functions.firestore.document('/triggers/upperCASE')
.onCreate((snap, context) => {
  const original = snap.data().original;
  console.log('Uppercasing', context.params.documentId, original);
  const uppercase = original.toUpperCase();
  return snap.ref.set({uppercase}, {merge: true});
});
```

This will trigger you write data to the document `upperCASE` in the collection `triggers`. The function receives a string and returns the string in UPPERCASE.

Add a form field to the HTML view:

```html
<div>
    <button mat-raised-button color="basic" (click)='callMe()'>Call me!</button>
</div>

{{ data$ | async }}

<form (ngSubmit)="upperCaseMe()">
    <input type="text" [(ngModel)]="message" name="message" placeholder="Message" required>
    <button type="submit" value="Submit">Submit</button>
</form>
```

In `app.module.ts` import Angular Forms:

```js
import { FormsModule } from '@angular/forms';
...
  imports: [
    BrowserModule,
    FormsModule,
    ...
    ]
```

Add the handler function to `app.component.ts`

```ts
  async upperCaseMe() {
    console.log(this.message);
      try {
        const docRef = await addDoc(collection(this.firestore, 'triggers'), {
          message: this.message,
        });
        console.log(docRef);
        this.message = null;
      } catch (error) {
        console.error(error);
      }
  }
```

Deploy to Firebase:

```bash
firebase deploy --only functions
```

### Switch from AngularFire 6 to 7

Functions and the emulator run in AngularFire 6. Firestore runs in AngularFire 7. You can't mix AngularFire 6 and 7. Comment out the AngularFire 6 code and comment in the AngularFire 7 code. You don't AngularFire Functions to trigger cloud functions (only for callable functions). 
