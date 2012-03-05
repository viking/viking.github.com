---
title: Redirecting e-mail and Ruby-based authentication with Postfix
layout: post
---
I recently had the task of setting up a mail server to redirect mail
from one domain to another. The server also needed to be able to send
mail from accounts on the former domain. I originally tried using
exim4, but the documentation confused me so much that I decided to try
using Postfix instead. It turned out to be the easier choice for me.
Here's what I did to get it working on Ubuntu (10.04 Server).

#### Installation and redirection configuration ####

First, install Postfix:

    sudo apt-get install postfix

During the installation, it asks you what kind of configuration you
have.  Select *Internet site*, and then input your domain name in the
next screen (or just hit enter if Postfix guessed correctly).

Next, it's time to add redirection. Edit `/etc/postfix/main.cf` and
add the following line:

    virtual_alias_maps = hash:/etc/postfix/virtual

Now, edit the `/etc/postfix/virtual` file and add the source and
destination addresses of the redirections you want:

    foo@domain-i-am-forwarding.com foo@receiving-domain.com

The usernames don't have to be the same. The `/etc/postfix/virtual`
file is a whitespace delimited file with two columns: source address
and destination. You can put as much whitespace between the two
addresses as you want.

Next, you have to convert the `/etc/postfix/virtual` file into a
database file that Postfix can read:

    sudo postmap /etc/postfix/virtual

This command creates a file called `virtual.db` in the same directory
as the input file.  Postfix will now redirect any incoming e-mails
that match your `/etc/postfix/virtual` file. Hooray!

P.S. Make sure your MX domain record is set correctly.

#### Authenticating with Ruby from Postfix ####

Since I already have an OpenID server (built in Ruby) on this machine,
I wanted to use the same credentials for users on Postfix that I do
for the OpenID server. Postfix supports several different
authentication methods, but the methods based on databases (MySQL,
PostgreSQL, SQLite) require the passwords to be stored in plaintext,
which I didn't want.

Postfix supports PAM authentication, which has the ability to run a
script to determine if someone is authenticated or not. However,
Postfix relies on a separate daemon (saslauthd) in order to
authenticate, so I had to jump through a couple of hoops to do what I
wanted.

##### saslauthd ####

Install saslauthd via the `sasl2-bin` package:

    sudo apt-get install sasl2-bin

Next, edit `/etc/default/saslauthd` and change the `START` variable:

    START=yes

Make sure the `MECHANISMS` variable is set to `"pam"`. Also, change
the `OPTIONS` variable:

    OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd"

The reason for this is that Postfix runs in a chrooted environment, so
the default command line option of `-m /var/run/saslauthd` will
created a named socket that Postfix can't find. Changing it to
`/var/spool/postfix/var/run/saslauthd` puts the named socket in
Postfix's chrooted environment.

Fire up the `saslauthd` daemon by running:

    sudo /etc/init.d/saslauthd start

Now it's time to write the PAM script.

##### PAM Ruby script ####

PAM's `pam_exec.so` module makes it possible to create a script to
authenticate a user. It sets certain environment variables before
calling the script and sends the user's password via `STDIN` (when you
use the `expose_authtok` option for `pam_exec.so`).

Here's an example Ruby script:

{% highlight ruby %}
#!/usr/bin/env ruby
# PAM authenticator using pam_exec
require 'rubygems'
require 'bundler'

PAM_SUCCESS = 0  # Successful function return
PAM_SYSTEM_ERR = 4  # System error
PAM_AUTH_ERR = 7 # Authentication failure

ENV['BUNDLE_GEMFILE'] = File.join(File.dirname(__FILE__), "..", "Gemfile")
ENV['RACK_ENV'] = 'production'
Bundler.require
require File.join(File.dirname(__FILE__), "..", "lib", "foo")

user = ENV['PAM_USER']
password = $stdin.read.chomp("\000")

if Foo::User.authenticate(user, password)
  exit(PAM_SUCCESS)
else
  exit(PAM_AUTH_ERR)
end
{% endhighlight %}

In this example, `Foo` is the (hypothetical) name of a modular Sinatra
application. You can put this script in the `bin` directory of your
source tree. Make sure to make the script executable:

    chmod +x bin/pam_foo

Now, edit `/etc/pam.d/smtp` (which should be a new file) and add the
following:

    auth required pam_exec.so expose_authtok /var/rails/foo/current/bin/pam_foo
    account required pam_permit.so

This configures the `smtp` PAM service to use your script. The `auth`
line tells PAM to use the `pam_exec.so` module to run the script
located at `/var/rails/foo/current/bin/pam_foo` and to put the user's
password on `STDIN`. The password is null terminated, hence the
`chomp` in the above example script. The script exits with different
error codes depending on the outcome.

The `account` line tells PAM to allow anyone that authenticates
successfully (using the `pam_permit.so` module, which always allows
access). You can write a similar script to check access rights if you
wish.

Now that that's all done, it's time to test to make sure the
`saslauthd` daemon authenticates correctly.

##### Testing saslauthd ####

From the [Postfix manual](http://www.postfix.org/SASL_README.html#testing_saslauthd),
we learn about the `testsaslauthd` utility. You'll need to run this as
root or add yourself to the `sasl` group:

    sudo testsaslauthd -s smtp -f /var/spool/postfix/var/run/saslauthd/mux -u username -p password

Change `username` and `password` to both real and fake accounts to see
if it behaves properly.

Once it works the way you intend, it's time to configure Postfix
again.

##### Turn on Postfix SASL authentication ####

Add the `postfix` user to the `sasl` group so that it can access the
`saslauthd` socket:

    adduser postfix sasl

Add the following lines to `/etc/postfix/main.cf`:

    smtpd_sasl_auth_enable = yes
    smtpd_sasl_type = cyrus
    smtpd_sasl_path = smtpd
    smtpd_sasl_security_options = noanonymous
    broken_sasl_auth_clients = yes

    smtpd_recipient_restrictions =
      permit_sasl_authenticated,
      permit_mynetworks,
      reject_unauth_destination

You can tweak these settings if you want. Check out the
[postconf manual](http://www.postfix.org/postconf.5.html).

Next edit `/etc/postfix/sasl/smtpd.conf` and add the following:

    pwcheck_method: saslauthd
    mech_list: PLAIN LOGIN

`saslauthd` can only handle `PLAIN` and `LOGIN` authentication
methods, so you'll probably want to turn on SSL/TLS for Postfix. Check
out the [Postfix TLS Guide](http://www.postfix.org/TLS_README.html)
for details on how to do that.

##### Testing #####

Now it's time for testing! Restart Postfix:

    sudo /etc/init.d/postfix restart

On a client machine, you can use this Ruby script to test to see if it
all works (depends on the `mail` and `highline` gems):

{% highlight ruby %}
require 'rubygems'
require 'mail'
require 'highline/import'

address = ask("Domain: ")
port = ask("Port: ")
user = ask("Username: ")
password = ask("Password: ") { |q| q.echo = false }
recipient = ask("Recipient: ")

Mail.defaults do
  delivery_method :smtp, { :address              => address,
                           :port                 => port,
                           :user_name            => user,
                           :password             => password,
                           :authentication       => :plain,
                           :enable_starttls_auto => true  }
end

m = Mail.deliver do
  to recipient
  from "#{user}@#{address}"
  subject "Hey!"
  body "Hey buddy!"
end
{% endhighlight %}

This will test to make sure the authentication system works. If you
get the e-mail, it works! I suggest entering a bogus user/password as
well to test for failure.

That's it!

#### Further reading ####

* [Postfix Address Rewriting](http://www.postfix.org/ADDRESS_REWRITING_README.html)
* [Postfix SASL Howto](http://www.postfix.org/SASL_README.html)
* [Postfix TLS Howto](http://www.postfix.org/TLS_README.html)
* [PAM](http://content.hccfl.edu/pollock/AUnix2/PAM-Help.htm)
* [pam_exec](http://linux.die.net/man/8/pam_exec)
