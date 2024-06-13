# Progress Notification API & Background Tasks with Service Workers

## Abstract
This explainer lays out a proposal for allowing the Progress Notification API to be used in Service Workers to perform tasks when there are no visible documents opened for the site. How the user agent presents this information to the user is a UX design question which may be left up to implementations; however, it should be carefully considered for security and privacy.

## Motivating Use Cases
Use cases for fall into 2 general categories:
* Extending the lifetime of a service worker when important work shouldn’t be interrupted by document closure
* Extending the lifetime of a service worker when important work is triggered by a background event
  

### Background File System Access
There are two primary ways in which sites may want to access the file system from the background:
1) Completing long-running file operations after a tab has been closed
2) Responding to a change in the underlying file system

(1) Similar to the System Wake Lock use case, it is a poor experience for the user if any long-running file operation (e.g. caching a multi-GB movie to watch offline or syncing the contents of an inbox) requires them to idle on the open tab, waiting for the operation to complete. This would allow the user to initiate the action, close the tab, and receive a cancellable notification while the operation is in progress.

(2) File System Change Observers outlines a new API for listening to changes in the local file system. By hooking into the Progress Notification API, a site which does not have an open tab (e.g. a sync client) can keep the user informed while responding to these change events from a Service Worker (e.g. by syncing the newly added files).

### Multi-app VDI Sessions
The [Multi-apps API](https://github.com/ivansandrk/multi-apps/blob/main/explainer.md) allows a single installed web app to integrate with the OS as if it were multiple independent apps. This is useful for VDI clients as part of providing a “seamless” experience where each application from the remote desktop is streamed independently to a different local window instead of providing a single window showing a full desktop environment with multiple windows inside.

Even though each app appears separate to the user, they may be backed by a single connection back to the remote desktop instance which must be managed somewhere. One option is to have a special “session” window which owns the connection. This is problematic because it’s unclear to the user why this particular window is special, it may be closed accidentally, and generally just gets in the way. These UX concerns can be resolved by moving the connection into a Shared Worker. This introduces a new problem because now the VDI session will immediately terminate as soon as the last app window is closed. If initiating the connection is expensive it is preferable to keep the session active in case the user immediately opens a new app. For example, closing a document in a remote word processor only to immediately open a different document in a remote spreadsheet program could require reauthenticating their VDI session.

Managing the session connection in a service worker and creating a progress notification representing the active session solves all these problems. As long as any remote app is in use the notification could be hidden because the service worker has an active client. As soon as the last client is closed the notification would be displayed informing the user that their VDI session is still active and offering the opportunity to end it. If a different remote app is opened instead the notification would simply disappear, no longer necessary.

### Low-level Device APIs
Like the connection back to the remote desktop instance in the VDI session example above, when accessing a device with the WebUSB, Web Bluetooth, WebHID or Web Serial APIs only one JavaScript environment can have exclusive control over the device. A complex application like a programming tool might allow you to open source files in different tabs. If the device connection were owned by a particular document, features like uploading or debugging would only be available in a single tab. The developer could work around this with a Shared Worker, as above, but the same problems persist as well. A Service Worker is a safer place to put the interface between the application logic and the low-level device.

Creating a progress notification could also allow a Service Worker to connect to a device when it has been launched because a “connect” event was fired on navigator.usb or one of the other device API global objects. As with receiving a “push” event a Service Worker could be required to display some kind of notification in response to being given an opportunity to execute code because of one of these events. A progress notification (in contrast to a regular notification) would allow the script to express the idea that it is responding to the event by performing a chunk of work.

## Protecting a task from document closure
This is an example of a long running task initiated from a Service Worker that would be protected from document closure until task completion.

```js
// index.js
navigator.serviceWorker.register(sw.js');

function uploadFiles() {
  const directoryHandle = await window.showDirectoryPicker();
  let registration = await navigator.serviceWorker.getRegistration();
  registration.postMessage({action: 'upload-files',
                            directory: directoryHandle});
}

// sw.js 
onmessage = async function(event) {
  switch(event.data.action) {
    case 'upload-files':
      const progress = new ProgressNotification("Uploading files...",
                                                { type: 'determinate'});
      while(!uploadTask.complete()) {
        uploadTask.step();
        progress.update(uploadTask.calculateProgress());
      }
      progress.close();
  }
}
```

## Waking up a Service Worker to perform a task
This example shows Service Worker code that will react to a file system change event, and creates a `ProgressNotification` to perform a long running task. 

```js
// sw.js
self.addEventListener('onfilesystemchange', event => {
  const task = new FileUploadTask(event.records);
  const progress = new ProgressNotification("Uploading files....",
                                            {type: 'determinate'});
  while (!task.complete()){
    task.step();
    progress.update(task.calculateProgress());
  }
  progress.close();
});
```

## UI Design Considerations
Browser implementations should display the following when running a background task
* Origin of the site
* Button to abort background work
* Option to never allow background work

Other UI considerations
* Provided title
* Progress bar or animation for showing progress 
* On task completion, user agents may mark UI as complete but keep the UI persisted for users to see to prevent sites from performing work surreptitiously

## Security Considerations
As with the notification APIs this proposal creates a UI surface under developer control. However without a document, this would allow a malicious script to execute in a scenario where the user may be less likely to notice it. Implementations must be careful to provide UI to inform users of background tasks and provide options to control them. They also must ensure that the developer-specified message is presented with context to prevent it from being confused with browser UI. The ability to extend Service Worker lifetime beyond that of an active document may be limited to installed web apps in order to better match user expectations.

## Privacy Considerations
Use cases above involve allowing a site to continue doing work in situations where currently they cannot. APIs like Push and (Periodic) Background Sync, which can cause a Service Worker to launch when a tab is not loaded, are designed to limit the frequency and amount of script that can execute to mitigate these risks. The implementation of Push in Chromium requires the Service Worker to display a notification (since that is the primary use case of the API anyways) and will warn the user if it does not. The Progress Notification API is designed to leverage a similar scheme by allowing sites to continue working after their tab has been closed but continuing to provide the user with an indication that the site is doing something. The notification can include a button to stop the action and give the user the option to prevent the site from doing background work in the future. This keeps the user in control.

APIs which allow a Service Worker to be launched in response to an event should consider requiring a Notification or Progress Notification to be shown. Browsers may keep a Progress Notification visible even after script has marked it complete in order to notify the user that the site performed some work in the past and prevent sites from rapidly showing and hiding the notification so that they can perform work surreptitiously.
