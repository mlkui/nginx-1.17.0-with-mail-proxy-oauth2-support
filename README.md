# nginx-1.17.0-with-mail-proxy-oauth2-support
Add SASL XOAUTH2 Bearer token(RFC 6750) support to POP3, IMAP and SMTP protocol for Nginx mail module

Nowadays, OATUH authentication mechanism has became a fundamental need in many scenarios, especially for enterprise users. Many modern mail service providers, like Google, have supported OAuth 2.0. However, the ngx_mail_proxy_module and ngx_mail_auth_http_module provide no support of XOAUTH2. Google's document(https://developers.google.com/gmail/imap/xoauth2-protocol) is a good place to learn the detail of this mechanism.

The SASL XOAUTH2 initial client response has the following format:
base64("user=" {User} "^Aauth=Bearer " {Access Token} "^A^A")
using the base64 encoding mechanism defined in RFC 4648. ^A represents a Control+A (\001).

For example, before base64-encoding, the initial client response might look like this:
user=someuser@example.com^Aauth=Bearer ya29.vF9dft4qmTc2Nvb3RlckBhdHRhdmlzdGEuY29tCg^A^A
After base64-encoding, it becomes:
dXNlcj1zb21ldXNlckBleGFtcGxlLmNvbQFhdXRoPUJlYXJlciB5YTI5LnZGOWRmdDRxbVRjMk52YjNSbGNrQmhkSFJoZG1semRHRXVZMjl0Q2cBAQ==

If we take IMAP for example, the AUTHENTICATE will be something like:
A0 AUTHENTICATE XOAUTH2 dXNlcj1zb21ldXNlckBleGFtcGxlLmNvbQFhdXRoPUJlYXJlciB5YTI5LnZGOWRmdDRxbVRjMk52YjNSbGNrQmhkSFJoZG1semRHRXVZMjl0Q2cBAQ==

Firstly, to support AUTHENTICATE XOAUTH2 command, we have to modify ngx_mail_imap_module, ngx_mail_pop3_module and ngx_mail_smtp_module to let them recognize and act to corresponding directives like 'imap_auth xoauth2', 'smtp_auth xoauth2' and 'pop3_auth xoauth2'. After the modification, the config might look like this:
mail
{
    auth_http http://127.0.0.1:8008/auth.do;

    server
    {
    	listen     0.0.0.0:243;
    	protocol   imap;
    	proxy      on;
    	imap_auth  XOAUTH2;
    }
}

Also, we have to modify the authentication procedure of the upstream proxying to pass the token to upstream mail server. To achieve this goal, handlers of oauth should be added to ngx_mail_proxy_module.

At last, we have to modify the ngx_mail_auth_http_module to extract the user information sent from the client and pass them to the http authentication server. The http request to http authentication service might look like this:
GET /auth.do HTTP/1.0
Host: 127.0.0.1
Auth-Method: oauth
Auth-User: someuser@example.com
Auth-Pass: dXNlcj1zb21ldXNlckBleGFtcGxlLmNvbQFhdXRoPUJlYXJlciB5YTI5LnZGOWRmdDRxbVRjMk52YjNSbGNrQmhkSFJoZG1semRHRXVZMjl0Q2cBAQ==
Auth-Protocol: imap
Auth-Login-Attempt: 1
Client-IP: 127.0.0.1
The http authentication service can then use corresponding headers to verify the request comes from Nginx.

The patch is based on the latest Nginx 1.17.0 mainline version.