# push-notificification
/**
 * checks if Push notification and service workers are supported by your browser
 */
function isPushNotificationSupported() {
  return "serviceWorker" in navigator && "PushManager" in window;
}

/**
 * asks user consent to receive push notifications and returns the response of the user, one of granted, default, denied
 */
function initializePushNotifications() {
  // request user grant to show notification
  return Notification.requestPermission(function(result) {
    return result;
  });
}
/**
 * shows a notification
 */
function sendNotification() {
  const img = "/images/jason-leung-HM6TMmevbZQ-unsplash.jpg";
  const text = "Take a look at this brand new t-shirt!";
  const title = "New Product Available";
  const options = {
    body: text,
    icon: "/images/jason-leung-HM6TMmevbZQ-unsplash.jpg",
    vibrate: [200, 100, 200],
    tag: "new-product",
    image: img,
    badge: "https://spyna.it/icons/android-icon-192x192.png",
    actions: [{ action: "Detail", title: "View", icon: "https://via.placeholder.com/128/ff0000" }]
  };
  navigator.serviceWorker.ready.then(function(serviceWorker) {
    serviceWorker.showNotification(title, options);
  });
}

/**
 * 
 */
function registerServiceWorker() {
  navigator.serviceWorker.register("/sw.js").then(function(swRegistration) {
    //you can do something with the service wrker registration (swRegistration)
  });
}

export {
  isPushNotificationSupported,
  initializePushNotifications,
  registerServiceWorker,
  sendNotification
};
This script exports all the functions described above. To use these function we can use a script like this: (This is just a sample, I’ll share the full code at the end of the article)
import {
 isPushNotificationSupported,
 sendNotification,
 initializePushNotifications,
 registerServiceWorker
} from "./push-notifications.js";
const pushNotificationSuported = isPushNotificationSupported();
if (pushNotificationSuported) {
registerServiceWorker();
  initializePushNotifications().then(function(consent){
    if(consent === 'granted') {
     sendNotification();
    }
  });
}

Next, create the service worker file: sw.js. It will be, just for now, an empty file.
Push
As stated in the beginning, you need a server to send notifications to the user. We will call it the Push server.
Imagine this scenario: an e-commerce website wants to send a notification to users when a new product is released. These are the steps required:
The service worker creates a push notifications subscription.
This subscription creates a unique endpoint for the application/service worker/user. This endpoint is the one the push server will use to send push messages to.
The application sends the endpoint to the push server.
The push server saves the subscription of the user.
When a new product is released, the push server sends a push message/notification to the saved subscription endpoint.
The service worker registered by the application receives the notification and shows it to the user.
Create the push notification subscription.
to create a push notification subscription you use the method serviceWorker.pushManager.subscribe() (described here) that takes a parameter that represents the options. the properties of these options are:
userVisibleOnly: a boolean indicating that the returned push subscription will only be used for messages whose effect is made visible to the user.
applicationServerKey: an ECDSA (Elliptic Curve Digital Signature Algorithm) P-256 public key the push server will use to authenticate your application. Don’t worry, I’ll show you how to create it later on.

To access the service worker the safest method navigator.serviceWorker.ready that returns a promise that resolves the service worker registration. The promise is reloved when the service worker is ready. If we access the service worker in other ways, it could be not ready (maybe installing, or waiting).
Send the subscription to the server.
To send the subscription to the server there is no standard way, the important things to remember are:
The communication between the app and the push server must be secure because the subscription information is what is needed to send the notification. If stolen can be used to send unwanted notifications on behalf of the app.
when possible the subscription should be associated with a specific user, to send only relevant notifications.
This could be achieved with a simple fetch or with the Http client of the app when present. The object to send is a PushSubscription. Something like this
fetch(’push-server/user/{id}/push-subscription’, {
headers: { "content-type": "application/json;charset=UTF-8", "sec-fetch-mode": "cors" , <AUTH HEADERS>},
body: JSON.stringify(subscription),
method: "POST",
mode: "cors"
})
The push server that receives this call should validate and save the request, this example made with express and NodeJS saves the subscription on a map in memory. In production you want to save the subscription more persistently, such as a Database.
function handlePushNotificationSubscription(req, res) {
  const subscriptionRequest = req.body;
  const susbscriptionId = createHash(JSON.stringify(subscriptionRequest));
  subscriptions[susbscriptionId] = subscriptionRequest;
  res.status(201).json({ id: susbscriptionId });
}
app.post("/subscription", handlePushNotificationSubscription);
The full code is available here: https://github.com/Spyna/push-notification-demo/blob/master/back-end/src/subscriptionHandler.js#L18
Send the push notification
This is the work of the push server. There are some library and framework that implements a push server, I’m going to show you an example made in JavaScript with NodeJS. The code below uses a library called web-push
const webpush = require("web-push");
// VAPID keys should only be generated only once.
// const vapidKeys = webpush.generateVAPIDKeys();
const vapidKeys = {
  privateKey: "bdSiNzUhUP6piAxLH-tW88zfBlWWveIx0dAsDO66aVU",
  publicKey: "BIN2Jc5Vmkmy-S3AUrcMlpKxJpLeVRAfu9WBqUbJ70SJOCWGCGXKY-Xzyh7HDr6KbRDGYHjqZ06OcS3BjD7uAm8"
};
webpush.setVapidDetails("example@yourdomain.org", vapidKeys.publicKey, vapidKeys.privateKey);
webpush.sendNotification(pushSubscription, "The text of the notification")
Vapid means Voluntary Application Server Identification and is a web standard documented here.
The VAPID public key is the one shared with the web app ( the one which sends the notification subscription).
The method sendNotification accepts two parameters:
pushSubscription: a PushSubscription object, that is the one obtained from the service worker by the method: serviceWorker.pushManager.subscribe() as said earlier.
payload: the content of the push notification, that must be a string or a buffer. The example is a simple text, but it could be something more interesting like information about the notification to display or a reference to an API to call, in short, whatever you want to be inside the notification.
const payload = JSON.stringify({
        title: "New Product Available ",
        text: "HEY! Take a look at this brand new t-shirt!",
        image: "/images/jason-leung-HM6TMmevbZQ-unsplash.jpg",
        tag: "new-product",
        url: "/new-product-jason-leung-HM6TMmevbZQ-unsplash.html"
      });
const payload = `/product/<product-id>`;
Receive the pushed notification
I said earlier that you need a service worker to send the notification. Forget it! At this point, the push messages are sent by the push server.
You still need a service worker to receive push messages. You already have a service worker registered, but it does nothing. So we are going to add a listener to receive the push event:

In this script, we added a listener to the event push in the service worker. The event is an object of type PushMessageData containing the data sent from the push server and can be accessed as String, Object, and some others format using the methods event.data.text(), event.data.json() or blob().
Once you read the data from the notification event you can display the notification. Outside the service worker you called earlier the method: serviceWorkerRegistration.showNotification. Inside the service worker, you use self.registration.showNotification().
Click on the notification
The notification could be self explicative and require no action, but usually, we want something to happen when the user clicks (or taps) the notification. You need to add a listener in the service worker for the event notificationclick

This snippet is part of the demo project, full code is available at https://github.com/Spyna/push-notification-demo/blob/master/front-end/public/sw.js#L19
In this code snippet, you added an event listener, that reads the event.notification.data property, and does something with it. In this example, the data is a URL (because we created as a URL in the example before). Next, we open that URL in the browser window. Of course, you can do pretty much anything you want, playing around with the content of the notification and the service worker.
For example, if the user has already a notification you can avoid to add another one, but modify the existing with something like you have N messages to read.
Or if you consider these actions in the notification:
self.registration.showNotification("Your friend has posted a video", {
  actions: [
   {action: 'like', title: 'Like'},
   {action: 'reply', title: ' Reply'}]
});
You can take different decision based on the action type, which you can access in the property action of the event.
function handleNotification(event) {
  event.notification.close();
  if (event.action === 'like') {
   // silently like the video;
  }
  else if (event.action === 'reply') {
   // open the reply page  }
  else {
    // do something else
  }
}}
self.addEventListener('notificationclick', handleNotificaiton);
