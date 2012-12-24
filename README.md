Introduction
============

Nopassword is an authentication server that authenticates users by sending
them an email with a one-time short-lived login link.  When a user clicks
that link on the same computer and browser that requested it, a secure
cookie is set with an auth token that authenticates the user on subsequent
requests.

Nopassword can easily be added to existing webapps, no matter what
programming language they were written in. All that's needed is an HTML form
for users to submit their email address and a web server directive to dispatch
auth requests to Nopassword. You can then authenticate users by calling a
simple command-line utility.


How to use nopassword with your webapp
======================================

* Copy config_file to /etc/nopassword/yoursite.com and edit it
  appropriately.

* Create a form with an `<input type="text" name="email">` and a submit button.
  Make this form POST to yoursite.com/login

* Setup your webserver to redirect everything below /login to nopassword.wsgi

* Create some authentication function/middleware in your webapp:
  - To authenticate requests, run the following command:
    `nopassword <yoursite.com> <auth_token>`
  - Check the exit status to determine if an auth token is valid.
  - If the auth token is valid, you can extract the user's id and email address
    from the standard output.

* You can set a user's email address with the following command:
  `nopassword <yoursite.com> <user_id> <email_address>`


Scaling up
==========

The command-line utility is a potential bottleneck because it has to run in
a subprocess for each authentication request.  Therefore, the command line
program simply connects to the running wsgi app and calls the appropriate
function.  This way, when the load gets high, webapp programmers can
incorporate the connection code into their own program and prevent forking a
subprocess for each authentication request.


Architecture
============

What follows is some more detail about the currently planned implementation.
This is still highly subject to change.


Auth procedure
--------------

- user enters email and clicks 'login'
- user receives email with login link
- user clicks link and is logged in


Auth procedure elaborated
-------------------------

- user enters email and clicks 'login'
- system performs the following checks:
  * did we send mail to this address more than x times in the past y hours?
  * did we send mail for this ip address more than x times in the past y
    hours?
- system generates a login token and an expiry time and stores it linked to
  the user
- system emails a login link containing the token to the requested address
- user opens email and clicks link
- system looks up the user belonging to the token
- system checks if the current time isn't later than the expiry time
- system generates an auth token and stores it linked to the user
- system sets auth token as a secure, infinitely persistent cookie to the user
- user sends back the cookie on every subsequent request
- system authenticates the user on every subsequent request
- everybody's happy


Relations
---------

- user
  * id (PK)
  * email address
  * login token
  * login token expiry time
  * auth token
