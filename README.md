# Introduction
When I first was approached to create an application for an event involving ann interactive quiz I immediatly got excited because it sounded both cool and challenging.

The idea evolved to a hybrid app were attendees to the event could see the agenda of an event and participate in an interactive quiz.

After the scoping session I had a pretty clear idea, we would need:

* **A client app** (hybrid both because of the timeframe 1 week maximum and also because the UI didn't seem to exigent) 
* **A dashboard web app** to show the quiz results live and historic data, 
* **Some short of repository** for the event agenda, quiz definition, etc.
* **A user repository** and user/pass authentication
* **An interactive protocol to deal with quiz questions and answers**
* **And a place to run these apps API**: so far including some REST APIs, getting the events for the current date, getting the quiz questions, storing attendees answers, etc.

All in one week... nice.

At least I had a couple of secret weapons, I had access to a Red Hat Mobile Application Platform (RHMAP for short) and a lot of enthusiasm.

# Selecting technology

Regarding the client app I was clear, Ionic 2, cleaner code (thanks to Angular 2/Typescript) and I had been using Ionic 1 for quite some time successfully on RHMAP. What could go wrong?

Some sort of interactive protocol... hmmm, that sounded pretty much like websockets. I had run some examples using socket.io and I knew some other redhatters had used socket.io on RHMAP. So, socket.io.

The dashboard... why not Angular 2+ (again cleaner and object oriented) plus Bootstrap 4? Maybe a bit bold here... Bootstrap 4 is still alpha... but I had used it before with no issues. So I took my chances.

Security, storage (for events, agenda, quiz, results, users, etc.) and running the APIs, all covered by RHMAP so I just have to care about the business logic of my apps.

# Implementation details
The final result of thes experience can be summarized as follows.

* 1 x Hybrid Client App (Ionic 2) [code](https://github.com/cvicens/Redhatters-Client-App)
* 1 x Responsive DashBoard to show Quiz results live/historic [code](https://github.com/cvicens/Redhatters-Dashboard-App)
* 1 x Cloud App (featuring socket.io set up for the interactiveQuiz) [code](https://github.com/cvicens/Redhatters-Cloud-App)
* 1 x Authentication MBaaS Service [code](https://github.com/cvicens/Redhatters-Auth)

## The Client App

To accelerate the development on RHMAP we usually start developing apps from a template, in this occasion the template had to be Ionic 2 based so I used [this](https://github.com/cvicens/quickstart-ionic-v2-tabs).

*Beware that in order to use Ionic 2 you need Node.js 6. Please go [here](https://ionicframework.com/docs/intro/installation/) for an explanation about the minimum requirements*

You might be wondering what's added to the base Ionic 2 tab template. Well, let me explain the main differences below, but basiclly changes were as follows:

1. Adding RHMAP SDK
2. Adding a script to copy RHMAP configuration file to ``www``
3. Modifying ``.gitignore`` to version control ``www``

### Adding RHMAP SDK

Because we'll need RHMAP SDK for Javascript, I installed ``fh-js-sdk`` package by executing ``npm install fh-js-sdk --save``. So in the ``dependencies`` area of ``package.json`` you'll see this entry ``"fh-js-sdk": "^2.18.4"``, as in:

```
  "dependencies": {
    "@angular/common": "2.4.8",
	 ...
    "fh-js-sdk": "^2.18.4",
    ...
  }
```

**But... wait... is it possible to use RHMAP SDK within Typescript?**
Indeed! You can even use proper typing! 
To test this just open your favorite IDE (like atom or VS Code) and Ctr+Space after the variable where you have imported RHMAP SDK. For instance in the following snippet the variable would be ``$fh``.

```
import * as $fh from 'fh-js-sdk';

$fh. <== Ctrl+space
```


Next snippet is part of the App source code and shows how to use it.

```
import * as $fh from 'fh-js-sdk';
...
$fh.cloud(
    params, 
    (data) => {
      resolve(data);
    }, 
    (msg, err) => {
      // An error occurred during the cloud call.
      console.log('Cloud call failed with error message:' + msg + '. Error properties:' + JSON.stringify(err));
      reject(msg);
    });
...

```

### Adding a script to copy RHMAP configuration file to www

Now, Ionic 2 rests on Angular 2 so Typescript needs to be compiled and ``www`` folder is generated... and assets copied if necessary. This is the case of our RHMAP configuration file ``fhconfig.json``. To copy this file to the ``www`` folder we added a configuration script ``copy.custom.config.js`` in the folder ``./scripts`` which is a copy of the original one but adding the following entry ``copyFHConfig ``

```
module.exports = {
  ...
  copyFHConfig: {
    src: ['{{SRC}}/fhconfig.json'],
    dest: '{{WWW}}'
  },
  ...
}
```

In order to allow **ionic cli** execute our new script we modified ``package.json`` to include the following section.

```
  "config": {
    "ionic_copy": "./scripts/copy.custom.config.js"    
  }
```

### Modifying .gitignore to version control www

Even though ``www`` folder is self-genetared and in general in an Ionic 2 project (and even more general in an Angular 2+ project) we shouldn't add it to the git repo we'll added to allow the App being compiled in [RHMAP build farm](https://access.redhat.com/products/red-hat-mobile-application-platform).

So I modified our well known friend ``.gitignore`` to uncomment ``www`` as in the following excerpt.

```
...
plugins/
plugins/android.json
plugins/ios.json
#www/ <-- commented out
$RECYCLE.BIN/
...
```

## The Dashboard App

In this case I started from the scratch using angular cli ``ng``, this time no template was used. I'd recommend you following this [guide](https://angular.io/guide/quickstart) to get started with Angular 2+. There you'll learn how to install angular cli ``ng`` which you'll need later.

Again it's worth noting what changes I had to apply to be able to run this web application in RHMAP.

This is the list of changes, details can be found on the next sections:

1. Installing RHMAP JS SDK ``fh-js-sdk``
2. Adapting the web application to run on Express.js
3. Modifying ``.angular-cli.json`` file to allow copying RHMAP configuration file to ``www``
3. Modify ``.gitignore`` to version control ``www``

### Installing RHMAP ``fh-js-sdk``

Once the app was created with ``ng new <app-name>`` I added our RHMAP SDK ``fh-js-sdk`` by executing ``npm install fh-js-sdk --save``

### Adapting the web application to run on Express.js

Before explaining how I adapted my angular app to run on Express.js let me give a bit of context for those not very familiar with angular. Angular is a JS framework meant to run in a browser, while you develop and test this kind of apps you usually run the app in a local http server and you point your browser to this http server. Specifically in the case of angular 2+ you do this by running ``ng serve``.

```
$ git clone https://github.com/cvicens/Redhatters-Dashboard-App.git
...
$ npm install
...
│   └── xdg-basedir@3.0.0 
├── typescript@2.2.2 
└── zone.js@0.8.12 
$ ng serve
** NG Live Development Server is running on http://localhost:4200 **
Hash: 2e0531a8c67bba2e1191                                                               
Time: 11869ms
...
webpack: Compiled successfully. 
```

As you can see above when you run ``ng serve`` the angular cli compiles and starts an http server process serving the app. To use the app just point your browser [here](http://localhost:4200)

This is perfect to develop, test, debug... but in order to deploy the app in RHMAP I needed to make a some of changes, namely:
* Install packages; express and corser
* Moving all the other packages from dependencies to devDependencies
* Add a new file, ``application.js`` to setup the Express.js to serve the files that make the angular app

RHMAP will run this application using plain ``node`` this means we have to 1st compile the Typescript source and 2nd adapt it to use Express.js. The idea is being able to start up the app using plain old ``node application.js`` as any other RHMAP web/cloud app.

Let's see how ``application.js`` looks:

```
var express = require('express');
var corser = require("corser");

var app = express();
app.use(corser.create());

// The compiled version is located in /www so expose it
app.use(express.static(__dirname + '/www'));

app.options("*", function (req, res) {
  // CORS
  res.writeHead(204);
  res.end();
});

// Used for App health checking
app.get('/sys/info/ping', function(req, res, next) {
  res.end('"OK"');
});

var port = process.env.FH_PORT || process.env.OPENSHIFT_NODEJS_PORT || 8111;
var host = process.env.OPENSHIFT_NODEJS_IP || '0.0.0.0';
var server = app.listen(port, host, function() {
  console.log("App started at: " + new Date() + " on port: " + port);
});
module.exports = server;
```

This file basicly sets up and run an Express.js app exposing the files of our angular app, listening in port ``8111`` if run locally, and at the ports defined by environments variables FH_PORT or OPENSHIFT_NODEJS_PORT when run inside RHMAP. As the compiled code ends up in ``www`` we have to expose that folder as static files.


### Modify ``.gitignore`` to version control ``www``

We just explained why we need the ``application.js`` file and why we need to expose folder ``www`` as static contents using Express.js, what we haven't explained is why we can't ignore ``www`` in our git repository.

The reason is simple, as of today the highest version of Node.js we can use in RHMAP is 4.4.3 and in order to compile (or [transpile?](https://www.stevefenton.co.uk/2012/11/compiling-vs-transpiling/)) we need 6.10.x, so instead of compiling in RHMAP as a previous step to actually deploying and running the code we have to compile locally and commit/push all, including the ``www`` folder.



```
{
  "name": "ionic-app-base",
  "author": "Ionic Framework",
  "homepage": "http://ionicframework.com/",
  "private": true,
  "scripts": {
    "clean": "ionic-app-scripts clean",
    "build": "ionic-app-scripts build",
    "ionic:build": "ionic-app-scripts build",
    "ionic:serve": "ionic-app-scripts serve"
  },
  "config": {
    "ionic_copy": "./scripts/copy.custom.config.js"    
  },
  "dependencies": {
    "@angular/common": "2.4.8",
    "@angular/compiler": "2.4.8",
    "@angular/compiler-cli": "2.4.8",
    "@angular/core": "2.4.8",
    "@angular/forms": "2.4.8",
    "@angular/http": "2.4.8",
    "@angular/platform-browser": "2.4.8",
    "@angular/platform-browser-dynamic": "2.4.8",
    "@angular/platform-server": "2.4.8",
    "@ionic-native/core": "3.1.0",
    "@ionic-native/splash-screen": "3.1.0",
    "@ionic-native/status-bar": "3.1.0",
    "@ionic/storage": "2.0.0",
    "fh-js-sdk": "^2.18.4",
    "ionic-angular": "2.2.0",
    "ionicons": "3.0.0",
    "rxjs": "5.0.1",
    "sw-toolbox": "3.4.0",
    "zone.js": "0.7.2"
  },
  "devDependencies": {
    "@ionic/app-scripts": "1.1.4",
    "typescript": "2.0.9"
  }
}
```


Links:



## Project complete!

## Results





```
{
    "id": "0015",
    "title": "Country Meeting FY17Q2 Kick-Off",
    "quizId": "0002",
    "address": "Paseo de la Castellana 259C",
    "city": "MADRID",
    "province": "MADRID",
    "country": "SPAIN",
    "date": "2017-06-19",
    "startTime": "10:00",
    "endTime": "21:00",
    "hashtag": "#madrid17Q1",
    "agenda": [
        {
            "day": 1,
            "segments": [
                {
                    "slot": "9:30am",
                    "sessions": [
                        {
                            "day": 1,
                            "title": "Recogida en las oficinas de Red Hat",
                            "slug": "day-1-pick-up",
                            "location": "Red Hat Office, MAdrid",
                            "startTime": "2017-06-09T07:30:00.000Z",
                            "endTime": "2017-06-09T08:00:00.000",
                            "hasDetails": false,
                            "onMySchedule": false,
                            "allDay": false,
                            "objectId": "xeBniOY5Qb",
                            "sortTime": "1497000600",
                            "displayTime": "9:30am"
                        }
                    ]
                },
                {
                    "slot": "10:00am",
                    "sessions": [
                        {
                            "day": 1,
                            "title": "Café de bienvenida",
                            "slug": "day-1-breakfast",
                            "location": "Hall, Las Rozas",
                            "startTime": "2017-06-09T08:00:00.000Z",
                            "endTime": "2017-06-09T08:30:00.000Z",
                            "hasDetails": false,
                            "onMySchedule": false,
                            "allDay": false,
                            "objectId": "TCICVoDUMX",
                            "sortTime": "1497002400",
                            "displayTime": "10:00am"
                        }
                    ]
                },
                {
                    "slot": "10:30am",
                    "sessions": [
                        {
                            "day": 1,
                            "title": "Country Meeting - Sala de conferencias",
                            "slug": "day-1-contry-meeting",
                            "location": "Sala conferencias, Las Rozas",
                            "startTime": "2017-06-09T08:30:00.000Z",
                            "endTime": "2017-06-09T10:30:00.000Z",
                            "hasDetails": false,
                            "onMySchedule": false,
                            "allDay": false,
                            "objectId": "TCICVoDUMX",
                            "sortTime": "1497004200",
                            "displayTime": "10:30am"
                        }
                    ]
                },
                {
                    "slot": "12:30am",
                    "sessions": [
                        {
                            "day": 1,
                            "title": "Preparación Técnica - Actividad en equipo",
                            "slug": "day-1-team-building-prep",
                            "location": "Sala conferencias, Las Rozas",
                            "startTime": "2017-06-09T10:30:00.000Z",
                            "endTime": "2017-06-09T11:00:00.000Z",
                            "hasDetails": false,
                            "onMySchedule": false,
                            "allDay": false,
                            "objectId": "TCICVoDUMX",
                            "sortTime": "1497011400",
                            "displayTime": "12:30am"
                        }
                    ]
                },
                {
                    "slot": "01:00pm",
                    "sessions": [
                        {
                            "day": 1,
                            "title": "Open Bar & Actividad de equipo",
                            "slug": "day-1-team-building-activity",
                            "location": "Túnel del viento",
                            "startTime": "2017-06-09T11:00:00.000Z",
                            "endTime": "2017-06-09T13:00:00.000Z",
                            "hasDetails": false,
                            "onMySchedule": false,
                            "allDay": false,
                            "objectId": "TCICVoDUMX",
                            "sortTime": "1497013200",
                            "displayTime": "01:00pm"
                        }
                    ]
                },
                {
                    "slot": "03:00pm",
                    "sessions": [
                        {
                            "day": 1,
                            "title": "Almuerzo - Cocktail",
                            "slug": "day-1-lunch",
                            "location": "Bar",
                            "startTime": "2017-06-09T13:00:00.000Z",
                            "endTime": "2017-06-09T15:00:00.000Z",
                            "hasDetails": false,
                            "onMySchedule": false,
                            "allDay": false,
                            "objectId": "TCICVoDUMX",
                            "sortTime": "1497020400",
                            "displayTime": "03:00pm"
                        }
                    ]
                },
                {
                    "slot": "05:00pm",
                    "sessions": [
                        {
                            "day": 1,
                            "title": "Regreso a las oficinas de Red Hat",
                            "slug": "day-1-return",
                            "location": "Bus Station",
                            "startTime": "2017-06-09T15:00:00.000Z",
                            "endTime": "2017-06-09T15:30:00.000Z",
                            "hasDetails": false,
                            "onMySchedule": false,
                            "allDay": false,
                            "objectId": "TCICVoDUMX",
                            "sortTime": "1497027600",
                            "displayTime": "05:00pm"
                        }
                    ]
                }
            ]
        }
    ],
    "_id": "5947a28cd168f54b3406989a"
}
```