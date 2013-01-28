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
dispatch auth requests to Nopassword.  Your webapp can then authenticate
users by calling a simple command-line utility.

Installation
============

1. Edit the config_file and save it in a convenient location.

2. Create a form with an `<input type="text" name="email">` and a submit
   button.  Make this form POST to a URL under your control (e.g.,
   https://example.com/login)

3. Start the Nopassword server by running `nopassword --server <path-to-config_file>`

4. Setup your webserver to reverse proxy login requests to the Nopassword
   server.  For instance, if you've used https://example.com/login in step
   2, the following nginx config should do the trick:

       '''perl
       location /login {
         proxy_pass        http://localhost:1500;
         proxy_set_header  X-Real-IP  $remote_addr;
       }
       '''

   Note that the default port number is 1500, but you can change that in the
   config_file.  Also note that the X-Real-IP http header should contain the
   client's IP address.  See the [nginx
   documentation](http://wiki.nginx.org/HttpProxyModule) for more details.

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
