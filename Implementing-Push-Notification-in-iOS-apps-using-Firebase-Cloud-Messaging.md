This article shows example codes in both iOS side and server side to let you know how to make use of Firebase Cloud Messaging (FCM) to implement push notification for your iOS app.

You need to use the [Firebase](https://firebase.google.com/) for your iOS app.

# Not in this article

I will not go into detail on how to setup in Xcode side nor the certification issues. There are tons of articles on the web for those. The steps for this part are same for any platforms.

# Basic concept about iOS push notification

Almost every iOS app needs push notification. And you don’t want to deal with [complicated details](https://developer.apple.com/design/human-interface-guidelines/ios/system-capabilities/notifications/).

Here is the basic components you need to know.

1. Your iOS app. This faces to human users.
2. The Apple push notification service. This service, or server, belongs to Apple. It sends the notification to every iOS device upon it receives requests from your application server.
3. A server application. This server, or a server application, will send requests to the Apple push notification service when it wants to notify users.

If you want to implement the server side logic, it is possible. However, for most of us, we want to get rid of the trouble and here the FCM comes.

# Setup in Firebase

We need to do these steps.

1. Firebase project setup. This is to put the key file of your project to the Firebase project setup for cloud messaging. So the Apple server will recognize the FCM server and allow it talks to the Apple server on behalf of your app.
2. Firebase functions. To send notifications you need backend logic. Firebase functions is the place you write your backend logic for FCM. To write this code you need to setup the Google Cloud command line tool. Then you write at your local computer and use the tool to deploy it to the Firebase server. Note that you cannot view the code on Firebase console. All you can review is to your source code. So please do some backend or version control for your source.

# iOS App setup

On the iOS app, we have several ways to receive messages from the FCM. In this case, for the simplicity, we choose “subscription by topic” approach. The Topic is actually a string, and our app subscribe it via a call:

```
Messaging.messaging().subscribe(toTopic: topicString)
```

Please note that the topic string can only accept certain characters as shown in this regular expression:

```
[a-zA-Z0-9-_.~%]+
```

# Backend logic example

Let me use an example to show how we trigger the push notification.

In this example project, it uses the Realtime database of Firebase. This is a JSON-like structure database and we refer data using path. In this use case, we want to send out a push notification whenever there is a new node (data object) is inserted to the node “/messages”. This is actually a new message is sent by some user to another.

```javascript
const functions = require("firebase-functions");
const admin = require("firebase-admin");
admin.initializeApp();
exports.sendPush = functions.database.ref('/messages/{pushId}')
.onCreate((snapshot, context) => {
const msgid = context.params.pushId;
const msg = snapshot.val();
const peer = msg.peer ;
const room = msg.chatroomid;
const text = msg.text ;
const sender = msg.userid;
const sendername = msg.username ;
// just in case. old version of messages may not have this field.
if (!peer) {
  return 0;
}
// room id has colon, but it is not valid for firebase topic name.
// this matches what the app subscribes for each room.
const topic = peer + "-" + room.replace(":","-");
const message = {
  notification: {
    title: sendername,
    body: text
  },
  data: {
    to: peer ,
    sender: sender ,
    room: room,
    sendername: sendername,
    text: text
  },
  topic: topic
};
admin.messaging().send(message)
.then((response) => {
  // Response is a message ID string.
  functions.logger.log('Successfully sent message:', response);
})
.catch((error) => {
  functions.logger.log('Error sending message:', error);
});
});
```

As you see, the key point here is to construct a JSON structure to represent your message to send to your iOS app. Please look at the message variable. It consists of 3 parts.

The `notification`: This is what displays in the push notification banner on the iOS device.

The `data`: This is the data you want to send to the iOS app to process with. The name of the fields seem could be any string but please try to avoid using strange ones and be simple, and avoid symbols.

The `topic`: This is the FCM topic string that your iOS app subscribes to.

# Conclusion

We hope this is helpful. Please let me know in comments that you have any doubts. We try to make this writing short. So you would still need to check Firebase documentations to know how it really works.
