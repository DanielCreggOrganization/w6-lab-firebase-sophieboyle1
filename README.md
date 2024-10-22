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

1. Implement password reset functionality using Firebase's `sendPasswordResetEmail` method. You will need to update the auth service and the login page. 
   
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
import { Injectable, inject } from '@angular/core'; // Used to make the service injectable.
import { Auth, onAuthStateChanged } from '@angular/fire/auth'; // Used to get the current user and subscribe to the auth state.
import {
  addDoc, // Used to add a document to Firestore.
  collection, // Used to create a reference to a collection in Firestore.
  collectionData, // Used to create an observable that will emit the current value of the tasks array.
  CollectionReference, // Used to create a reference to a collection in Firestore.
  deleteDoc, // Used to delete a document in Firestore.
  doc, // Used to get a reference to a document in Firestore.
  Firestore, // Used to interact with Firestore.
  query, // Used to create a query to get the tasks for the current user.
  updateDoc, // Used to update a document in Firestore.
  where, // Used to create a query to get the tasks for the current user.
} from '@angular/fire/firestore'; // Import the functions needed to interact with Firestore.
import { BehaviorSubject, Observable, Subscription } from 'rxjs'; // Used to create an observable that will emit the current value of the tasks array.

// Task is an interface that defines the structure of a task. The ? after the property name means that the property is optional.
export interface Task {
  id?: string;            // The id is optional because Firestore does not store the id in the document.
  content: string;
  completed: boolean;
  user?: string;          // The user id is optional because Firestore does not store the user id in the document.
}

// The @Injectable decorator is used to make the service injectable. The service is injected into the constructor.
// The providedIn option is used to specify that the service should be provided in the root injector (AppModule).
// This means that the service will be available to the entire application.
@Injectable({
  providedIn: 'root',
})
export class TasksService {

  private firestore = inject(Firestore);
  private auth = inject(Auth);

  // Create a reference to the tasks collection. This is a reference to the collection in Firestore.
  private collectionRef: CollectionReference;
  // Create a BehaviorSubject observable. This stores the current value of tasks and will emit its current value to any new subscribers immediately upon subscription
  private tasks$: BehaviorSubject<Task[]> = new BehaviorSubject<Task[]>([]);
  // Create a subscription to the tasks collection. This is a subscription to the collection in Firestore.
  private tasksSub!: Subscription;

  constructor() {
    // Create a reference to the tasks collection. This is a reference to the collection in Firestore.
    this.collectionRef = collection(this.firestore, 'tasks'); // The second argument is the path to the collection in Firestore.
    // Subscribe to the auth state. This will allow us to subscribe to the tasks collection when the user logs in.
    this.subscribeToAuthState();
  }

  /**
   * Subscribes to the authentication state of the application.
   * This method is called when the authentication state changes (i.e., when a user logs in or out).
   * If a user is logged in, it subscribes to the tasks of the logged-in user by calling `subscribeToTasks`.
   * If no user is logged in, it unsubscribes from the tasks by calling `unsubscribeFromTasks`.
   */
  private subscribeToAuthState(): void {
    onAuthStateChanged(this.auth, (user) => { // When the authentication state changes, check if a user is logged in. 
      if (user) { // If a user is logged in
        this.subscribeToTasks(user.uid); // subscribe to users tasks
      } else { // If no user is logged in 
        this.unsubscribeFromTasks(); // unsubscribe from tasks. This will save resources and prevent errors.
      }
    });
  }

  private subscribeToTasks(userId: string): void {
    // Create a query to get only the tasks for the current user.
    const tasksQuery = query(this.collectionRef, where('user', '==', userId));

    // Create an observable that will emit the current value of the tasks array.
    const tasks$ = collectionData(tasksQuery, {
      idField: 'id', // Include the document ID in the emitted data, under the field name 'id'.
    }) as Observable<Task[]>; // Treat the result of collectionData as an Observable that emits arrays of Task objects

    // Subscribing to an Observable. This is the process of connecting a consumer (usually a function) to the Observable.
    // When you subscribe to an Observable, you provide a function that will be called each time the Observable emits a new value. 
    // In this case, the function takes one argument, tasks, which will be the new value emitted by the Observable.
    this.tasksSub = tasks$.subscribe((tasks) => {
      this.tasks$.next(tasks); // Calling next emits a new value to its subscribers. In this case, it's emitting the tasks value that was just received from collectionSub and it's emitting it to the tasks$ BehaviorSubject.
    });
  }

  // Clear Tasks and unsubscribe from tasks observable. This saves resources and prevents errors.
  private unsubscribeFromTasks(): void {
    this.tasks$.next([]); // Clear tasks by emitting an empty array to the tasks$ BehaviorSubject.
    if (this.tasksSub) { // If there is a subscription to the tasks collection
      this.tasksSub.unsubscribe(); // unsubscribe from the tasks collection
    }
  }

  // Create a task and add it to the tasks collection. This will add a document to the collection on Firestore.
  async createTask(task: Task) {
    try {
      await addDoc(this.collectionRef, { // Add a document to the collection. The first argument is the reference to the collection. The second argument is the document to add to the collection.
        ...task, // Use the spread operator to add the task properties to the document.
        user: this.auth.currentUser?.uid, // Add the user id to the document. This will allow us to query the tasks for the current user.
      });
    } catch (error) {
      console.error('Error creating task:', error);
    }
  }

  // Return the tasks BehaviorSubject as an observable. This will allow us to subscribe to the tasks array.
  // The async keyword is not needed here because we are not calling to firestore.
  readTasks() {
    return this.tasks$.asObservable(); //returning an Observable version of the tasks$ BehaviorSubject that can be safely exposed to consumers
  }

  updateTask(task: Task) {
    // Use the task id to get the reference to the document
    const ref = doc(this.firestore, `tasks/${task.id}`);
    // Update the document. Here we set the value of the content field to the value of the task.content
    return updateDoc(ref, { content: task.content });
  }

  async deleteTask(task: Task) {
    try {
      // Use the task id to get the reference to the document
      const ref = doc(this.firestore, `tasks/${task.id}`);
      // Delete the document
      await deleteDoc(ref);
    } catch (error) {
      // Log the error to the console
      console.error('Error deleting document: ', error);
    }
  }

  // This method is used update the checkbox in the Firestore database when the user toggles the checkbox in the UI.

}
```

We will use the Home page to dispay all tasks.  
![tasks image](https://github.com/user-attachments/assets/e6137af5-e7a6-44ea-82e1-6f4ec0fc56c3)

```typescript
// src/app/home/home.page.ts
import { AfterViewInit, Component, ViewChild, inject, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Router } from '@angular/router';
import { FormsModule } from '@angular/forms';
import {
  AlertController, 
  LoadingController, 
  CheckboxCustomEvent, 
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
  IonModal, 
  IonInput, 
  IonRow, 
  IonCol, 
  IonFab, 
  IonFabButton, 
} from '@ionic/angular/standalone';
import { addIcons } from 'ionicons';
import { Observable } from 'rxjs';
import { logOutOutline, pencilOutline, trashOutline, add } from 'ionicons/icons';
import { AuthService } from '../auth.service';
import { TasksService, Task } from '../tasks.service';

@Component({
  selector: 'app-home',
  templateUrl: 'home.page.html',
  styleUrls: ['home.page.scss'],
  standalone: true,
  imports: [
    CommonModule,
    FormsModule,
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
    IonModal,
    IonInput,
    IonRow,
    IonCol,
    IonFab,
    IonFabButton,
  ],
})

export class HomePage implements AfterViewInit {
  newTask!: Task; // This is the task that will be added to the database.
  @ViewChild(IonModal) modal!: IonModal; // Find the first IonModal in my template and assign it to the modal property of my class.
  tasks$!: Observable<Task[]>; // This is an observable that will emit the current value of the tasks array. // This is an observable that will emit the current value of the tasks array.
  
  private authService = inject(AuthService);
  private tasksService = inject(TasksService);
  private router = inject(Router);
  private loadingController = inject(LoadingController);
  private alertController = inject(AlertController);

  constructor() {
    //this.resetTask();
    //addIcons({ logOutOutline, pencilOutline, trashOutline, add });
  }

  ngOnInit() {
    this.resetTask();
    addIcons({ logOutOutline, pencilOutline, trashOutline, add });
    this.tasks$ = this.tasksService.readTasks();
  }

  // This method is used to reset the newTask property. This will clear the input in the modal.
  resetTask() {
    this.newTask = {
      content: '',
      completed: false,
    };
  }

  // This method is used to log the user out. The button will be found in the top right corner of the home page.
  async logout() {
    // Call the logout method in the auth service. Use await to wait for the logout to complete before continuing.
    await this.authService.logout();
    // Navigate to the login page with the replaceUrl option.
    // This means that the login page will replace the home page in the navigation stack.
    this.router.navigateByUrl('/', { replaceUrl: true });
  }

  // This method is used to add a task to the database
  async addTask() {
    const loading = await this.loadingController.create();
    // await means that the code will wait for the loading to be presented before continuing
    await loading.present();
    // Add the task to the database
    this.tasksService.createTask(this.newTask);
    // Dismiss the loading
    await loading.dismiss();
    // Dismiss the modal
    this.modal.dismiss(null, 'confirm');
    // Reset the task. This will clear the input in the modal.
    this.resetTask();
  }

  // This method is used to update the checkbox in the UI when the user toggles the checkbox


  async updateTask() {
    await this.tasksService.updateTask(this.newTask);
    this.resetTask();
  }

  async openUpdateInput(task: Task) {
    const alert = await this.alertController.create({
      header: 'Update Task',
      inputs: [
        {
          name: 'Task',
          type: 'text',
          placeholder: 'Task content',
          value: task.content,
        },
      ],
      buttons: [
        {
          text: 'Cancel',
          role: 'cancel',
        },
        {
          text: 'Update',
          // Call the updateTask method when the user clicks the update button
          handler: (data) => {
            task.content = data.Task;
            this.tasksService.updateTask(task);
          },
        },
      ],
    });
    await alert.present(); // Present the alert to the user
    // Get the alert's first input element and focus the mouse blinker on it.
    // The setTimeout function is used to allow some time for the browser to render the alert's DOM elements.
    setTimeout(() => {
      const firstInput: any = document.querySelector('ion-alert input');
      firstInput.focus();
    }, 250);
  }

  deleteTask(task: Task) {
    // Print task to console
    console.log('Deleting task: ', task);
    this.tasksService.deleteTask(task);
  }

  // The method is used inside the modal to close the modal and reset the newTask property.
  cancel() {
    this.modal.dismiss(null, 'cancel');
    this.resetTask();
  }

  // This method is used to focus the cursor in the input box of the modal when we open it. We subscribe to
  // the ionModalDidPresent event of the modal. When the modal is presented, we use setTimeout to wait for
  // the browser to render the modal's DOM elements, then we select the first input element in the modal and focus on it.
  ngAfterViewInit() {
    this.modal.ionModalDidPresent.subscribe(() => {
      setTimeout(() => {
        const firstInput: any = document.querySelector('ion-modal input');
        firstInput.focus();
      }, 250);
    });
  }
}
```
Here is the Template for the home page
```html
<!-- Add a header to the page with a logout button and the title "My Tasks" -->
<ion-header>
  <ion-toolbar color="primary">
    <ion-buttons slot="end">
      <ion-button (click)="logout()">
        <ion-icon slot="icon-only" name="log-out-outline"></ion-icon>
      </ion-button>
    </ion-buttons>
    <ion-title> My Tasks </ion-title>
  </ion-toolbar>
</ion-header>

<ion-content>
  <ion-list>
    <!-- The tasks array is an observable that emits an array of tasks. The | async pipe is used to subscribe 
         to the tasks observable and unwrap the emitted value. The home.page.html file knows about the tasks 
         property because it is bound to it using the *ngFor directive. The *ngFor directive is used to loop 
         over the tasks array and generate HTML elements for each task.-->
    <ion-item-sliding *ngFor = "let task of tasks$ | async">
      <ion-item>
        <!-- Display the task content -->
        <ion-label>
          <h3>{{task.content}}</h3>
        </ion-label>
        <!-- Display a checkbox to mark the task as completed. Use event binding that listens for the ionChange event. 
             This event is emitted by Ionic components when their value changes (i.e. the tick box is checked or unchecked)
             [checked]="task.completed" is an example of property binding.the checked property of an element is being bound
             to the completed property of the task object   -->
        <ion-checkbox
          slot="end"
          (ionChange)="toggleTask($event, task)"
          [checked]="task.completed"
          aria-label="Task checkbox"
        >
        </ion-checkbox>
      </ion-item>
      <!-- Create a sliding button to edit the task title or delete the task -->
      <ion-item-options side="end">
        <ion-item-option color="primary" (click)="openUpdateInput(task)">
          <ion-icon name="pencil-outline" slot="icon-only"></ion-icon>
        </ion-item-option>
        <ion-item-option color="danger" (click)="deleteTask(task)">
          <ion-icon name="trash-outline" slot="icon-only"></ion-icon>
        </ion-item-option>
      </ion-item-options>
    </ion-item-sliding>
  </ion-list>

  <!-- Create a pop up modal to add a new task. Trigger the modal with FAB button created below. -->
  <!-- #modal is used to access the modal in the template -->
  <ion-modal trigger="open-modal" #modal>
    <!-- ng-template is used with the ViewChild decorator to create a template reference variable, 
         which is then used to present the modal. The ViewChild decorator is used to access the modal
         in the component class. The #modal variable is used to access the modal in the template.-->
    <ng-template>
      <!-- In the modal, create an input to enter a new task. Use two-way binding to bind the input to the newTask property -->
      <ion-item>
        <ion-label position="stacked">New Task</ion-label>
        <ion-input
          type="text"
          placeholder="Enter task content here..."
          [(ngModel)]="newTask.content"
        ></ion-input>
      </ion-item>

      <!-- Add buttons in the modal to save or cancel the task entry. Place them side by side using rows and columns. -->
      <ion-row>
        <ion-col>
          <ion-button (click)="addTask()" color="primary" expand="full">
            Save
          </ion-button>
        </ion-col>
        <ion-col>
          <ion-button (click)="cancel()" color="danger" expand="full">
            Cancel
          </ion-button>
        </ion-col>
      </ion-row>
    </ng-template>
  </ion-modal>

  <!-- FAB = Floating Action Button. This creates a button that is always visible on the screen.
       The button contains a plus symbol and is used to add a new task -->
  <ion-fab slot="fixed" vertical="bottom" horizontal="end">
    <ion-fab-button id="open-modal">
      <ion-icon name="add"></ion-icon>
    </ion-fab-button>
  </ion-fab>
</ion-content>
```

### DIY Tasks

1. Add a method to toggle the Task completed atribute. Use a checkbox as seen in the image above.

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
