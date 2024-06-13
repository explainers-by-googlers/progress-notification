# Progress Notification API Explainer

## Authors
- Reilly Grant
- Ayu Ishii

## Participate
- https://github.com/explainers-by-googlers/progress-notification/issues


## Abstract
The Progress Notification API is a proposal for a generic API which a developer can use to indicate to the user agent that it is performing an important task which will take some time. How the user agent presents this information to the user is up to browser implementors; however an indication of an operation in progress may be a prerequisite for the developer to use other APIs. This provides user agents with information about a site’s intention to perform such a task and therefore may allow the task to continue while the site is no longer visible because its tab is occluded.

## Motivating Use Cases

### System Wake Lock
A system wake lock allows an application to keep the system from entering a sleep state. It is similar to a screen wake lock, which prevents the display from turning off due to inactivity. Some APIs already implicitly acquire a screen or system wake lock, for example, playing audio or video. The challenge for developers is that if their application isn’t performing one of these tasks but still has work in progress (for example, encoding an image for upload) then the system could go to sleep, particularly if the user puts their device down to wait for the operation to complete. This creates a poor user experience when the user returns to the device expecting the operation to be complete only to discover it stopped after a few seconds and they need to keep the phone awake manually.

### Background Tab Resources
To remain fast with many tabs, browsers have to restrict the resource usage of background tabs. There are browsers today which run background tabs with a low CPU/network priority or with timer throttling. To further improve the speed of what’s in the foreground, browsers could freeze background tabs (as defined in the [Page Lifecycle API](https://wicg.github.io/page-lifecycle/spec.html)). This is challenging to do without breaking important use cases, such as uploading large files to the cloud or processing data from a background tab. The progress notification API could be used by background tabs to indicate when they are doing important work for the user. The browser could take that as a hint to not restrict their resource usage (no freezing, but also no CPU/network priority downgrading and no timer throttling). To avoid abuse of the progress notification API, browsers should surface to the user which tabs use it and let the user decide if they want their resource usage to be restricted anyways.

### Key Scenarios
These use cases aim to help with the following class of operations that sites would not want interrupted, even if the tab is occluded.

* Downloading offline videos
* Uploading large files
* Video or image processing
* Long payment transactions on shopping sites

## Non-goals
* Extending the lifetime of a service worker when important work shouldn’t be interrupted by document closure is deferred to [Phase 2](/sw-explainer.md)
* Extending the lifetime of a service worker when important work is triggered by a background event is deferred to [Phase 2](/sw-explainer.md)


## Creating a Progress Notification for a Task
This explainer introduces a new `ProgressNotification` interface that can be initialized with a title and options to specify progress type. Browsers may use these inputs to surface details of the ongoing task in the UI when the site is not visible.

The `update` method is for informing the browser when progress has been made, and to update the UI. This will take in a progress parameter ranging from [1-100] and `timeRemaining` value in seconds.

The `close` method is for informing the browser when a task has completed, and can be used by the browser to hide or update the UI as complete. 

```js
const progress = new ProgressNotification("Processing image...",
                                          { type: 'determinate' });
while (!task.complete()) {
  task.step();
  progress.update(task.calculateProgress());
}
progress.close();
```

## WebIDL
```js
enum ProgressNotificationType {
  determinate,
  indeterminate
};

dictionary ProgressNotificationOptions {
  required ProgressNotificationType type;
};

[Exposed=(Window,Worker), SecureContext]
interface ProgressNotification {
  constructor(DOMString title, ProgressNotificationOptions options);

  undefined update(double progress, optional double timeRemaining);
  undefined close();
};
```

## UI Design Considerations
Browser implementations should display the following when a tab running a background task is occluded
* Origin of the site
* Button to abort background work
* Option to never allow background work

Other UI considerations
* Provided title
* Progress bar or animation for showing progress 
* On task completion, user agents may choose to hide the task UI, or mark the UI as complete keeping it persisted for users to see

## Security Considerations
This proposal creates a UI surface under developer control. Implementations must be careful to ensure that the developer-specified message is presented with context to prevent it from being confused with browser UI.

Since this proposal is limited to only allow a document to continue executing scripts in cases where they would otherwise be throttled or the system put to sleep, there should be no additional security issues. 

## Privacy Considerations
Use cases above involve allowing a site to continue doing work in situations where currently they cannot. This has privacy implications because the execution of script itself can leak information such as the user’s current location through network accesses. The Progress Notification UI can include a button to stop the action and give the user the option to prevent the site from doing background work in the future. This keeps the user in control.  

## Considered alternatives
One alternative to a general progress notification API is to add this capability to individual APIs. There are a number of cases where browsers do that. For example, the browser may hold a system wake lock when playing audio or a screen wake lock while playing video in order to match user expectations about how their device should behave when performing those functions. As we found with the Screen Wake Lock API however there are cases where the browser can’t guess about that user expectation. For example, when following a recipe you don’t want your screen to turn off while your hands are too dirty to touch the device. There isn’t any one particular thing the site is doing that would indicate that user expectation but it is something the developer understands. An explicit API to tell the browser that the site is doing something which the user might want to continue to see on the screen solves this problem. For this proposal, the background tab throttling use case provides the strongest signal that the ability for a developer to tell the browser that there is a user visible reason why script should be running is useful.

## References & Acknowledgements
Many thanks to valuable feedback and advice from:
* François Doray
* Vincent Scheib
* Austin Sullivan
