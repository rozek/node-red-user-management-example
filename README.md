# node-red-user-management-example #

This repository contains an example for user management based on Node-RED flows. While it was designed to be immediately usable with the server implemented in [node-red-within-express](https://github.com/rozek/node-red-within-express) and play well together with the authentication and authorization mechanisms described in [node-red-authorization-examples](https://github.com/rozek/node-red-authorization-examples), the example may also be used in other environments.

> Nota bene: this work is currently in progress, do not expect it to be finished before end of October 2021

> Nota bene: this example uses an extended format for the "User Registry" introduced in "node-red-within-express". If you plan to use it, please update your `registeredUsers.json` file accordingly and take the new user id format into account when accessing the Node-RED editor (e.g., `node-red@mail.de` instead of `node-red`)

> Nota bene: while this example already considers a few legal regulations, no guarantee can be given that these measures are correct and/or complete. **It is entirely your responsibility to offer a service which complies with the legal requirements!**

> Just another small note: if you like this work and plan to use it, consider "starring" this repository (you will find the "Star" button on the top right of this page), so that I know which of my repositories to take most care of.

## Prerequisites ##

The example requires the following Node-RED extensions

* [node-red-reusable-flows](https://github.com/rozek/node-red-reusable-flows)<br>"Reusable Flows" allow multiply needed flows to be defined once and then invoked from multiple places
* [node-red-node-email](https://github.com/node-red/node-red-nodes/tree/master/social/email)<br>allows to create and send EMails from Node-RED

Additionally, the example expects the global flow context to contain an object called `UserRegistry` which has a format based on the one described in "node-red-within-express" but with a few extensions:

* the object's property names are the **email addresses of registered users**<br>(this specification differs from the original one!)
* the object's property values are JavaScript objects with the following properties, at least (additional properties may be added at will):
  * **Roles**<br>is either missing or contains a list of strings with the user's roles. There is no specific format for role names except that a role name must not be longer than 64 characters and must not contain any white space. This example already defines the role `user-admin` to identify "User Administrators", i.e., users who are allowed to manage the settings of other users
  * **Salt**<br>is a string containing a random "salt" value which is used during PBKDF2 password hash calculation
  * **Hash**<br>is a string containing the actual PBKDF2 hash of the user's password
  * **UUID**<br>is a string containing a [version 4 UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) which uniquely identifies a given user even after email address changes (this property has been added to the original specification). If you want to generate UUIDs for already existing users, this [online generator](https://www.uuidgenerator.net/version4) might be useful
  * **agreedToDPS**<br>is a boolean indicating that the user has read and agreed to this service's "Data Privacy Statement" and must be set to `true` for an account to get confirmed (this property has been added to the original specification)
  * **agreedToTOS**<br>is a boolean indicating that the user has read and agreed to this service's "Terms of Service" and must be set to `true` for an account to get confirmed (this property has been added to the original specification)
  * **ConfirmationDeadline**<br>if present, `ConfirmationDeadline` indicates a pending user id conformation and contains the (UTC) time until when that process has to be completed (or the previously reserved new user id will be released)
  * **ConfirmationToken**<br>contains the token required to complete a pending user id confirmation. Actually, this token should not be saved at all as it could be recomputed from known key data - but then, any server reboot or flow redeployment would break pending user id confirmations (because of a new token key). In order to avoid inconveniences for users, tokens are therefore explicitly saved
  * **PasswordDeadline**<br>if present, `PasswordDeadline` indicates a pending password reset and contains the (UTC) time until when that process has to be completed
  * **PasswordToken**<br>contains the token required to complete a pending password reset. Actually, this token should not be saved at all as it could be recomputed from known key data - but then, any server reboot or flow redeployment would break pending password resets (because of a new token key). In order to avoid inconveniences for users, tokens are therefore explicitly saved

When used outside "node-red-within-express", the following flows allow such a registry to be loaded from an external JSON file called `registeredUsers.json` (or to be created if no such file exists or an existing file can not be loaded) and written back after changes:

![](outside-node-red-within-express.png)

Just import [these flows](outside-node-red-within-express.json), place them on your Node-RED workspace but - if need be - disable the node labelled "at Startup" as the startup sequence is now a different one and will be handled by a different flow. By default, the created user registry contains a single user `node-red@mail.de` with the password `t0pS3cr3t!` and a single role `node-red` (this is exactly the same user who is also included in "node-red-within-express" by default)

## Typical User Management Lifecycles ##

A typical user lifecycle looks as follows:

* **a new user gets registered**<br>either by him/herself or by an administrator. In this implementation, new users who register themselves only have to specify their email address in order to start a registration. To limit the users who can register, a list of permitted email addresses may be provided to the server
* **upon registration, an "account confirmation message" is sent** by email to the given address<br>This email contains a link which, when clicked, should navigate the user to a web page (or web application) where the he/she *may* add additional information (such as his/her real name, f.e.), *must* define a password for his/her account, *must* (read and) agree to a "Data Privacy Statement" and to some "Terms of Service" (for legal reasons) and then send that information back to the server in order to **confirm the account**. If such a confirmation does not happen within a certain period, the registration is automatically cancelled - until then, the given email address is reserved and no other account with the same address may be registered
* after successful account confirmation, **the user may "log-in"** and use the offered service<br>While this implementation uses the "Header-based Authorization" described in [node-red-authorization-examples](https://github.com/rozek/node-red-authorization-examples) any of the other methods will work as well - but you will have to modify the flows in this repository accordingly
* while logged-in, a user may additionally
  * **change his/her email address**<br>changing one's email address will actually start a confirmation process similar to that after initial registration. As soon as the new email address is confirmed, the old one will be removed and the new one used from now on for the already existing account. Until then (or if the new address does not get confirmed in time) the old email address continues to work as before
  * **change his/her password**<br>changing one's password requires to specify the old password again for authorization - if that password is fogotten, a "password reset" may be initiated (see below)
  * **change other account details**<br>custom server implementations may need additional user information (such as a user's real name, his/her postal address, credit card details etc.) These details may be changed online as well - except the user's agreement to "Data Privacy Statement" and "Terms of Service", which may only be cleared)
* should a user have forgotten his/her password, he/she may **start a "password reset" process**<br>As a consequence, a "password reset message" is sent by email to the user's address. This email contains a link which, when clicked, should navigate the user to a web page (or web application) where he/she may define a new password without having to specify the current one
* last, but not least, every user may **delete his/her account**<br>This action immediately removes the user's account and wipes out any data associated with this user

### User Administrators ###

While "normal" users may only inspect and modify their own accounts, "User Administrators" (i.e., users with the role `user-admin`) may also manage the accounts of other people. In particular, they may

* **change other user's email addresses**<br>although the new addresses will have to be confirmed in the same way as if the users would have initiated the change themselves
* **change other user's roles**<br>this function is only available to user administrators
* **inspect or change other user's account details**<br>any detail may be changed except email addresses, passwords, `Salt` and `Hash` values or the agreement to "Data Privacy Statement" or "Terms of Service" (which may only be cleared)
* **delete other users**

But even "User Administrators" neither have access to other user's passwords nor can they set other user's agreements to the service's "Data Privacy Statement" and "Terms of Service"

### "Data Privacy Statement" and "Terms of Service" ###

> Nota bene: the legal documents mentioned above (i.e., "Data Privacy Statement" and "Terms of Service") are not part of this user management implementation but have to be provided separately

Should "Data Privacy Statement" and/or "Terms of Service" change, properties `agreedToDPS` and/or `agreedToTOS` may be set to `false` again. In that case, any new login should immediately redirect the user to a separate web document where he/she may either (read the changed documents and) agree again or delete his/her account. Without such an agreement, the user should no longer be allowed to access the service and logged out immediately.

Administrators also have to agree to these legal statements.

## REST Interface for User Management ##

### POST /user/&lt;user-id&gt;/register ###
 
* POST without body
* &lt;user-id&gt; must be a valid email address (max. 64 char.s long) which is neither currently registered nor the new address of an account which is currently being renamed
* the requesting user does not have to authenticate him/herself
* if all conditions are met a registration email will be sent to the given &lt;user-id&gt;
* warning: currently, success of email submission can not be tested - it may be, that an email address is successfully registered which never receives any email

### POST /user/&lt;user-id&gt;/send-confirmation-message ###
 
* POST without body
* &lt;user-id&gt; must be the email address of an account which still has to be confirmed (either because it has just been registered or because it is the new address of an account which is currently being renamed)
* the requesting user does not have to authenticate him/herself
* if all conditions are met, another registration email (with the original deadline) will be sent to the given &lt;user-id&gt;
* warning: currently, success of email submission can not be tested - it may be, that an email address is successfully registered which never receives any email

### POST /user/&lt;user-id&gt;/confirm ###

* will be used either to confirm a newly registered account or to confirm the new email address of an existing account
* **confirming a newly registered account**
  * POST with form variables `newPassword`, `agreedToDPS`, `agreedToTOS` and the confirmation `Token` from the submitted confirmation link, at least
  * &lt;user-id&gt; must be the email address of an account which still has to be confirmed because it has just been registered
  * the requesting user does not have to authenticate him/herself
  * `newPassword` must contain a valid password (in this example, with at least 12 char.s and no other constraints)
  * `agreedToDPS` must be `true`
  * `agreedToTOS` must be `true`
  * `Token` must contain a valid confirmation token for the given account
  * if all conditions are met, a hash for the given `newPassword`, `agreedToDPS` and `agreedToTOS` will be stored and the account marked as confirmed
  * the user may now log-in
* **confirming the new email address of an existing account**
  * POST with the confirmation `Token` from the submitted confirmation link, at least
  * &lt;user-id&gt; must be the email address of an account which still has to be confirmed because it is the new address of another account which is currently being renamed
  * the requesting user does not have to authenticate him/herself
  * `Token` must contain a valid confirmation token for the given account
  * if all conditions are met, any settings from the old account are copied to the new one, the new account is marked as confirmed and the old account deleted
  * the user may now log-in using the new email address - the old one may no longer be used

### POST /user/&lt;user-id&gt;/start-password-reset ###
 
* POST without body
* &lt;user-id&gt; must be the email address of an account which has already been confirmed
* the requesting user does not have to authenticate him/herself
* if this condition is met, a "password reset message" is sent to the given &lt;user-id&gt;
* warning: currently, success of email submission can not be tested - it may be, that a "password reset message" is sent to an address which no longer receives any email

### POST /user/&lt;user-id&gt;/reset-password ###

* POST with form variable `newPassword` and the `Token` from the submitted password reset link
* &lt;user-id&gt; must be the email address of an account which has already been confirmed but is currently in the process of a password reset
* the requesting user does not have to authenticate him/herself
* `newPassword` must contain a valid password (in this example, with at least 12 char.s and no other constraints)
* if all conditions are met, a hash for the given `newPassword` will be stored and the password reset process be marked as complete
* warning: only the latest token will be accepted - and this only once

### POST /user/&lt;user-id&gt;/update-legal-agreements ###

* POST with form variables `agreedToDPS` and `agreedToTOS`
* &lt;user-id&gt; must be the email address of an account which has already been confirmed
* the requesting user must have authenticated him/herself as the user with the given &lt;user-id&gt;
* `agreedToDPS` must be `true`
* `agreedToTOS` must be `true`
* if all conditions are met, `agreedToDPS` and `agreedToTOS` will be stored and the user may continue using the offered service

### POST /user/&lt;user-id&gt;/authenticate ###

* POST with form variable `Password`
* &lt;user-id&gt; must be the email address of an account which has already been confirmed
* `Password` must be the current password for this account
* if all conditions are met, a "bearer" token with the currently configured lifetime will be generated and returned in an "authorization" header. This token can now be used for further requests to authenticate the logged-in user
* the token expires after a configured time of inactivity but - within its lifetime - it gets refreshed by every request which requires an authentication
* explit authentications are not necessary if "basic HTTP authentication" is used

### POST /user/&lt;user-id&gt;/change-user-id ###

* POST with form variable `newUserId`
* &lt;user-id&gt; must be the email address of an account which has already been confirmed
* the requesting user must have authenticated him/herself either as the user with the given &lt;user-id&gt; or as a user with the role `user-admin`
* `newUserId` must be a valid email address (max. 64 char.s long) which is neither currently registered nor the new address of an account which is currently being renamed
* if all conditions are met, a registration email will be sent to `newUserId`
* warning: currently, success of email submission can not be tested - it may be, that an email address is successfully registered which never receives any email

### POST /user/&lt;user-id&gt;/change-password ###

* POST with form variables `oldPassword` and `newPassword`
* the requesting user must have authenticated him/herself as the user with the given &lt;user-id&gt;
* `oldPassword` must be the current password for the account with the given &lt;user-id&gt;
* `newPassword` must contain a valid password (in this example, with at least 12 char.s and no other constraints)
* if all conditions are met, a hash for the given `newPassword` will be stored in the account and the new password will become active

### POST /user/&lt;user-id&gt;/change-roles ###

* POST with form variable `Roles` (containing a space-separated list of permitted user roles)
* the requesting user must have authenticated him/herself as a user with the role `user-admin`
* if all conditions are met, the given `Roles` will be stored in the account for the given &lt;user-id&gt; and immediately become active

### GET /user/&lt;user-id&gt; ###

* the requesting user must have authenticated him/herself as the user with the given &lt;user-id&gt; or as a user with the role `user-admin`
* if this condition is met, a JSON object with any public details of the account for the given &lt;user-id&gt; is sent back

### POST /user/&lt;user-id&gt; ###

* POST with arbitrary form variables expect `Password`, `Salt`, `Hash`, `Salt`, `ConfirmationDeadline`, `PasswordDeadline`, `agreedToDPS`, `agreedToTOS` and `Roles`
* the requesting user must have authenticated him/herself as the user with the given &lt;user-id&gt; or as a user with the role `user-admin`
* if all conditions are met, the given variables are written to the account for the given &lt;user-id&gt;

### DELETE /user/&lt;user-id&gt; ###

* the requesting user must have authenticated him/herself as the user with the given &lt;user-id&gt; or as a user with the role `user-admin`
* if this condition is met, the account for the given &lt;user-id&gt; and any associated data is deleted
* for a user with the role `user-admin` it is permitted to delete a non-existing user

## REST Interface for Server Administration ##

The following HTTP endpoints require the requesting user to authenticate as an administrator:

* **`GET /registered-users`**<br>responds with a JSON file containing all currently registered users and their settings (see above for their description)
* **`PUT /registered-users`**<br>updates the list of currently registered users. The request body should therefore contain a JSON specification of all these users and their settings (see above for their description)
<br>&nbsp;<br>
* **`GET /permitted-users`**<br>responds with a list of all currently permitted users and their intended roles in plain text form. The list contains one user per line, each line starts with the user id (i.e., the user's email address) and (optionally) a blank-separated list of role names for that user
* **`PUT /permitted-users`**<br>updates the list of all permitted users and their intended roles. The request body should therefore contain a text file with one line per user, starting with the user id (i.e., the user's email address) and (optionally) followed by a blank-separated list of role names for that user
<br>&nbsp;<br>
* **`GET /permitted-roles`**<br>responds with a list of all currently permitted roles in plain text form - one role name per line
* **`PUT /permitted-roles`**<br>updates the list of all permitted roles. The request body should therefore contain a text file with a white-space separated list of role names. You may use spaces, tabs, line-feeds and similar white-space characters for separation

## Installation ##

This example may be installed as follows:

* import [this flow](UserManagement.json) into your Node-RED instance
* configure the "email" node, i.e. enter service details and credentials of an email account you may use for sending email messages
* copy files `registeredUsers.json`, `permittedUsers.txt` and `permittedRoles.txt` into your Node-RED working directory
* customize these files as needed

## Postman Collection ##

As usual, this repository contains a [collection of requests](PostmanCollection.json) which is to be imported into a [Postman](https://www.postman.com/) instance and simplifies testing the feedback mechanism.

> Nota bene: you will have to change the user ids mentioned in the Postman Collection in order to access confirmation and password reset emails

## License ##

[MIT License](LICENSE.md)
