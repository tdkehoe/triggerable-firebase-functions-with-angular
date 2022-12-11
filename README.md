# Using Firebase Cloud Functions with Angular

This tutorial will make a simple Angular app that calls Firebase Cloud Functions directly and triggers Firebase Cloud Functions by writing to Firestore.

This project uses Angular 15, AngularFire 7, and Firestore Web version 9 (modular).

I assume that you know the basics of Angular (nothing advanced is required). No CSS or other styling is used, to make the code easier to understand.

## Callable vs Triggerable Cloud Functions

Firebase Cloud Functions can be executed in three ways:

* Call functions from your app, e.g., Angular or React
* Call functions via HTTP request, which is good for Express apps
* Trigger from a write to Firestore (other Firebase products such as Storage are in beta)

### What about AngularFire?
AngularFire 6 handles Callable Cloud Functions. AngularFire 7 doesn't and you don't need it. Don't try to use the AngularFire 6. It uses Firebase 8, not Firebase 9, and it only works with TypeScript 4.7.2. I wrote a tutorial for [AngularFire 6 with Cloud Functions](https://github.com/tdkehoe/Firebase-Cloud-Functions-with-Angular). 

## Make Angular app

In your parent directory, spin up a new Angular app and call it `CloudFunctions`

```
npm install -g @angular/cli
ng new CloudFunctions
```

Accept the defaults (no routing, CSS).

## Install Firebase

Open the official documentation for (Get started: write, test, and deploy your first functions)[https://firebase.google.com/docs/functions/get-started].

Change directories into your Angular project directory. Install AngularFire and Firebase from `npm`.

```bash
cd CloudFunctions
npm install -g firebase-tools
firebase init
```

Select `Emulators`.

Select `Create a new project`.

Select `Functions Emulator` and `Firestore Emulator`.

Run `firebase login` to log in via the browser and authenticate the firebase tool:

```
firebase login
```

## Emulators Setup

Select `Functions Emulator` and `Firestore Emulator`. Accept the default ports, but download the emulators.

## Functions Setup: TypeScript or JavaScript? CommonJS or ES Modules?

I initialize my Firebase Functions directories for TypseScript. However, I can't get Firebase Functions to work with TypeScript so I use JavaScript. I presume that a future version will work with TypeScript.

### What goes wrong with TypeScript?

With `index.ts` the emulator throws this error:

```
SyntaxError: Cannot use import statement outside a module
```

This means that the compiler doesn't recognize that your `index.ts` file is an ES module. Supposedly this can be fixed by adding `"type": "module",` to your `functions/package.json` but that doesn't fix the error for me.

There are two types of modules (including npm packages). The older CommonJS modules use the keywords `require` and `exports`. Newer ES modules use the keywords `import` and `export`. ES modules can import CommonJS modules; CommonJS modules can't import ES modules.

But maybe TypeScript Cloud Functions will work in a future Firebase update. Select `TypeScript` and we'll switch to using JavaScript below. We'll be able to switch back to TypeScript easily.

### ESLint, installing dependencies

Don't use ESLint. Visual Studio Code will lint your code as you type. ESLint will lint your code at compile time, i.e., after you've made lots of typos.

Install dependencies.

## Start emulators

Start the Firestore and Functions Emulators.

```bash
firebase emulators:start
```

You should see this, without errors:

```bash
┌─────────────────────────────────────────────────────────────┐
│ ✔  All emulators ready! It is now safe to connect your app. │
│ i  View Emulator UI at http://127.0.0.1:4000/               │
└─────────────────────────────────────────────────────────────┘

┌───────────┬────────────────┬─────────────────────────────────┐
│ Emulator  │ Host:Port      │ View in Emulator UI             │
├───────────┼────────────────┼─────────────────────────────────┤
│ Functions │ 127.0.0.1:5001 │ http://127.0.0.1:4000/functions │
├───────────┼────────────────┼─────────────────────────────────┤
│ Firestore │ 127.0.0.1:8080 │ http://127.0.0.1:4000/firestore │
└───────────┴────────────────┴─────────────────────────────────┘
  Emulator Hub running at 127.0.0.1:4400
  Other reserved ports: 4500, 9150

Issues? Report them at https://github.com/firebase/firebase-tools/issues and attach the *-debug.log files.
```

If you get an error `SyntaxError: Cannot use import statement outside a module` ignore this, we'll fix it below.

### Directory structure
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

## Emulator or cloud?

Notice what we didn't do: we didn't connect to a Firestore database in the Firebase cloud. We'll do that later. First we'll develop our Cloud Functions using the Firebase Emulators, specifically the Firestore and Functions Emulators. (We won't use the other emulators such as Auth.) We use emulators for code development for a few reasons:

Emulators are fast. The cycle of making a change in your Cloud Function, testing the Cloud Function, and viewing the results is about ten minutes in the cloud. It's about one minute in the emulators. Developing your code will be up to 10x faster.

Emulators cost nothing. If you make an infinite loop in a Cloud Function in the Firebase cloud, you'll pay for the computer time. In the emulator there's no cost.

## Configure `package.json`

Your `functions` directory has its own `package.json`, and the default configuration is incorrect. Open `functions/package.json`.

Add `"type": "module",` at the highest level. This enables you to use ES module npm packages, in theory at least.

Change `"main": "lib/index.js",` to `"main": "src/index.js",`. If you want to use TypeScript change this to `"main": "src/index.ts",`. I don't recommend using TypeScript in Cloud Functions.

In your terminal run

```bash
npm install firebase-admin@latest
npm install firebase-functions@latest
npm install typescript@latest
```

Notice that the version numbers change in your `package.json`.

Your `package.json` should now look like this (perhaps with newer version numbers):

```js
{
    "name": "functions",
    "type": "module",
    "scripts": {
        "build": "tsc",
        "build:watch": "tsc --watch",
        "serve": "npm run build && firebase emulators:start --only functions",
        "shell": "npm run build && firebase functions:shell",
        "start": "npm run shell",
        "deploy": "firebase deploy --only functions",
        "logs": "firebase functions:log"
    },
    "engines": {
        "node": "16"
    },
    "main": "src/index.js",
    "dependencies": {
        "firebase-admin": "^11.2.0",
        "firebase-functions": "^4.0.2"
    },
    "devDependencies": {
        "typescript": "^4.9.4"
    },
    "private": true
}
```

### Optional: configure `tsconfig.json`

We won't be using TypeScript and so we won't use `tsconfig.json`. But if you want to get ready for using TypeScript in the future you can change `"module": "CommonJS",` to `"module": "ESNext",` add `"moduleResolution": "node",` and change `"target": "es2017"` to `"target": "ESNext"`. `ESNext` means "the latest version of JavaScript`.

`tsconfig.json` now looks like this:

```js
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "node",
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "outDir": "src",
    "sourceMap": true,
    "strict": true,
    "target": "ESNext"
  },
  "compileOnSave": true,
  "include": [
    "src/index.ts"
  ],
  "exclude": ["wwwroot"],
}
```

# Write a Triggerable Firebase Cloud Function

If you chose TypeScript, rename `functions/src/index.ts` to `functions/src/index.js`. This will run your Cloud Functions as JavaScript.

Your emulators should now run without the error `SyntaxError: Cannot use import statement outside a module`. Your Cloud Functions are now in an ES module.

Open `functions/src/index.js` or `functions/index.js`. Import the `firebase-functions` module, then write a Cloud Function.

```ts
import * as functions from "firebase-functions";

export const makeUppercase = functions.firestore.document('messages/{docId}').onCreate((snap, context) => {
    const original = snap.data().original;
    functions.logger.log('Uppercasing', context.params.docId, original);
    const uppercase = original.toUpperCase();
    return snap.ref.set({ uppercase }, { merge: true });
});
```

We import `firebase-functions`. It's a CommonJS module but `import` handles both CommonJS and ES modules, without changing syntax.  This won't work:

```
import { getFunctions } from 'firebase/functions';
```

You can't import individual methods as needed from the `firebase/functions` module because it's a CommonJS module, not an ES module. Most Firebase modules are ES modules, allowing you to import only the functions you need.

The Cloud Function `makeUppercase` has six lines.

The first line sets up a trigger from the collection `messages` and a wildcard for the `docId`. The trigger is on `onCreate()`, i.e., when a new document is created. The parameters are `data`, which is what was written to Firestore, and `context`, which is metadata including the `docId`.

The second line stores the message written to Firestore in a local variable `original`.

The third line is the same as `console.log`.

The fourth line converts the original message to UPPERCASE.

The fifth line returns something. All Cloud Functions must return something. This line uses `snap.ref` as the Firestore address of the document. It uses `set` with the `merge` parameter to add a field to an existing document. The new field is named `uppercase`. 

## Run your Triggerable Cloud Function

In your browser, open a tab to http://127.0.0.1:4000/functions. You should see the `Firebase Emulator Suite`.

Click on `Firestore`. Click `Start a collection`. Enter `messages` as the name of the collection. The `docId` will be automatically entered. In `Field` enter `original` and in `Value` enter `hello world`. Click `SAVE` and in few moments you should see a new field appear: `uppercase: HELLO WORLD`. Yay, your first Cloud Function is running!

Click on the `Logs` tab. You should see something like this:

```
13:32:00 I function[us-central1-makeUppercase] Beginning execution of "makeUppercase"
13:32:00 I function[us-central1-makeUppercase] {
                                                "severity": "INFO",
                                                "message": "Uppercasing Ay6qtx2DD8CaCniEME2I hello world"
                                               }
13:32:01 I function[us-central1-makeUppercase] Finished "makeUppercase" in 501.081045ms
```

## Connect Your Triggerable Firebase Cloud Function to Angular

Put a form and `Submit` button in your HTML view.

*app.component.html*
```
<h3>Trigger Firebase Cloud Function</h3>
<form (ngSubmit)="triggerMe(triggerText)">
    <input type="text" [(ngModel)]="triggerText" name="message" placeholder="message" required>
    <button type="submit" value="Submit">Submit</button>
</form>
```

Add the handler function in the controller.

*app.component.ts*
```

```


# Write a Callable Firebase Cloud Function

Let's make a callable cloud function.

*index.js*
```js
export const addMessage = functions.https.onCall((data, context) => {
  try {
    const original = data.text;
    const uppercase = original.toUpperCase();
    return uppercase;
  } catch (error) {
    console.error(error);
  }
});
```

In app.component.html make a form.

```html
<h3>Call cloud function</h3>
<form (ngSubmit)="callMe(messageText)">
    <input type="text" [(ngModel)]="messageText" name="message" placeholder="message" required>
    <button type="submit" value="Submit">Submit</button>
</form>
```

In app.component.ts set up config and initialization boilerplate, then write the handler function.

```js
import { Component } from '@angular/core';
import { getFunctions, httpsCallable, httpsCallableFromURL } from "firebase/functions";
import { initializeApp } from 'firebase/app';
import { getFirestore, setDoc, doc } from "firebase/firestore";
import { environment } from '../environments/environment';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  firebaseConfig = environment.firebaseConfig;
  firebaseApp = initializeApp(this.firebaseConfig);
  db = getFirestore(this.firebaseApp);

  messageText: string | null = null;
  functions = getFunctions(this.firebaseApp);

  callMe(messageText: string | null) {
    console.log("Calling Cloud Function: " + messageText);
    // const addMessage = httpsCallable(this.functions, 'addMessage'); // throws CORS error
    const addMessage = httpsCallableFromURL(this.functions, 'http://localhost:5001/triggerable-functions-project/us-central1/addMessage');
    addMessage({ text: messageText })
      .then((result) => {
        console.log(result.data)
      });
  };
}
```

The documentation says to use `httpsCallable`. You can try this, I get a CORS error. Instead I use `httpsCallableFromURL`. The function then only accepts calls from the specified URL. The URL has the location of the Angular server, the project ID, the location of the Firebase server, and the name of the function.

## Deploy to the Firebase cloud

In your Firebase console, add a new project. Give your project a Firestore database. Upgrade your project to the `Blaze` (paid) plan. (Cloud Functions aren't free.)

Deploy your Cloud Function to Firebase:

```bash
firebase deploy --only functions:uppercaseMe
 ```

You'll be prompted to select your project.

In your Cloud Firestore database, make a directory `messages` and a document. You should see the new field `uppercase` appear. You can read your cloud logs by clicking on `Functions` in your Firebase console.

## Open Angular app

Open your app.

```bash
ng serve -o
```

You should see the Angular default page.

## Add AngularFire

Add AngularFire to your project.

```bash
ng add @angular/fire
```

Open the [AngularFire documentation](https://github.com/angular/angularfire/blob/master/docs/functions/functions.md) for this section.

## `app.module.ts`

Add `Forms` to `app.module.ts`:

```ts
import { FormsModule } from '@angular/forms';
...
 imports: [
    FormsModule,
    ...
    ]
```

Your `app.module.ts` should look like this:

```js
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';
import { environment } from '../environments/environment';

import { initializeApp, provideFirebaseApp } from '@angular/fire/app';
import { provideFirestore, getFirestore } from '@angular/fire/firestore';
import { provideFunctions, getFunctions } from '@angular/fire/functions';

import { FormsModule } from '@angular/forms';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    provideFirebaseApp(() => initializeApp(environment.firebase)),
    provideFirestore(() => getFirestore()),
    provideFunctions(() => getFunctions()),
    ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

## Optional: `useEmulators`

 In `src/environments.ts` add a property `useEmulators`:
 
 ```js
 useEmulators: false
 ```
 
 Your `environments.ts` should look like:
 
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

## Make the HTML view

Open `app.component.html`. Replace the placeholder view with:

```html
<form (ngSubmit)="triggerMe()">
    <input type="text" [(ngModel)]="message" name="message" placeholder="Message" required>
    <button type="submit" value="Submit">Submit</button>
</form>

<div>{{ data$ }}</div>
<div>{{ upperca$e }}</div>
```

We made a form to send a message to be stored in Firestore. `data$` displays what we read from Firestore after the write. `upperca$e` displays what we read from Firestore after the Cloud Function run. (`$` at the end of a variable flags that it's an Observable.)

## Make the component controller

In `app.component.ts` 

```ts
import { Component } from '@angular/core';
import { Firestore, doc, getDoc, collection, addDoc, onSnapshot } from '@angular/fire/firestore';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  data$: any;
  docSnap: any;
  message: string | null = null;
  upperca$e: string | null = null;
  unsubMessage$: any;

  constructor(public firestore: Firestore) {}

  async triggerMe() {
    try {
      // write to Firestore
      const docRef = await addDoc(collection(this.firestore, 'messages'), {
        message: this.message,
      });
      this.message = null; // clear form fields
      // read from Firestore
      this.docSnap = await getDoc(doc(this.firestore, 'messages', docRef.id));
      this.data$ = this.docSnap.data().message;
      // document listener
      this.unsubMessage$ = onSnapshot(doc(this.firestore, 'messages', docRef.id), (snapshot: any) => {
        this.upperca$e = snapshot.data().uppercase;
      });
    } catch (error) {
      console.error(error);
    }
  }
}
```

The handler function writes a message to Firestore, reads back what Firestore stored, then listens for changes in the document.

We could unsubscribe the listener with

```ts
this.unsubMessage$();
```

## Run your code

Open your browser to `http://localhost:4200/` and you should see a form. Enter a message and click the `Submit` button. You should see your message appear below the form.

In your Firestore database you should see your message.

 

