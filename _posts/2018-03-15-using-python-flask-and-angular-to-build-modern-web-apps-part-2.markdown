---
layout: post
title: "Using Python, Flask, and Angular to Build Modern Web Apps - Part 2"
description: "In this series, you will learn how to create modern web applications with Python, Flask, and Angular."
longdescription: "In this series, you will learn how to create modern web applications with Python, Flask, and Angular. You will create a SPA and a backend API to expose exams and questions so users can test their knowledge regarding different technologies."
date: 2018-03-15 08:30
category: Technical Guide, Python
author:
  name: "Bruno Krebs"
  url: "https://twitter.com/brunoskrebs"
  mail: "bruno.krebs@gmail.com"
  avatar: "https://twitter.com/brunoskrebs/profile_image?size=original"
design:
  bg_color: "#4A4A4A"
  image: https://cdn.auth0.com/blog/python-flask-angular/logo.png
tags:
- python
- flask
- sqlalchemy
- angular
- typescript
- auth0
related:
- 2017-09-28-developing-restful-apis-with-python-and-flask
- 2017-11-09-sqlalchemy-orm-tutorial-for-python-developers
- 2018-03-13-using-python-flask-and-angular-to-build-modern-apps-part-1
---

**TL;DR:** In this series, you will learn how to create modern web applications with Python, Flask, and Angular. You will use this stack to build a SPA and a backend API to expose exams and questions so users can test their knowledge regarding different technologies. [In this GitHub repository](https://github.com/auth0-blog/online-exam/), you can find the final code created throughout the first part of the series.

---

## What You Will Build

In this series, you will use Python, Flask, and Angular to build a web application based on a modern architecture. With Angular, you will build a SPA (Single Page App) that allows users to browse through exams and questions. These users, when authenticated, will be able to test their knowledge regarding a specific topic by choosing one of the multiple choices that a question exposes. Then, when your users submit their answers, your backend will check if they are right or wrong, record the result, and send back this result to users.

In this part of the series, you will start by configuring Auth0 as the identity management system of your app. You will configure an Auth0 API to represent and secure your Flask application and you will configure an Auth0 Client to represent and secure your Angular SPA. After securing your app with Auth0, you are going to enhance the application to allow users to test their knowledge.

{% include tweet_quote.html quote_text="I'm building modern webapps with Angular, Flask, and Python!" %}

## Managing Identity with Auth0

Instead of investing time to develop rudimentary authentication mechanisms to manage the identity of your users, you are going to use Auth0. For startup projects like this one, [the free tier provided by Auth0](https://auth0.com/pricing) is more than enough. Besides being free, by choosing Auth0, you will get a modern, easy to use, and reliable service capable of integrating with tons of different social identity providers (e.g. Facebook, Google, Twitter, etc). Also, if you ever need to integrate with enterprise identity providers using protocols like [OpenID Connect](https://auth0.com/docs/protocols/oidc), [SAML](https://auth0.com/docs/protocols/saml), and [WS-Federation](https://auth0.com/docs/protocols/ws-fed), don't worry, Auth0's got you covered.

That is, Auth0 can help you focus on what matters the most to you, the special features of your product. In addition, Auth0 can improve your product's security with state-of-the-art features like [passwordless](https://auth0.com/passwordless), [breached password surveillance](https://auth0.com/breached-passwords), and [multifactor authentication](https://auth0.com/multifactor-authentication).

## Securing Flask Apps with Auth0

To integrate Auth0 into your Flask application, you will need to create an Auth0 API. If you haven't done so yet, you can <a href="https://auth0.com/signup" data-amp-replace="CLIENT_ID" data-amp-addparams="anonId=CLIENT_ID(cid-scope-cookie-fallback-name)">sign up for a free Auth0 account now</a>. After creating your account, head to [the APIs page on your Auth0 dashboard](https://manage.auth0.com/#/apis) and click on the *Create API* button. When clicked, this button will bring up a form where Auth0 will ask you for three properties. The following list summarizes these properties and how to fill them:

1. *Name*: This is a friendly name to remind you what this API is about. Here, you can enter something like "Online Exam".
2. *Identifier*: This is the logical identifier (a.k.a. audience) of your API. Usually, developers use an URL to represent their APIs. Although this is not mandatory, it is a good approach. So, in this field, enter something like `https://online-exam.digituz.com.br`.
3. *Signing Algorithm*: In this field, you will have to select between two strategies: `RS256` (the default one) and `HS256`. The best option is to stick with the default one (`RS256`). If you are curious, you can [check this thread to understand the difference between them](https://community.auth0.com/answers/6945/view).

When you finish filling out this form, you can hit the *Create* button. This will redirect you to a tab called *Quick Start* inside your new Auth0 API. From there, select the *Scopes* tab and add a new scope called `manage:exams` ([as a good practice, always define scopes when dealing with the OAuth 2.0 authorization mechanism](https://auth0.com/docs/scopes/current)). You can add a simple description like *Manage exams* to it. After that, you can leave this page open and head back to your Flask project (i.e. to the `backend` directory inside the project root).

Back in your project, you will need to create a [Python decorator](https://realpython.com/blog/python/primer-on-python-decorators/) to wrap the endpoints that must be secured. To define this decorator, create a new file called `auth.py` inside the `./backend/src/` directory. Then, inside this file, paste the following code:

```python
import json
from flask import request, _request_ctx_stack
from functools import wraps
from jose import jwt
from urllib.request import urlopen


AUTH0_DOMAIN = 'bk-samples.auth0.com'
ALGORITHMS = ['RS256']
API_AUDIENCE = 'https://online-exam.digituz.com.br'


class AuthError(Exception):
    def __init__(self, error, status_code):
        self.error = error
        self.status_code = status_code


def get_token_auth_header():
    """Obtains the Access Token from the Authorization Header
    """
    auth = request.headers.get('Authorization', None)
    if not auth:
        raise AuthError({
            'code': 'authorization_header_missing',
            'description': 'Authorization header is expected.'
        }, 401)

    parts = auth.split()

    if parts[0].lower() != 'bearer':
        raise AuthError({
            'code': 'invalid_header',
            'description': 'Authorization header must start with "Bearer".'
        }, 401)

    elif len(parts) == 1:
        raise AuthError({
            'code': 'invalid_header',
            'description': 'Token not found.'
        }, 401)

    elif len(parts) > 2:
        raise AuthError({
            'code': 'invalid_header',
            'description': 'Authorization header must be bearer token.'
        }, 401)

    token = parts[1]
    return token


def requires_auth(f):
    """Determines if the Access Token is valid
    """

    @wraps(f)
    def decorated(*args, **kwargs):
        token = get_token_auth_header()
        jsonurl = urlopen(f'https://{AUTH0_DOMAIN}/.well-known/jwks.json')
        jwks = json.loads(jsonurl.read())
        unverified_header = jwt.get_unverified_header(token)
        rsa_key = {}
        for key in jwks['keys']:
            if key['kid'] == unverified_header['kid']:
                rsa_key = {
                    'kty': key['kty'],
                    'kid': key['kid'],
                    'use': key['use'],
                    'n': key['n'],
                    'e': key['e']
                }
        if rsa_key:
            try:
                payload = jwt.decode(
                    token,
                    rsa_key,
                    algorithms=ALGORITHMS,
                    audience=API_AUDIENCE,
                    issuer='https://' + AUTH0_DOMAIN + '/'
                )

            except jwt.ExpiredSignatureError:
                raise AuthError({
                    'code': 'token_expired',
                    'description': 'Token expired.'
                }, 401)

            except jwt.JWTClaimsError:
                raise AuthError({
                    'code': 'invalid_claims',
                    'description': 'Incorrect claims. Please, check the audience and issuer.'
                }, 401)
            except Exception:
                raise AuthError({
                    'code': 'invalid_header',
                    'description': 'Unable to parse authentication token.'
                }, 400)

            _request_ctx_stack.top.current_user = payload
            return f(*args, **kwargs)

        raise AuthError({
            'code': 'invalid_header',
            'description': 'Unable to find the appropriate key.'
        }, 400)

    return decorated
```

Although lengthy, this code is quite simple. Here is a list to explain what this new module does:

- First, it defines three constants: `AUTH0_DOMAIN`, `ALGORITHMS`, and `API_AUDIENCE`. Your project will use these constants to communicate with Auth0 to validate users (tokens). **Note that** you will have to replace the value of these constants with the details of your Auth0 account.
- Second, it defines a class called `AuthError`. This class exists to represent errors originated in this module.
- Third, it defines a function called `get_token_auth_header`. Your app will use this function to read `Authorization` headers to fetch their [access tokens](https://auth0.com/docs/tokens/access-token).
- Lastly, this module defines the `requires_auth` decorator. This decorator might look complex but, if you analyze carefully, you will see that all it does is to fetch the correct public key from Auth0 to validate tokens. Instead of sharing static public keys, Auth0 uses the JWK specification to represent the cryptographic keys used for signing tokens. [You can learn more about this subject here](https://auth0.com/blog/navigating-rs256-and-jwks/).

You probably noticed that the decorator that your created is using a module called `jwt` that is not available to your project. To install this dependency, issue the following command inside the `backend` directory:

```bash
pipenv install python-jose-cryptodome
```

Now, to use this decorator, you can update the `main.py` file as follows:

```python
# coding=utf-8

# ... other import statements ...
from .auth import AuthError, requires_auth

# ... create Flask app, schema generation ...

# ... get_exams implementation ...

@app.route('/exams', methods=['POST'])
@requires_auth
def add_exam():
    # ... implementation ...

@app.errorhandler(AuthError)
def handle_auth_error(ex):
    response = jsonify(ex.error)
    response.status_code = ex.status_code
    return response
```

Note that you are securing only the `add_exam` function (i.e. the endpoint that accepts `POST` requests). The endpoint that retrieves exams will remain public so visitors can see what exams your application contains. Another important feature added by the new version of this file is the `handle_auth_error` function. By using the `@app.errorhandler` decorator, you are configuring your Flask application to call this function when `AuthErrors` are raised. This error handler makes errors look nice and easier to fix as they simply return a JSON object with the details.

That's it! You Flask application is now secured with Auth0. To test it, you can issue the following commands:

```bash
# run the app in the background
./bootstrap &

# good to go, endpoint not secured
curl http://0.0.0.0:5000/exams

# unauthorized, endpoint secured and no bearer
curl -X POST -H 'Content-Type: application/json' -d '{
  "title": "TypeScript Advanced Exam",
  "description": "Tricky questions about TypeScript."
}' http://0.0.0.0:5000/exams
```

Now, to get an access token to test the secured endpoint, you will need to copy a command from your Auth0 API. So, head back to the page that you left open, then click on the *Test* tab, and copy the first `curl` command shown there. The code snippet below shows how to use this command to fetch the access token and how to send it to your Flask application (replace the first command with the one you copied from your Auth0 API):

```bash
# retrieve token from Auth0
curl -X POST -H 'Content-Type: application/json' -d '{
    "client_id":"Rp4He7RGWAnMfY34SxwnWHpw7gjdOOPI","client_secret":"2cCS521K7k2Btc434VMBmIBTeLbRfw1tvEKw6l8DYt9OvIEYHUqgdZPioM_rBKsx","audience":"https://online-exam.digituz.com.br","grant_type":"client_credentials"
}' https://bk-samples.auth0.com/oauth/token

# copy token into env variable
JWT="aaa.bbb.ccc"

# create new exam
curl -X POST -H 'Content-Type: application/json' -H 'Authorization: Bearer '$JWT -d '{
  "title": "TypeScript Advanced Exam",
  "description": "Tricky questions about TypeScript."
}' http://0.0.0.0:5000/exams
```

Hurray! You have a Flask application secured with a modern identity management solution. Time to save your progress:

```bash
git add . && git commit -m "securing Flask with Auth0"
```

## Adding Form to Create Exams

Next, you can go back to your code. Before integrating Auth0 into your Angular app, you will add two new components to it: `ExamsComponent` and `ExamFormComponent`. The first one will render the list of exams (i.e. you will remove this functionality from the `AppComponent`) and the second one will allow authenticated users to create exams.

As such, open the `app.component.ts` file and replace all its content with this:

```typescript
import {Component} from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <div style="text-align:center">
      <h1>Exams</h1>
    </div>
    <h2>Here are the exams created so far: </h2>
    <router-outlet></router-outlet>
  `,
  styleUrls: ['./app.component.css']
})
export class AppComponent { }
```

As you can see, this component exists only to show a title and to define where other components will be rendered (`<router-outlet></router-outlet>`). As you have moved this component's template to this file, you can remove the `app.component.html` file.

Now, to show the exams list again, you are going to create the `ExamsComponent`. To do so, create a new file called `exams.component.ts` inside the `src/app/exams` directory with the following code:

{% highlight html %}
{% raw %}
import {Component, OnDestroy, OnInit} from '@angular/core';
import {Subscription} from 'rxjs/Subscription';
import {Exam} from './exam.model';
import {ExamsApiService} from './exams-api.service';

@Component({
  selector: 'exams',
  template: `
    <div>
      <button routerLink="/new-exam">New Exam</button>
      <ul>
        <li *ngFor="let exam of examsList">
          {{exam.title}}
        </li>
      </ul>
    </div>
  `
})
export class ExamsComponent implements OnInit, OnDestroy {
  examsListSubs: Subscription;
  examsList: Exam[];

  constructor(private examsApi: ExamsApiService) {
  }

  ngOnInit() {
    this.examsListSubs = this.examsApi
      .getExams()
      .subscribe(res => {
          this.examsList = res;
        },
        console.error
      );
  }

  ngOnDestroy() {
    this.examsListSubs.unsubscribe();
  }
}
{% endraw %}
{% endhighlight %}

There is nothing fancy about this component, you are simply moving the code that renders the list of exams from the `AppComponent` to this one and adding a `button` to enable users to navigate to the `/new-exam` page.

Speaking of which, you have to create the component that will be responsible for this URI. Therefore, create a new file called `exam-form.component.ts` inside the `src/app/exams` directory and add the following code to it:

```typescript
import {Component} from '@angular/core';
import {HttpClient} from '@angular/common/http';
import {ExamsApiService} from "./exams-api.service";
import {Router} from "@angular/router";

@Component({
  selector: 'exam-form',
  template: `
    <div>
      <h2>New Exam</h2>
      <label for="exam-title">Title</label>
      <input id="exam-title" (keyup)="updateTitle($event)">
      <label for="exam-description">Description</label>
      <input id="exam-description" (keyup)="updateDescription($event)">
      <button (click)="saveExam()">Save Exam</button>
    </div>
  `
})
export class ExamFormComponent {
  exam = {
    title: '',
    description: '',
  };

  constructor(private examsApi: ExamsApiService, private router: Router) { }

  updateTitle(event: any) {
    this.exam.title = event.target.value;
  }

  updateDescription(event: any) {
    this.exam.description = event.target.value;
  }

  saveExam() {
    this.examsApi
      .saveExam(this.exam)
      .subscribe(
        () => this.router.navigate(['/']),
        error => alert(error.message)
      );
  }
}
```

In this component, you are defining two `input` elements where users will be able to inform the `title` and `description` of the exam that they are creating and a `button` that triggers the `saveExam` method. This method calls another (homonymous) method that you will define in the `examsApi` service. Then, if the call to this method is successful, users are redirected to the home page (`/`) and, if the call is unsuccessful, an `alert` is shown by the browser.

Now, to define the `saveExam` method on the `ExamsApiService`, open the `exams-api.service.ts` file and update it as follows:

```typescript
// ... import statements ...

@Injectable()
export class ExamsApiService {
  // ... constructor, _handleError, getExams ...

  saveExam(exam: Exam): Observable<any> {
    return this.http
      .post(`${API_URL}/exams`, exam);
  }
}
```

Lastly, you will have to update the `app.module.ts` file as follows:

```typescript
// ... other imports ...

import {ExamFormComponent} from './exams/exam-form.component';
import {RouterModule, Routes} from '@angular/router';
import {ExamsComponent} from './exams/exams.component';

const appRoutes: Routes = [
  { path: 'new-exam', component: ExamFormComponent },
  { path: '', component: ExamsComponent },
];

@NgModule({
  declarations: [
    AppComponent,
    ExamFormComponent,
    ExamsComponent,
  ],
  imports: [
    // ... BrowserModule and HttpClientModule ...
    RouterModule.forRoot(
      appRoutes,
    ),
  ],
  // ... providers and bootstrap ...
})
// ... AppModule class definition
```

With these changes in place, you can open a terminal, move to the `frontend` directory, and run the `ng serve` command. Then, if you open [`http://localhost:4200/`](http://localhost:4200/) in a browser, you will see the list of exams and a button labeled *New Exam*. Clicking on this button, you will be redirected to your new form. The problem now is that you will get an error saying "Http failure response for http://localhost:5000/exams: 401 UNAUTHORIZED" when clicking on the *Save Exam* button. The reason for that is simple, you haven't configured Auth0 in your Angular app yet.

## Securing Angular Apps with Auth0

To solve the `401 UNAUTHORIZED` issue, the first thing you will have to do is to create an [Auth0 Client](https://auth0.com/docs/clients) to represent your Angular app. To do so, browse to [the Client page on the Auth0 dashboard](https://manage.auth0.com/#/clients) and click on the *Create Client* button. This time, you will have to provide only two things to Auth0:

1. *Name*: Another friendly reminder, this time to your Auth0 Client. Here, you can add something like "Online Exam Client".
2. *Client Type*: The type of the client that you are creating. In this case, as you are using Angular to create a SPA, you will choose *Single Page Web Applications*.

![Creating an Auth0 Client to represent the Angular app.](https://cdn.auth0.com/blog/flask-angular/creating-auth0-client.png)

Having filled out this form, click on the *Create* button. When finished creating your client (it takes just a second or two), Auth0 will redirect you to the *Quick Start* tab of the new client. From there, click on the *Settings* tab to inform to Auth0 what the *Allowed Callback URLs* are. As for the moment you are only running your app locally, you can simply add the `http://localhost:4200/callback` URL to this field. Now, you can hit the *Save Changes* button at the bottom of the page and leave it open (you will need to copy some properties from it soon).

Now, you can go back to your Angular project and integrate it with Auth0. To do so, you will have to install the [`auth0-web` NPM package](https://github.com/brunokrebs/auth0-web). So, open a terminal, move to the `frontend` directory, and issue the following command:

```bash
npm i auth0-web
```

After installing this package, you will have to create the component responsible for [handling the callback URL called by Auth0](https://auth0.com/docs/client-auth/current/client-side-web#handle-the-callback). As such, create a new file called `callback.component.ts` in the `src/app/` directory and add the following code to it:

```typescript
import * as Auth0 from 'auth0-web';
import {Component, OnInit} from '@angular/core';
import {Router} from "@angular/router";

@Component({
  selector: 'callback',
  template: `
    <div>Loading authentication details...</div>
  `,
})
export class CallbackComponent implements OnInit {
  constructor(private router: Router) { }

  ngOnInit(): void {
    const self = this;
    Auth0.handleAuthCallback((err) => {
      if (err) alert(err);
      self.router.navigate(['/']);
    });
  }
}
```

As you can see, this component simply shows a message saying "Loading authentication details..." to users and delegates to `auth0-web` the process of parsing the callback URL. Then, when `auth0-web` finishes its job, users are redirected to the home page (`self.router.navigate(['/'])`).

Next, you will have to refactor the `exams.component.ts` file to allow users to sign in and sign out. So, open this file and replace its code with the following:

{% highlight html %}
{% raw %}
import * as Auth0 from 'auth0-web';
import {Component, OnDestroy, OnInit} from '@angular/core';
import {Subscription} from 'rxjs/Subscription';
import {Exam} from './exam.model';
import {ExamsApiService} from './exams-api.service';

@Component({
  selector: 'exams',
  template: `
    <div>
      <button routerLink="/new-exam">New Exam</button>
      <button (click)="signIn()" *ngIf="!authenticated">Sign In</button>
      <button (click)="signOut()" *ngIf="authenticated">Sign Out</button>
      <p *ngIf="authenticated">Hello, {{getProfile().name}}</p>
      <ul>
        <li *ngFor="let exam of examsList">
          {{exam.title}}
        </li>
      </ul>
    </div>
  `
})
export class ExamsComponent implements OnInit, OnDestroy {
  examsListSubs: Subscription;
  examsList: Exam[];
  authenticated = false;

  constructor(private examsApi: ExamsApiService) { }

  signIn = Auth0.signIn;
  signOut = Auth0.signOut;
  getProfile = Auth0.getProfile;

  ngOnInit() {
    this.examsListSubs = this.examsApi
      .getExams()
      .subscribe(res => {
          this.examsList = res;
        },
        console.error
      );
    const self = this;
    Auth0.subscribe((authenticated) => (self.authenticated = authenticated));
  }

  ngOnDestroy() {
    this.examsListSubs.unsubscribe();
  }
}
{% endraw %}
{% endhighlight %}

In the new version of this component, you are adding two new buttons (*Sign In* and *Sign Out*) and a paragraph that shows names of authenticated users. All these new elements are conditionally showed according to the `authenticated` flag. The *Sign Out* and *New Exam* buttons are visible when this flag is set, otherwise you'll see the *Sign In* button.

To make everything work, the new `ExamsComponent` references and calls three methods provided by `auth0-web` (`signIn`, `signOut`, and `getProfile`). Besides these methods, this component also subscribes an anonymous function to change the value of the `authenticated` flag whenever the authentication status changes.

After updating the `ExamsComponent`, you will have to update the `ExamsApiService` to make use of the `access_token` retrieved from Auth0. So, open the `exams-api.service.ts` file and update it as follows:

```typescript
// ... other imports ...
import {HttpClient, HttpErrorResponse, HttpHeaders} from '@angular/common/http';
// ... other imports ...
import * as Auth0 from 'auth0-web';

@Injectable()
export class ExamsApiService {

  // ... constructor, _handleError, and getExams

  saveExam(exam: Exam): Observable<any> {
    const httpOptions = {
      headers: new HttpHeaders({
        'Authorization': `Bearer ${Auth0.getAccessToken()}`
      })
    };
    return this.http
      .post(`${API_URL}/exams`, exam, httpOptions);
  }
}
```

The difference to the previous version is that now you are adding the `Authorization` header to the `POST` request that is sent when users try to save a new exam. This header is the missing piece to make your Angular and your Flask apps communicate properly.

So, to wrap up everything, you will have to update your `AppModule` to declare the `CallbackComponent`, register the `/callback` route, and to configure the `auth0-web` package. As such, open the `app.module.ts` and change it as follows:

```typescript
import * as Auth0 from 'auth0-web';
import {CallbackComponent} from './callback.component';

// ... other import statements ...
const appRoutes: Routes = [
  { path: 'callback', component: CallbackComponent },
  // ... other routes ...
];

@NgModule({
  declarations: [
    // ... other components ...
    CallbackComponent,
  ],
  // ... imports, providers, and bootstrap ...
})
export class AppModule {
  constructor() {
    Auth0.configure({
      domain: 'bk-samples.auth0.com',
      audience: 'https://online-exam.digituz.com.br',
      clientID: 'oxiIp4EX1diCft0rOjzTc9PnHRvtuh9a',
      redirectUri: 'http://localhost:4200/callback',
      scope: 'openid profile manage:exams'
    });
  }
}
```

**Note that** you will have to replace the `domain`, `audience`, and `clientID` properties on this code with values from your Auth0 account. So, head back to your Auth0 management dashboard and copy the `domain` and `clientID` from the Auth0 Client created before and `audience` from the Auth0 API (that is, copy the API identifier and paste it here).

That's it! If you run your Angular application now, you will be able to create new exams.

![Developing modern applications with Python, Flask, and Angular.](https://cdn.auth0.com/blog/flask-angular/angular-and-flask-secured-with-auth0.png)

Wait! Don't forget to save your progress!!

```bash
git add .
git commit -m "Integrating Angular with Auth0"
```

{% include tweet_quote.html quote_text="Securing applications with Auth0 is easy and allows me to focus on my apps features." %}

## Conclusion and Next Steps

In the second part of this series, you focused on adding Auth0 to act as the identity management service of your Flask and Angular applications. You started by defining an Auth0 API to represent your Flask backend app, then you added a new feature into your project (a form that allows users to add exams), and, finally, you integrated Auth0 into your Angular application.

In the next article, you are going to add even more features to your project and, after that, you will start preparing your code for CI/CD (Continuous Integration and Continuous Delivery) tools. These tools will help you automate the development pipeline. Stay tuned!
