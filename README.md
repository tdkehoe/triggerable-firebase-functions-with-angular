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

Open another tab in your terminal, check that you're in `TriggerableFunctionsTutorial`, and install AngularFire and Firebase from `npm`.

```bash
npm install firebase
ng add @angular/fire
```

Select `ng deploy -- hosting`, `Firestore`, and `Cloud Functions (callable)`. (We won't use Hosting or callable Cloud Functions in this tutorial.)

It will ask you for your email address associated with your Firebase account. Then it will ask you to associate a Firebase project. Select `[CREATE NEW PROJECT]`. It will then ask for a unique project ID. This must be 6-30 characters, no spaces, all lower case. Call it `triggerable-functions-project`. 

Then you'll be asked for project name. This can have spaces and upper case. Call it `Triggerable Functions Project`.

You'll also be asked to create a new app. Call it `triggerable-functions-app`.

You should see:

```bash
UPDATE .gitignore (602 bytes)
UPDATE src/app/app.module.ts (627 bytes)
UPDATE src/environments/environment.ts (998 bytes)
UPDATE src/environments/environment.prod.ts (391 bytes)
```

If you look in `app.module.ts` you'll see some new imported modules. In `environment.ts` you'll see your new Firebase credentials.

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
firebase use --add triggerable-functions-project
```

## Install and initialize Functions

Install `firebase-functions` locally in your project directory.

```bash
npm install firebase-functions@latest
```

Initialize functions:

```bash
firebase init functions
```

You'll be asked to select `JavaScript` or `TypeScript`. The TypeScript transpiler throws endless errors when I try to deploy cloud functions. I'll tell you how to fix some of these errors but you can avoid headaches by selecting JavaScript.

Don't use ESLint. This will cancel deployment because of endless style issues. I use Visual Studio Code to catch syntax errors.

Install the dependencies.

You should now have a subdirectory `functions`. This subdirectory has its own `package.json`. 

## Update dependencies

In `functions/package.json` 

```bash
cd functions
```

update the dependencies.

```bash
npm install --save firebase-admin@latest
npm install --save firebase-functions@latest
```

If you chose TypeScript:

```bash
npm install --save typescript@latest
```

Regularly run these commands in your `functions` directory to keep the npm modules up to date:

```bash
npm outdated
npm update
```

Look up the lastest versions:
[Firebase Admin](https://www.npmjs.com/package/firebase-admin)
[Firebase Functions](https://www.npmjs.com/package/firebase-functions)
[TypeScript](https://www.npmjs.com/package/typescript)

### TypeScript

Your `functions` subdirectory should also have its own `tsconfig.json`.

Open `functions/package.json` and add:

```js
"type": "module",
```

Try removing this if you have get errors on deployment.

Change:

```js
"main": "lib/index.js",
```

to

```js
"main": "src/index.ts",
```

Your `package.json` should now look like:

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
    "main": "src/index.ts",
    "dependencies": {
        "firebase-admin": "^11.2.0",
        "firebase-functions": "^4.0.1"
    },
    "devDependencies": {
        "typescript": "^4.8.4"
    },
    "private": true
}
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

Even with all this reconfiguration when I deploy TypeScript cloud functions I get a pair of errors. If I set types, e.g., `data: any`, the transpiler throws this error:

```bash
SyntaxError: Unexpected token ':'
```

Leaving out the type, e.g., `data`, the transpiler throws this error:

```bash
Function failed on loading user code. This is likely due to a bug in the user code.
```

## Initialize emulator

Firebase comes with an emulator. The emulator will simulate many Firebase services, including Firestore, Auth, etc. I've never found a need for the emulator with Firestore and Auth as these execute quickly in the cloud. 

Functions are different. Without the emulator developing code can be painfully slow. Deploying your code changes to the cloud takes about two minutes. Then I test my code changes and I have to wait for the console logs. This takes a few more minutes, with clicking various buttons in the console to get the logs to stream. I've seen a lag time between deploying functions to the cloud and the new version running so this can add a minute or two. All in all, waiting five minutes between writing new code and seeing the results feels painfully slow. With the emulator there's no waiting.

Another advantage of the emulator is that you screw up your code, such as writing an infinite loop, without affecting your Google Cloud Services bill. In other words, test your functions in the emulator before deploying them to the cloud.

Return to your project directory

```bash
cd ..
```

Initiate the emulators:

```bash
firebase init emulators
```

Select `Functions Emulator`, `Firestore Emulator`, and `Hosting Emulator`. (This tutorial won't use the Hosting Emulator.) Accept the default ports, but download the emulators.

Run:

```bash
npm run build
```

The latter command might ask you to update Java on your computer. 

In `src/environments.ts` add a property `useEmulators`:

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

Add Forms.

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
      const docRef = await addDoc(collection(this.firestore, 'Triggers'), {
        message: this.message,
      });
      this.message = null; // clear form fields
      // read from Firestore
      this.docSnap = await getDoc(doc(this.firestore, 'Triggers', docRef.id));
      this.data$ = this.docSnap.data().message;
      // document listener
      this.unsubMessage$ = onSnapshot(doc(this.firestore, 'Triggers', docRef.id), (snapshot: any) => {
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

## Write your Firebase Cloud Functions

Open `functions/src/index.ts` or `functions/index.js`. Import two Firebase modules, initialize your app, and then write your callable functions.

```ts
const functions = require('firebase-functions');
const admin = require('firebase-admin');
admin.initializeApp();

exports.uppercaseMe = functions.firestore.document('Triggers/{docId}').onCreate((snap, context) => {
    var original = snap.data().message;
    functions.logger.log('Uppercasing', context.params.docId, original);
    var uppercase = original.toUpperCase();
    return snap.ref.set({ uppercase }, { merge: true });
});
```

This Cloud Function has four lines.

The first line stores the message written to Firestore in a local variable `original`.

The second line is the same as `console.log`.

The third line converts the original message to UPPERCASE.

The last line returns something. All Cloud Functions must return something. This line uses `snap.ref` as the Firestore address of the document. It uses `set` with the `merge` parameter to add a field to an existing document. The new field is named `uppercase`. 

## Run emulator

Start the Firebase Emulator.

```bash
firebase emulators:start
```

You should see this with no error messages or warnings:

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

## Run your code

Open your browser to `http://localhost:4200/` and you should see a form. Enter a message and click the `Submit` button. You should see your message appear below the form.

In your Firestore database you should see your message.

## Check the functions logs

Did your Cloud Function run?

Open a browser tab to http://127.0.0.1:4000/functions. You should see your message in the logs. If not, your Cloud Function didn't run in the emulator.

## Deploy to Firebase

Deploy your Cloud Function to Firebase:

```bash
firebase deploy --only functions:uppercaseMe
 ```
 
 You may need to upgrade your project to the `Blaze` (paid) plan. Cloud Functions aren't free.
 
 In `src/environments.ts` change `useEmulators` to `false`:
 
 ```js
 useEmulators: false
 ```
 
Check the logs in your Firebase Console to see if your functions run.

I find it takes five to ten minutes for a Cloud Function to deploy, then I test the Cloud Function, then I wait for the logs. Sometimes there's a lag time between deploying a Cloud Function and the new code running, i.e., the results I get back in the logs sometimes show the previous version of the code I deployed. It's best to wait a minute after deploying a function finishes and testing the function. This ten-minute wait is why the emulators should be used for development.
