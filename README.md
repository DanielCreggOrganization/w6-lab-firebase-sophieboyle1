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
export const environment = {
  production: false,
  firebase: {
    apiKey: "your-api-key",
    authDomain: "your-auth-domain",
    projectId: "your-project-id",
    storageBucket: "your-storage-bucket",
    messagingSenderId: "your-messaging-sender-id",
    appId: "your-app-id"
  }
};
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
import { Injectable } from '@angular/core';
import { Auth, createUserWithEmailAndPassword, signInWithEmailAndPassword } from '@angular/fire/auth';
import { Firestore, doc, setDoc } from '@angular/fire/firestore';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  constructor(
    private auth: Auth,
    private firestore: Firestore
  ) {}

  async register({ email, password }) {
    try {
      const userCredential = await createUserWithEmailAndPassword(
        this.auth,
        email,
        password
      );
      
      await setDoc(doc(this.firestore, `users/${userCredential.user.uid}`), {
        email
      });
      
      return userCredential;
    } catch (error) {
      throw error;
    }
  }

  login({ email, password }) {
    return signInWithEmailAndPassword(this.auth, email, password);
  }

  logout() {
    return this.auth.signOut();
  }
}
```

### Login Page Implementation

```typescript
// src/app/pages/login/login.page.ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { Router } from '@angular/router';
import { AlertController } from '@ionic/angular';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-login',
  templateUrl: './login.page.html',
  styleUrls: ['./login.page.scss'],
})
export class LoginPage {
  credentials: FormGroup;

  constructor(
    private fb: FormBuilder,
    private authService: AuthService,
    private alertController: AlertController,
    private router: Router
  ) {
    this.credentials = this.fb.group({
      email: ['', [Validators.required, Validators.email]],
      password: ['', [Validators.required, Validators.minLength(6)]]
    });
  }

  async register() {
    try {
      await this.authService.register(this.credentials.value);
      this.router.navigateByUrl('/home');
    } catch (error) {
      this.showAlert('Registration failed', error.message);
    }
  }

  async login() {
    try {
      await this.authService.login(this.credentials.value);
      this.router.navigateByUrl('/home');
    } catch (error) {
      this.showAlert('Login failed', error.message);
    }
  }

  private async showAlert(header: string, message: string) {
    const alert = await this.alertController.create({
      header,
      message,
      buttons: ['OK'],
    });
    await alert.present();
  }
}
```

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

1. Implement password reset functionality using Firebase's `sendPasswordResetEmail` method.
2. Add a logout button to the home page header.
3. Create a task modal component for adding new tasks.
4. Implement task editing using the AlertController.
5. Add slide-to-delete functionality for tasks.

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
