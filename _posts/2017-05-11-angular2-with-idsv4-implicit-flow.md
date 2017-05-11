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

Steps
* Write an Authentication Service using Oidc-Client.js.
* Add a `signin-callback.html` (as the callback after successful sign-in).
* Add a `silent-renew-callback.html` (to be loaded in the hidden iframe).
* Add a AuthenticationCheck to secure the Routes.

`silent-renew-callback.html`
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

`authentication.service.ts`
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

`authentication-check.ts`
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
export class AuthorizationCheck implements CanActivate {
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