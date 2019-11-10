# [Hash me please](https://ringzer0ctf.com/challenges/13)

## Challenge

```
You have 2 seconds to hash this message using sha512 algorithm
Send the answer back using https://ringzer0ctf.com/challenges/13/[your_hash]

----- BEGIN MESSAGE -----
otK4DQOM8nl5p14mKMX01jQJjqynjy2xRCsli6Xhk9cA17MBJAsAK8aTpz7zX0XFsfQBcEJmDMNuJpWiQeJqdKat97SXXFsgL8INDh06TDrsTdBzibQlLQELOnyCTQIuPg8inZf7swqcARCISiTuZo5ECt6l9EGPLEYZt3WMpcPQUhoCq8WgnSKPbG1bbxQM2EBmyoYOqDuaLID1Hq8U8IAaerbfOSSHnkTLyHpPbJQMikEPADAzb0AgiBlWk4uxed8CKnhMXXp68UMyncYo3pubQGY0AinElm7WAfyn3Oj1yVqLYf0SulTaRI1iQfN2sKOSQd6JRfBg1RSQWIyhTihBQ9JxfnpxY4gE8cePiFWanEQadfhXnpo4pYrubHSZBYuz1yf9429iwQiAVpn9EC3TqlesSXijMDJD1PDWHC44jcu5sH4X9YHq9LISzR2bkBEch8ZOATJKW3FfBA2ApzRpbq7A7ZCir7kz59dwTN6GGCL73Dyi3gx5wvwukYCCWM1RM5evIb1fDCdw6CF0I3V4ohoz6R1StTA6OErnGjs9MwwIY2zwVkr9sGzpnq8HayDP3U2z4lzGHWfwOETzPaz8HYnVflsgKVVDGO3AZs7xecUTHDjmEJkcyyXEJfKj1vNxaGY0ZVo4Y8NvCXI6wS9UgWoQ2Y0TkDgka5aZQoUEny0PlyMHgLsnyG4qvU9FnfQnbRdR5YmjmcYxAA5HcnVATPQezPKNVq1W75E3TQc5T1tjsoRuCCVliCqIi1l3idQg8laR2cML36VmkDGM6sYeUfM27YWf1DmZOmHGokiig3urx14tjTx4YaWWYI2QbeGQqdnFnvOup9LM0F5aot4dtRZipRZrWv8czlINHm7WlIzbevcsO7w8Olh336lQsjSSvqv2DsPO1eQ5ATofQLeuXloQizxBJfj4wEW0XBEOGlK74ZcKAg4osi4BHs2hxbbUFXKtpf8WqJTkzWUZ3OdmX8NvqFCOHDzdrawHfutv4dGtZqjS5n5TmIeDeGhL
----- END MESSAGE -----
```

> Note: The message changes each time the page is refreshed.

## Observations

If we reload the page we see the message changes.The only way would could
possibly solve this challenge in the time allotted would be to automate it.

While there are many programming languages we could use, I chose Python.

Firstly we will need to somehow access the challenge page as if we are our
browser. Most websites track users via `cookies` stored in the browser. If
we open up the developer tools, we will see a cookie named `PHPSESSID`.

## The Script

Firstly, we import the modules we will need.

```py
from requests import Session
import hashlib
```

Then we initiate our `Session` and configure it to send up the proper
credentials so we appear logged in to the server.

```py
s = Session()

# rather than messing with `s.cookies`, since we only have
# one cookie, it's easier to just set the `cookie` header
# but this is not always the best or even a remotely good
# idea
s.headers.update({
    'cookie': 'PHPSESSID={YOUR PHPSESSID COOKIE VALUE GOES HERE}'
})
```

Next, we fetch the web page and parse out the message to be hashed.

```py
res = s.get('https://ringzer0ctf.com/challenges/13')

# although you won't be able to see them in a browser,
# before and after the message there exist `<br />` tags
# followed by whitespace which is what the `replace` and 
# `strip` calls are for
msg = res.text\
         .split('----- BEGIN MESSAGE -----')[1]\
         .split('----- END MESSAGE -----')[0]\
         .replace('<br />', '')\
         .strip()
```

> Note: rather than using RegEx or an HTML parser like BeautifulSoup
> we can simply hack a very simple parser using `<str>.split` with
> `<str>.replace` and `<str>.strip` to tidy it up a little which saves
> us the overhead of the former and a little bit of time.

Almost done! Now we use `sha512` to hash the message and send it back to the server.

```py
hash = hashlib.sha512(msg).hexdigest()

res = s.post('https://ringzer0ctf.com/challenges/13/' + hash)
```

Finally, to save you a little time looking for the flag in the HTML...
Let's automate it!

```py
flag = 'FLAG-' + res.text\
          .split('FLAG-')[1]\
          .split('<')[0]

print(flag)
```

## Putting It All Together

```py
from requests import Session
import hashlib

s = Session()

s.headers.update({
    'cookie': 'PHPSESSID={YOUR PHPSESSID COOKIE VALUE GOES HERE}'
})

res = s.get('https://ringzer0ctf.com/challenges/13')

msg = res.text\
        .split('----- BEGIN MESSAGE -----')[1]\
        .split('----- END MESSAGE -----')[0]\
        .replace('<br />', '')\
        .strip()

hash = hashlib.sha512(msg).hexdigest()

res = s.post('https://ringzer0ctf.com/challenges/13/' + hash)

flag = 'FLAG-' + res.text\
          .split('FLAG-')[1]\
          .split('<')[0]

print(flag)
```
