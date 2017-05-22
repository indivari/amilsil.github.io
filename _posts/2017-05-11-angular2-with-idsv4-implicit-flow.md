---
layout: post
title: Angular 2 With Identity Server 4 Implicit Flow
tags:
- identityserver
- angular2
- javascript
- implicit-flow
- token-refresh
---

If you are using Identity Server 4 for authenticating an *angular 2* or higher based web application, chances are you are using **identity server implicit authentication flow**. Means you are using browser redirects to grab the *access token*. In that case *token refresh* is done through a *hidden iframe*. In this post I am trying to show you how this could be done using Angular 2. 

Identity Server comes with a javascript client *oidc-client.js* that can do the work for you. However, since it's not targeted for angular clients, there are a few things we have to do to make it work on angular.

Steps
* Write an Authentication Service using Oidc-Client.js.
* Add a `signin-callback.html` (as the callback after successful sign-in, to grab the *access token*).
* Add a `silent-renew-callback.html` (to be loaded in the hidden iframe, to refresh the *access token*).
* Add a AuthenticationCheck to secure the Routes.

### Main Pieces of the Flow
`authentication.service.ts`

The most important bit. Initializes the *oidc-client.js* with the following settings. The few methods in the *UserManager* will hereafter work with those settings. Like *StartSigninMainWindow()* which will redirect the browser to the identity server for authenticating the user.

Few settings to note here are,

* **authority**: the identity server uri
* **redirect_uri**: the page the user is redirected to, after authentication (along with the token)
* **slient_redirect_uri**: the page the user is redirected to, after refreshing a token (inside of a hidden iframe)

```typescript
import { AppState } from './../../app.service';
import { Injectable, EventEmitter } from '@angular/core';
import { Http, Headers, RequestOptions, Response } from '@angular/http';
import { Observable } from 'rxjs/Rx';

import { UserManager, User, Log } from 'oidc-client';
import { appConfig } from '../../app-config';

const settings: any = {
    authority: 'http://localhost:5000',
    client_id: 'mvc.client',
    redirect_uri: 'http://localhost:3000/signin-callback.html',
    post_logout_redirect_uri: 'http://localhost:3000',
    response_type: 'id_token token',
    scope: 'openid profile api1',

    silent_redirect_uri: 'http://localhost:3000/silent-renew-callback.html',
    automaticSilentRenew: true,
    accessTokenExpiringNotificationTime: 10,
    silentRequestTimeout: 10000,

    filterProtocolClaims: true,
    loadUserInfo: false
};

Log.logger = console;
Log.level = Log.DEBUG;

@Injectable()
export class AuthenticationService {
    mgr: UserManager = new UserManager(settings);
    userLoadededEvent: EventEmitter<User> = new EventEmitter<User>();
    currentUser: User;
    loggedIn = false;

    authHeaders: Headers;


    constructor(private http: Http, private appState: AppState) {

        this.mgr.getUser()
            .then((user) => {
                if (user) {
                    this.loggedIn = true;
                    this.appState.setUser(user);
                    this.currentUser = user;
                    this.userLoadededEvent.emit(user);
                    this.appState.authenticate(true);
                }
                else {
                    this.loggedIn = false;
                }
            })
            .catch((err) => {
                this.loggedIn = false;
            });

        this.mgr.events.addUserLoaded((user) => {
            this.currentUser = user;
            this.appState.setUser(user);
            this.appState.authenticate(true);
            console.log('authService addUserLoaded', user);
        });

        this.mgr.events.addAccessTokenExpired((e) => {
            this.startSigninMainWindow();
        });

        this.mgr.events.addUserUnloaded((e) => {
            this.appState.authenticate(false);
            console.log('user unloaded');
            this.loggedIn = false;
        });

    }

    isLoggedInObs(): Observable<boolean> {
        return Observable.fromPromise(this.mgr.getUser()).map<User, boolean>((user) => {
            if (user) {
                return true;
            } else {
                return false;
            }
        });
    }
}
```

`signin-callback.html`

When the user is redirected to this page with the new token, it should be stored in the *local storage*. This is the only functionality of this page. Done with `UserManager().signinRedirectCallback()`. If this was successful the user is signed in. So we redirect the user to the *dashboard*. 

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title></title>
</head>
<body>
    <h2 id="waiting" style="text-align: center">Signing you in...</h2>
    <div id="error"></div>
    <script src="assets/js/oidc-client.min.js"></script>
    <script>
        Log.logger = console;
        new UserManager().signinRedirectCallback().then(function (user) {
            if (user == null) {
                document.getElementById("waiting").style.display = "none";
                document.getElementById("error").innerText = "No sign-in request pending.";
            }
            else {
                window.location = "/";
            }
        })
        .catch(function (er) {
            document.getElementById("waiting").style.display = "none";
            document.getElementById("error").innerText = er.message;
        });
    </script>
</body>
</html>
```

`silent-renew-callback.html`

Looks almost identical to the `signin-redirect-callback.html`, except this one calls the `UserManager().signinSilentCallback()`. Since this is done within an iframe, the func will propagate the call to the hosting window and persist the new token in the local storage.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title></title>
</head>
<body>
    <h1 id="waiting">Waiting...</h1>
    <div id="error"></div>
    <script src="assets/js/oidc-client.min.js"></script>
    <script>
         new UserManager().signinSilentCallback();
    </script>
</body>
</html>
```

That's basically about it on getting the *authentication flow* working. One additional thing that you might want to do is to guard the securable pages only to be available to a signed in user. This has more to do with angular routing. 


`authentication-check.ts`

AuthenticationCheck implements the `CanActivate` of `@angular/router`. This decides if a route should be navigated to or not. We simply call the `AuthenticationService` to check if the user is signed in. In which case we allow the user to navigate to the page, else not.

```typescript
import { Injectable } from "@angular/core";
import { CanActivate, RouterStateSnapshot, ActivatedRouteSnapshot, Router } from "@angular/router";
import { Observable } from 'rxjs/Observable';

import { AuthenticationService } from './authentication.service';

/*
* Checks if the user is authenticated & authorized before routing to the component. 
* Saves loading time.
*/
@Injectable()
export class AuthenticationCheck implements CanActivate {
    constructor(
        private router: Router,
        private authenticationService: AuthenticationService) { }

    canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean> {

        /*
         * If not authorized, redirect to identity server.
         */
        let isLoggedIn = this.authenticationService.isLoggedInObs();
        
        isLoggedIn.subscribe((loggedin) => {
            console.debug('AuthorizationCheck', `Authenticated ${loggedin}`);
            if (!loggedin) {
                this.authenticationService.startSigninMainWindow();
            }
        });

        return isLoggedIn;
    }
}
```

### Changes to Routes

The last thing in the puzzle is to add the *Check* to the routing. This makes sure those *Securable* components only have authenticated access, by using the `AuthenticationCheck` we wrote before.

```typescript
import { Routes, RouterModule } from '@angular/router';

import { DashboardComponent } from './dashboard';
import { ProtectedComponent } from './protected/protected.component';
import { AuthenticationService } from './shared/services/auth.service';
import { AuthenticationCheck } from './shared/services/auth-guard.service';

const appRoutes: Routes = [
    {
        path: '',
        redirectTo: '/dashboard',
        pathMatch: 'full'
    },
    {
        path: 'dashboard',
        component: DashboardComponent
    },
    {
        path: 'protected',
        component: ProtectedComponent,
        canActivate:[ AuthenticationCheck ]
    }

];
export const authProviders = [
    AuthenticationCheck,
    AuthenticationService
];

export const routing = RouterModule.forRoot(appRoutes);
```