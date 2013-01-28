Introduction
============

Nopassword is an authentication server that authenticates users by sending
them an email with a one-time short-lived login link.  When a user clicks
that link on the same computer and browser that requested it, a secure
cookie is set with an auth token that authenticates the user on subsequent
requests.

Nopassword can easily be added to existing webapps, no matter what
programming language they were written in.  All that's needed is an HTML
form for users to submit their email address and a web server directive to
dispatch authentication requests to Nopassword.  Your webapp can then
authenticate users by issuing an HTTP request to the Nopassword server.

Nopassword was inspired by this
[blog post by Ben Brown](http://notes.xoxco.com/post/27999787765/is-it-time-for-password-less-login)
and the desire of creating a language-agnostic version of the
[NoPassword plugin for Ruby on Rails by Alex Smolen](https://github.com/alsmola/nopassword).

Installation
============

1. Edit the config_file and save it in a convenient location.

2. Create a form with an `<input type="text" name="email">` and a submit
   button.  Make this form POST to a URL under your control (e.g.,
   https://example.com/login)
 
3. Start the Nopassword server by running `nopassword <path-to-config_file>`

4. Setup your webserver to reverse proxy login requests to the Nopassword
   server.  For instance, if you've used https://example.com/login in step
   2, the following nginx config should do the trick:

       ```
       location /login {
         rewrite /login/(.*) /$1 break;
         proxy_pass http://localhost:1500;
         proxy_redirect off;
         proxy_set_header X-Real-IP  $remote_addr;
       }
       ```

   Note that the default port number is 1500, but you can change that in the
   config_file. 

5. Create some authentication function/middleware in your webapp that
   authenticates requests by contacting the Nopassword server and giving it
   the auth token the user supplied.  The auth token is the value of the
   cookie with the name `Nopassword`. In pseudocode:

       ```python
       auth_token = request.cookies.get('Nopassword')
       if auth_token:
           response = requests.get('http://localhost:1500/' + auth_token)
           if response.status_code == 200:
               user_id = response.json['user_id']
               user_email = response.json['user_email']
               '''
               At this point, the user is logged in and you have his/her id
               and email-address. You can use the id as the primary key in
               your user database. Next, you should probably check if the id
               is already present in your database or else create a new user
               and show a nice welcome message.
               '''
           else:
               raise AuthenticationFailed
       else
           show_login_page()
	   
       ```

   If you use Python with
   [Requests](http://docs.python-requests.org/en/latest/), this pseudocode
   actually works!


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
- system sets auth token as a secure, persistent cookie
- user sends back the cookie on every subsequent request
- system authenticates the user on every subsequent request
- everybody's happy


URL Routes
----------

- `GET /` -> 404

- `POST /` expects a value for the key 'email' in the request body. If
  sending the login email succeeds, returns a 303 redirect to the
  `email_succeeded` URL specified in the config_file.  If sending the email
  fails, returns a 303 redirect to the `email_failed` URL.  Otherwise,
  returns a 400 Bad Request.

- `GET /<login_token>` is requested when the user clicks the login link in
  the email.  It sets the Nopassword cookie and 303 redirects to the
  `login_succeeded` URL defined in the config_file. Otherwise, redirects to
  the `login_failed` URL.

- `GET /<auth_token>` is usually requested by the webapp itself. If the
  auth token is valid, the 200 response contains a JSON object with the
  fields `user_id` and `user_email`.  Else the response is 401.  Note that
  this URL, like all the others, is public and an attacker can request the
  email address of a compromised auth cookie.  That's why Nopassword sets
  secure (SSL) cookies by default, to prevent eavesdroppers from stealing
  cookies.

- `PUT /<auth_token>` is used to change a users email address. The request
  body should contain the new email address.  Note that anyone who knows a valid
  auth token can issue this request to change the email address.

TODO: Find a way to distinguish between login tokens and auth tokens.


Relations
---------

- user
  * id (PK)
  * email address
  * login token
  * login token expiry time
  * auth token
