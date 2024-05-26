---
title: Thomas DEVerson - NahamconCTF 2024
date: 2024-05-26
categories: [CTF Writeups]
tags: [CTF, NahamconCTF 2024]
---

## About the challenge

NahamCon CTF 2024

![](https://raw.githubusercontent.com/jmrcsnchz/jmrcsnchz.github.io/main/assets/nahamcon2024/Screenshot%202024-05-26%20124409.png)

The web application is running a python flask web server. On `/status`, we can see the total uptime of the application

![Screenshot 2024-05-25 132445](https://github.com/jmrcsnchz/jmrcsnchz.github.io/assets/77505889/bc86d13d-3f54-4323-b787-347ec5085bdd)

On `/backup`, partial source code was disclosed:

![Screenshot 2024-05-25 132524](https://github.com/jmrcsnchz/jmrcsnchz.github.io/assets/77505889/135b46f5-2fd7-46dd-bee2-e52b70d65efe)

According to the source code, the three allowed users were Jefferson, Madison, and Burr. Additionally, the flask secret is the concatenation of `THE_REYNOLDS_PAMPHLET-` and the current time when the application was initialized. According to the uptime, the application was initialized sometime during 1797.

The flask signed session format is seen below:

![image](https://github.com/jmrcsnchz/jmrcsnchz.github.io/assets/77505889/b31effb3-e7ba-4a92-8b97-eed64f9fb17f)


## Calculating the secret

If the secret was initialized at the same time the application went online in 1797, we could calculate it by subtracting the total uptime to the current time. I made the following python script that calculates the secret:

```python
from datetime import datetime
from dateutil.relativedelta import relativedelta
import requests

r = requests.get('https://challenge.nahamcon.com:32197/status')
data = r.text.split()

days = int(data[4])
hours = int(data[6])
minutes = int(data[8])

print((datetime.now() - relativedelta(days=days, hours=hours, minutes=minutes)).strftime("%Y%m%d%H%M"))
```

The output of the above script would consistently be 1797/08/15 16:45, which translates to `THE_REYNOLDS_PAMPHLET-179708251645` in secret format.

However, the calculated secret was incorrect and did not match the signature of the cookie. This was may be due to a delay on the application's shown uptime and the real date on when the secret was initialized.

So, I made a script that generates a wordlist of all possible secrets possible during the day 1797/08/25:

```python
x = 179708250000
for j in range(x, x+9999):
	with open("secrets.txt", "a") as secrets:
		secrets.write('THE_REYNOLDS_PAMPHLET-'+str(j)+"\n")
print('done')	

```

Using the generated wordlist, the secret key of the signed session was bruteforced using `flask-unsign` and yielded the following result:

![image](https://github.com/jmrcsnchz/jmrcsnchz.github.io/assets/77505889/f04c48dc-85ee-422a-a173-2abca586e32c)

The correct secret is `THE_REYNOLDS_PAMPHLET-179708250845`

There was an exact 8 hours interval from the initialization of secret and the application's supposed initialization date shown in its uptime

Using the now obtained secret, I forged a cookie that sets my user to Jefferson

![image](https://github.com/jmrcsnchz/jmrcsnchz.github.io/assets/77505889/2a118282-fd8b-4306-bd52-f9222c7c9f91)

Using the now forged cookie, I visited the `/messages` route to obtain the flag:

![image](https://github.com/jmrcsnchz/jmrcsnchz.github.io/assets/77505889/36c977c1-62f0-452e-ae93-85eba9dcf64f)


