# Firebase Lab

## Agenda

1. [Project Setup and Firebase Configuration](#1-project-setup-and-firebase-configuration)
2. [Authentication Implementation](#2-authentication-implementation)
3. [Firestore Integration](#3-firestore-integration)
4. [Route Protection and Guards](#4-route-protection-and-guards)
5. [Task Management Implementation](#5-task-management-implementation)

## 1. Project Setup and Firebase Configuration

### Concept Introduction: Project Architecture

The lab demonstrates building a task management application using Ionic Angular with Firebase services. The architecture combines Firebase Authentication for user management and Firestore for data storage.

```mermaid
graph TD
    A[Ionic Angular App] --> B[Firebase Services]
    B --> C[Authentication]
    B --> D[Firestore Database]
    C --> E[Email/Password Auth]
    D --> F[Tasks Collection]
    A --> G[Components]
    G --> H[Login Page]
    G --> I[Home Page]
    G --> J[Task Modal]
```

### Procedure

1. Create Firebase Project:
   - Navigate to the Firebase website. Hold the ``ctrl`` key on your keyboard and click on this link: [Firebase Console](https://console.firebase.google.com)
   - Sign in to Firebase.
   - Click on the "Create a project" tile.
   - Name it "firebase-ionic-project".
   - Disable Google Analytics and click the ``Create project`` button. This may take a minute to create.
   - Click on the ``Build`` dropdown in the left hand panel, select ``Authentication`` and click the ``Get started`` button. Select Email/Password. Enable Email/Password and click ``Save``
   - Click on the ``Build`` dropdown in the left hand panel, select ``Firestore Database`` and click the ``Create database`` button. Select Location as ``europe-west2 (London)``. Select ``Start in test mode`` and click the ``Create`` button.
   - Click on the Rules tab and change the timestamp date to the last day of the year, December 31st. Click on the ``Publish`` button. 
   - Click on ``Project Overview`` button in the top left of the page and then click on the Web button marked ``</>``. Register a web app in your Firebase project called ``ionic-web-app``. Click the ``Continue to console`` button.
   - We will now integrate our new firebase project with our Ionic app using the below procedure.

2. Configure Firebase Services:
   ```bash
   # Create new Ionic standalone project
   ionic start w6-lab-firebase blank --type=angular

   # Install required tools
   npm install -g @angular/cli
   npm install -g firebase-tools

   # Login to Firebase
   firebase login

   # Install AngularFire
   ng add @angular/fire
   ```

### Configuration Example  
Take a look at your ``main.ts`` file. Make sure you have inputs like shown below:

```typescript
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    { provide: RouteReuseStrategy, useClass: IonicRouteStrategy },
    provideIonicAngular(),
    provideRouter(routes, withPreloading(PreloadAllModules)),
    provideFirebaseApp(() => initializeApp({
      projectId: "fir-ionic-project-dc52e",
      appId: "1:769063483414:web:0b402d09efd31d324dca57",
      storageBucket: "fir-ionic-project-dc52e.appspot.com",
      apiKey: "AIzaSyDibzo0p2mUnQmjN6RlfXlHjbgkzSIUjFY",
      authDomain: "fir-ionic-project-dc52e.firebaseapp.com",
      messagingSenderId: "769063383314"
    })),
    provideAuth(() => getAuth()),
    provideFirestore(() => getFirestore()),
  ],
});
```

## 2. Authentication Implementation

### Concept Introduction: Firebase Authentication

```mermaid
sequenceDiagram
    participant User
    participant App
    participant Firebase
    
    User->>App: Enter Credentials
    App->>Firebase: Authentication Request
    Firebase-->>App: Auth Response
    App-->>User: Success/Error Message
```

### Generate Required Components

```bash
# Generate auth service
ionic g service auth --standalone

# Generate login page
ionic g page login --standalone
```
The login page will look like this:  
![login page](https://github.com/user-attachments/assets/d5710f9c-d0d3-4043-bd5a-44083b8ba81b)


### Auth Service Implementation

```typescript
// src/app/services/auth.service.ts
import { Injectable, inject } from '@angular/core';
import {
  Auth, // Used to get the current user and subscribe to the auth state.
  createUserWithEmailAndPassword, // Used to create a user in Firebase auth.
  signInWithEmailAndPassword, // Used to sign in a user with email and password.
  signOut, // Used to sign out a user.
} from '@angular/fire/auth';
import { doc, Firestore, setDoc } from '@angular/fire/firestore'; // Used to interact with Firestore databse. We store user info in Firestore.

@Injectable({
  providedIn: 'root', // This service is provided in the root injector (AppModule). This means that the service will be available to the entire application.
})
export class AuthService {
  // Inject the Auth and Firestore services. 
  private auth = inject(Auth); // Inject AngularFireAuth service. We need it to create a user in Firebase auth.
  private firestore = inject(Firestore);
  
  constructor() {}
  
  // Sign up with email/password. Creates user in Firebase auth and adds user info to Firestore database
  async register({ email, password }: { email: string; password: string }) {
    try {
      const credentials = await createUserWithEmailAndPassword(
        this.auth,
        email,
        password
      );
      // In case the user is created successfully, create a document in `users` collection
      const ref = doc(this.firestore, `users/${credentials.user.uid}`);
      setDoc(ref, { email }); // Set the document. Data is written to the database.
      return credentials;
    } catch (e) {
      console.log("Error in register: ", e);
      return null;
    }
  }

  // Sign in with email/password. We pass the email and password as parameters.
  async login({ email, password }: { email: string; password: string }) {
    try {
      // Sign in user. If successful, the user object is returned. Otherwise, null is returned.
      const credentials = await signInWithEmailAndPassword(
        this.auth, // <-- Injected AngularFireAuth service
        email, // <-- Email passed as parameter
        password // <-- Password passed as parameter
      );
      return credentials; // <-- Return the user object
    } catch (e) {
      console.log("Error in register: ", e);
      return null;
    }
  }

  logout() {
    return signOut(this.auth);
  }
}
```

### Login Page Implementation

```typescript
// src/app/login/login.page.ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';
import { Router } from '@angular/router';
import { AuthService } from '../auth.service';
import { AlertController, LoadingController } from '@ionic/angular/standalone';
import { IonicModule } from '@ionic/angular';

@Component({
  selector: 'app-login',
  templateUrl: './login.page.html',
  styleUrls: ['./login.page.scss'],
  standalone: true,
  imports: [
    CommonModule,
    ReactiveFormsModule,
    IonicModule
  ],
})
export class LoginPage {

  private fb = inject(FormBuilder); // Inject the FormBuilder to handle form validation in reactive forms
  private loadingController = inject(LoadingController); // Inject the LoadingController to handle loading state by displaying a spinner
  private alertController = inject(AlertController); // Inject the AlertController to handle errors and display alert messages
  private authService = inject(AuthService); // Inject the AuthService to handle login and registration
  private router = inject(Router); // Inject the Router to redirect after successful login

  credentials = this.fb.nonNullable.group({
    email: ['daniel.cregg@atu.ie', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(6)]],
  });

  constructor() { }

  // Easy access for form fields. This is used in the template to display validation errors.
  get email() {
    return this.credentials.controls.email;
  }

  // Easy access for form fields. This is used in the template to display validation errors.
  get password() {
    return this.credentials.controls.password;
  }

  // Register a new user with the AuthService. If successful, redirect to the home page.
  async register() {
    // Create a loading overlay. This will be displayed while the request is running.
    const loading = await this.loadingController.create();
    await loading.present(); // <-- Show loading spinner
    // Call the register method from the AuthService. This returns a user object if successful, or null if unsuccessful.
    const user = await this.authService.register(
      this.credentials.getRawValue() // <-- Pass the raw value of the form fields to the register method
    );
    // Log the user object to the console. This will be `null` if the user was not created.
    console.log(
      '🚀 ~ file: login.page.ts:50 ~ LoginPage ~ register ~ user',
      user
    );
    // Dismiss the loading spinner
    await loading.dismiss();

    // If the user is successfully created, redirect to the home page. Otherwise, display an error.
    if (user) {
      this.router.navigateByUrl('/home', { replaceUrl: true });
    } else {
      this.showAlert('Registration failed', 'Please try again!');
    }
  }

  // The login function is called when the login form is submitted (i.e. the login button is clicked)
  // Login an existing user with the AuthService. If successful, redirect to the home page.
  async login() {
    // Create a loading overlay. This will be displayed while the request is running.
    const loading = await this.loadingController.create();
    await loading.present();
    // Call the login method from the AuthService. This returns a user object if successful, or null if unsuccessful.
    const user = await this.authService.login(this.credentials.getRawValue());
    // Log the user object to the console. This will be `null` if the user was not logged in.
    console.log('🚀 ~ file: login.page.ts:73 ~ LoginPage ~ login ~ user', user);
    // Dismiss the loading spinner
    await loading.dismiss();
    // If the user is successfully logged in, redirect to the home page. Otherwise, display an error via alert.
    if (user) {
      this.router.navigateByUrl('/home', { replaceUrl: true });
    } else {
      this.showAlert('Login failed', 'Please try again!');
    }
  }

  // Add sendReset function to the LoginPage class. This will call the resetPw method from the AuthService.
  // This method will send a password reset email to the email address passed as parameter.
  async sendReset() {
    // Create a loading overlay. This will be displayed while the request is running.
    const loading = await this.loadingController.create();
    await loading.present();
    // Call the resetPw method from the AuthService. This returns a promise.
    await this.authService.resetPw(this.email.value);
    // Dismiss the loading spinner
    await loading.dismiss();
    // Show an alert message
    this.showAlert(
      'Password reset',
      'Check your inbox for the password reset link'
    );
  }

  // Show an alert message with the given header and message.
  async showAlert(header: string, message: string) {
    const alert = await this.alertController.create({
      header,
      message,
      buttons: ['OK'],
    });
    await alert.present();
  }
}
```
### DIY Tasks

1. Implement password reset functionality using Firebase's `sendPasswordResetEmail` method.
2. Add a logout button to the home page header.
   
## 3. Firestore Integration

### Concept Introduction: Firestore Database

```mermaid
graph LR
    A[Firestore] --> B[Collections]
    B --> C[Tasks]
    C --> D[Document 1]
    C --> E[Document 2]
    D --> F[title: string<br/>description: string<br/>userId: string]
    E --> F
```

### Task Service Implementation

```typescript
// src/app/services/tasks.service.ts
import { Injectable } from '@angular/core';
import { Firestore, collection, addDoc, query, where, getDocs, deleteDoc, updateDoc } from '@angular/fire/firestore';
import { Auth } from '@angular/fire/auth';

export interface Task {
  id?: string;
  title: string;
  description: string;
  userId: string;
}

@Injectable({
  providedIn: 'root'
})
export class TasksService {
  constructor(
    private firestore: Firestore,
    private auth: Auth
  ) {}

  async addTask(task: Task) {
    const userId = this.auth.currentUser?.uid;
    const tasksRef = collection(this.firestore, 'tasks');
    return addDoc(tasksRef, { ...task, userId });
  }

  async getTasks() {
    const userId = this.auth.currentUser?.uid;
    const tasksRef = collection(this.firestore, 'tasks');
    const q = query(tasksRef, where('userId', '==', userId));
    const querySnapshot = await getDocs(q);
    
    return querySnapshot.docs.map(doc => ({
      id: doc.id,
      ...doc.data()
    }));
  }

  // Additional CRUD operations...
}
```

[Continue with sections 4 and 5...]

### DIY Tasks

1. Create a task modal component for adding new tasks.
2. Implement task editing using the AlertController.
3. Add slide-to-delete functionality for tasks.

## Common Issues and Troubleshooting

1. Firebase Initialization Errors:
   - Check if environment variables are correctly configured
   - Ensure Firebase services are enabled in console

2. Authentication Errors:
   - Verify email/password requirements
   - Check Firebase console for auth settings

3. Firestore Permission Errors:
   - Review security rules
   - Verify user authentication state

## Additional Resources

- [Firebase Documentation](https://firebase.google.com/docs)
- [Ionic Framework Documentation](https://ionicframework.com/docs)
- [AngularFire Documentation](https://github.com/angular/angularfire)

---
End of Lab
