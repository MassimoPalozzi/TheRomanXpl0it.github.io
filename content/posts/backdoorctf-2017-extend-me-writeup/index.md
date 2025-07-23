---
title: backdoorctf 2017 - extend-me Writeup
date: '2017-09-24'
lastmod: '2019-04-07T13:46:27+02:00'
categories:
- ctf_backdoorctf17
- writeup
- backdoorctf17
tags:
- web
authors:
- dp1
---

For this challenge we get access to a login prompt. This is the code on the backend, stripped of the useless parts:

```python

key = file('SECRET').read().strip()

@app.route('/login',methods = ['GET', 'POST'])
def login():
	if request.method == 'POST':
		if  not request.form.get('username'):
			return render_template('login.html')
		else:
			username = str(request.form.get('username'))
			if request.cookies.get('data') and request.cookies.get('user'):
				data = str(request.cookies.get('data')).decode('base64').strip()
				user = str(request.cookies.get('user')).decode('base64').strip()
				temp = '|'.join([key,username,user])
				if data != SLHA1(temp).digest():
					temp = SLHA1(temp).digest().encode('base64').strip().replace('\n','')
					resp = make_response(render_template('welcome_new.html',name = username))
					resp.set_cookie('user','user'.encode('base64').strip())
					resp.set_cookie('data',temp)
					return resp
				else:
					if 'admin' in user: # too lazy to check properly :p
						return "Here you go : CTF{XXXXXXXXXXXXXXXXXXXXXXXXX}"
					else:
						return render_template('welcome_back.html',name = username)
			else:
				resp = make_response(render_template('welcome_new.html',name = username))
				temp = '|'.join([key,username,'user'])
				resp.set_cookie('data',SLHA1(temp).digest().encode('base64').strip().replace('\n',''))
				resp.set_cookie('user','user'.encode('base64').strip())
				return resp

	else:
		return render_template('login.html')

```

It uses the username in the login form and the contents of two cookies, user and data, to authenticate the user, alongside with the contents of file SECRET, which we don't have access to.

The vulnerable check is after the SLHA1 algorithm - a custom hashing function - is called, since we can force the site to generate the right value for the data cookie for us.

Our goal here is to end up with SLHA1("$SECRET$|admin|admin") in data. To do so we can login a first time to get the cookies, set the contents of the user cookie to YWRtaW4=,
the base64 encoding for admin, and login a second time. This time the site will find that the contents of our data cookie are wrong and recalculate them based on our input, so
by setting once again the user cookie to YWRtaW4= and logging in a third time we finally get access to the flag.

Remember to login with the same username every time, otherwise the checks on data will fail.

Voilà


Reading other writeups, it seems that the intended solving method was hash extension (as even the challenge's name seems to suggest), yet I believe this cookie based solution to be much simpler and faster to execute. It also uses as its only tool any web browser that can edit cookies, or in my case Firefox with an extension to do the same.
