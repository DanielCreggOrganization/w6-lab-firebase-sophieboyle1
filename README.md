[![Open in Codespaces](https://classroom.github.com/assets/launch-codespace-2972f46106e565e64193e422d61a12cf1da4916b45550586e14ef0a7c637dd04.svg)](https://classroom.github.com/open-in-codespaces?assignment_repo_id=17419244)
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
ionic g service auth

# Generate login page
ionic g page login --standalone
```
The login page will look like this:  
![login page](https://github.com/user-attachments/assets/d5710f9c-d0d3-4043-bd5a-44083b8ba81b)


### Auth Service Implementation

```typescript
// src/app/services/auth.service.ts
/**
 * Service responsible for handling authentication operations
 * including user registration, authentication, password reset, and sign out.
 */
import { Injectable, inject } from '@angular/core';
import {
  Auth,
  createUserWithEmailAndPassword,
  sendPasswordResetEmail,
  signInWithEmailAndPassword,
  signOut,
  User,
  UserCredential,
} from '@angular/fire/auth';

/**
 * Interface for authentication request data
 */
interface UserAuthData {
  email: string;
  password: string;
}

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  /**
   * Firebase Authentication instance
   */
  private readonly firebaseAuth = inject(Auth);

  /**
   * Registers a new user with email and password
   * @param userAuthData - The user's email and password
   * @returns Promise resolving to UserCredential
   */
  async registerUser(userAuthData: UserAuthData): Promise<UserCredential> {
    return createUserWithEmailAndPassword(
      this.firebaseAuth,
      userAuthData.email,
      userAuthData.password
    );
  }

  /**
   * Authenticates a user with email and password
   * @param userAuthData - The user's email and password
   * @returns Promise resolving to UserCredential
   */
  async authenticateUser(userAuthData: UserAuthData): Promise<UserCredential> {
    return signInWithEmailAndPassword(
      this.firebaseAuth,
      userAuthData.email,
      userAuthData.password
    );
  }

  /**
   * Gets the current authenticated user
   * @returns The current User or null
   */
  fetchActiveUser(): User | null {
    return this.firebaseAuth.currentUser;
  }

  /**
   * Initiates password reset process for a user
   * @param userEmail - The email address for password reset
   * @returns Promise resolving when email is sent
   */


  /**
   * Signs out the current user
   * @returns Promise resolving when sign out is complete
   */
  async signOutUser(): Promise<void> {
    return signOut(this.firebaseAuth);
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
import { IonicModule } from '@ionic/angular';
import { 
  mailOutline, 
  lockClosedOutline, 
  eyeOutline, 
  eyeOffOutline,
  logInOutline,
  personAddOutline 
} from 'ionicons/icons';
import { addIcons } from 'ionicons';

@Component({
  selector: 'app-login',
  templateUrl: './login.page.html',
  standalone: true,
  imports: [
    CommonModule,
    ReactiveFormsModule,
    IonicModule
  ],
})
export class LoginPage {
  private auth = inject(AuthService);
  private router = inject(Router);
  private fb = inject(FormBuilder);

  isPasswordVisible = false;

  userAuthForm = this.fb.nonNullable.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(6)]],
  });

  constructor() {
    addIcons({
      mailOutline,
      lockClosedOutline,
      eyeOutline,
      eyeOffOutline,
      logInOutline,
      personAddOutline
    });
  }

  async handleRegistration() {
    if (this.userAuthForm.valid) {
      await this.auth.registerUser(this.userAuthForm.getRawValue());
      await this.router.navigateByUrl('/home', { replaceUrl: true });
    }
  }

  async handleAuthentication() {
    if (this.userAuthForm.valid) {
      await this.auth.authenticateUser(this.userAuthForm.getRawValue());
      await this.router.navigateByUrl('/home', { replaceUrl: true });
    }
  }

  async handlePasswordReset() {


  }
}
```

### DIY Tasks

1. Implement password reset functionality using Firebase's `sendPasswordResetEmail` method. You will need to update the auth service and the login page.
2. Add the Create Account button.
   
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
// src/app/tasks.service.ts
/**
 * This service manages task data using Firebase/Firestore.
 * It handles user authentication state and provides methods for CRUD operations on tasks.
 */
import { Injectable, inject } from '@angular/core';
import { Auth, onAuthStateChanged } from '@angular/fire/auth';
import {
  collection,
  collectionData,
  deleteDoc,
  doc,
  Firestore,
  query,
  updateDoc,
  where,
  addDoc,
} from '@angular/fire/firestore';
import { Observable, BehaviorSubject, switchMap, of } from 'rxjs';

/**
 * Interface defining the structure of a Task.
 * @property id - Optional unique identifier (provided by Firestore)
 * @property content - The task description or content
 * @property completed - Boolean indicating if the task is done
 * @property user - Optional user ID who owns the task
 */
export interface Task {
  id?: string;
  content: string;
  completed: boolean;
  user?: string;
}

/**
 * Service responsible for managing tasks in the application.
 * Uses Firebase Authentication and Firestore for data persistence.
 */
@Injectable({ providedIn: 'root' })
export class TasksService {
  // Inject Firestore service to interact with Firestore and access tasks collection
  private readonly firestoreDb = inject(Firestore);
  // Inject the Auth service to manage user authentication
  private readonly authService = inject(Auth);
  // Reference to the 'tasks' collection in Firestore database used to interact with tasks
  private readonly tasksCollectionRef = collection(this.firestoreDb, 'tasks');
  
  /**
   * BehaviorSubject keeping track of the current authenticated user.
   * BehaviorSubject is used because it:
   * 1. Requires an initial value
   * 2. Caches the latest value
   * 3. Emits the latest value to new subscribers
   */
  private readonly authenticatedUser$ = new BehaviorSubject(this.authService.currentUser);

  /**
   * Observable stream of tasks that automatically updates based on authentication state.
   * Using switchMap to cancel previous subscriptions when user changes and create
   * new subscriptions for the new user state.
   */
  readonly userTasks$ = this.authenticatedUser$.pipe(
    switchMap(user => !user ? of([]) : 
      collectionData(
        query(this.tasksCollectionRef, where('user', '==', user.uid)),
        { idField: 'id' }
      ) as Observable<Task[]>
    )
  );

  constructor() {
    // Set up authentication state listener
    onAuthStateChanged(this.authService, user => this.authenticatedUser$.next(user));
  }

  /**
   * Returns an Observable of all tasks for the current user.
   * The Observable automatically updates when:
   * 1. User logs in/out
   * 2. Tasks are added/modified/deleted
   */
  getUserTasks(): Observable<Task[]> {
    return this.userTasks$;
  }

  /**
   * Creates a new task in Firestore.
   * @param task The task to create (without id and user)
   * @returns Promise that resolves when the task is created
   */
  async createTask(task: Task) {
    const userId = this.authService.currentUser?.uid;
    return addDoc(this.tasksCollectionRef, { ...task, user: userId });
  }

  /**
   * Updates an existing task's content in Firestore.
   * @param task The task to update (must include id)
   * @returns Promise that resolves when the update is complete
   */
  async updateTask({ id, content }: Task) {
    return updateDoc(doc(this.firestoreDb, `tasks/${id}`), { content });
  }

  /**
   * Deletes a task from Firestore.
   * @param task The task to delete (must include id)
   * @returns Promise that resolves when the deletion is complete
   */
  async deleteTask({ id }: Task) {
    return deleteDoc(doc(this.firestoreDb, `tasks/${id}`));
  }

  /**
   * Toggles the completed status of a task in Firestore.
   * @param task The task to toggle (must include id and completed status)
   * @returns Promise that resolves when the update is complete
   */
  async toggleTaskCompleted({ id, completed }: Task) {
    return updateDoc(doc(this.firestoreDb, `tasks/${id}`), { completed });
  }
}
```

We will use the Home page to dispay all tasks.  
![tasks image](https://github.com/user-attachments/assets/e6137af5-e7a6-44ea-82e1-6f4ec0fc56c3)

```typescript
// src/app/home/home.page.ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Router } from '@angular/router';
import {
  AlertController,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonButton,
  IonButtons,
  IonList,
  IonItemSliding,
  IonItem,
  IonLabel,
  IonIcon,
  IonCheckbox,
  IonItemOptions,
  IonItemOption,
  IonFooter,
  IonText,
  IonFab,
  IonFabButton,
} from '@ionic/angular/standalone';
import { addIcons } from 'ionicons';
import { logOutOutline, pencilOutline, trashOutline, add } from 'ionicons/icons';
import { AuthService } from '../auth.service';
import { TasksService, Task } from '../tasks.service';

@Component({
  selector: 'app-home',
  templateUrl: './home.page.html',
  standalone: true,
  imports: [
    CommonModule,
    IonHeader,
    IonToolbar,
    IonTitle,
    IonContent,
    IonButton,
    IonButtons,
    IonIcon,
    IonList,
    IonItemSliding,
    IonItem,
    IonLabel,
    IonCheckbox,
    IonItemOptions,
    IonItemOption,
    IonFooter,
    IonText,
    IonFab,
    IonFabButton,
  ],
})
export class HomePage {
  private authService = inject(AuthService);
  private tasksService = inject(TasksService);
  private routerService = inject(Router);
  private alertController = inject(AlertController);

  // Get the user's tasks from the tasks service
  userTasks$ = this.tasksService.getUserTasks();
  // Get the current user from the auth service to display the user's email at the bottom of the page
  currentUser = this.authService.fetchActiveUser();

  constructor() {
    // Add the necessary icons to the page
    addIcons({ logOutOutline, pencilOutline, trashOutline, add });
  }

  async addTask() {
    const alert = await this.alertController.create({
      header: 'New Task',
      inputs: [
        {
          name: 'content',
          type: 'text',
          placeholder: 'Enter task description'
        }
      ],
      buttons: [
        { text: 'Cancel', role: 'cancel' },
        {
          text: 'Add',
          handler: (data) => {
            if (data.content?.trim()) {
              this.tasksService.createTask({
                content: data.content,
                completed: false
              });
            }
          }
        }
      ]
    });

    await alert.present();
  }

  async toggleTask(event: Event, task: Task) {
    task.completed = (event as CustomEvent).detail.checked;
    await this.tasksService.toggleTaskCompleted(task);
  }

  async editTask(task: Task, slidingItem: IonItemSliding) {
    const alert = await this.alertController.create({
      header: 'Update Task',
      inputs: [{ 
        name: 'content', 
        value: task.content, 
        type: 'text' 
      }],
      buttons: [
        { 
          text: 'Cancel', 
          role: 'cancel',
          handler: () => slidingItem.close()
        },
        { 
          text: 'Update',
          handler: (data) => {
            this.tasksService.updateTask({ ...task, content: data.content });
            slidingItem.close();
          }
        }
      ]
    });

    await alert.present();
  }

  async deleteTask(task: Task) {
    await this.tasksService.deleteTask(task);
  }

  async signOut() {
    await this.authService.signOutUser();
    await this.routerService.navigateByUrl('/', { replaceUrl: true });
  }
}
```

Here is the Template for the home page.

```html
<ion-header>
  <ion-toolbar color="primary">
    <ion-title>Tasks</ion-title>
    <ion-buttons slot="end">
      <ion-button (click)="signOut()">
        <ion-icon slot="icon-only" name="log-out-outline"></ion-icon>
      </ion-button>
    </ion-buttons>
  </ion-toolbar>
</ion-header>

<ion-content>
  <ion-list>
    <!-- Empty state -->
    @if ((userTasks$ | async)?.length === 0) {
    <ion-item>
      <ion-label class="ion-text-center">
        <p>Add your first task!</p>
      </ion-label>
    </ion-item>
    }

    <!-- Task list -->
    @for (task of userTasks$ | async; track task.id) {
    <ion-item-sliding #slidingItem>
      <ion-item>
        <ion-label [color]="task.completed ? 'medium' : ''">
          {{task.content}}
        </ion-label>
      </ion-item>

      <ion-item-options>
        <ion-item-option color="primary" (click)="editTask(task, slidingItem)">
          <ion-icon name="pencil-outline"></ion-icon>
        </ion-item-option>
        <ion-item-option color="danger" (click)="deleteTask(task)">
          <ion-icon name="trash-outline"></ion-icon>
        </ion-item-option>
      </ion-item-options>
    </ion-item-sliding>
    }
  </ion-list>
</ion-content>

<ion-footer class="ion-no-border">
  <ion-toolbar class="ion-text-center">
    <ion-buttons slot="end">
      <ion-button (click)="addTask()" fill="solid" color="primary" class="ion-no-margin" shape="round">
        <ion-icon slot="icon-only" name="add" color="light"></ion-icon>
      </ion-button>
    </ion-buttons>
  </ion-toolbar>
</ion-footer>
```

### DIY Tasks
1. Add a check box to each task to toggle the task complete attribute.
2. Add a "Logged in user: user email" to the bottom of the home page. Hint: Use an ion-text item in the footer.
3. Add Try catch error checking to the methods in you auth and tasks services.

## Additional Resources

- [Firebase Documentation](https://firebase.google.com/docs)
- [Ionic Framework Documentation](https://ionicframework.com/docs)
- [AngularFire Documentation](https://github.com/angular/angularfire)

---
End of Lab
