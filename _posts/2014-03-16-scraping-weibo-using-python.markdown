---
layout: post
title: "Scraping Data from Sina Weibo using Python"
date: 2014-03-16 12:01
comments: true
categories: 
- python
tags:
- weibo
- diffusion
---

![](http://weblab.com.cityu.edu.hk/blog/chengjun/files/2012/09/33.png)

### Weibo Oauth2.0

I would like to introduce you how to use python to scrape tweets from Sina Weibo in this post. 

#### Automatically get authorization by oauth2.0

First, following this webpage to set your app, especially to get the app key, app secret, and set the callback url.  See an [introduction](http://blog.laisky.us/2012/01/278/).  

Second, following this [link](http://www.how2dns.com/blog/?p=538), you can automatically get the code from the callback url. 

The following python script demonstrates how to automatically get authorization. 

	#!/usr/bin/env python
	# -*- coding: utf8 -*-
	
	from weibo import APIClient
	import urllib2
	import urllib
	import sys
	import time
	from time import clock
	import csv
	import random
	
	reload(sys)
	sys.setdefaultencoding('utf-8')
	
	'''Step 0 Login with OAuth2.0'''
	if __name__ == "__main__":
		APP_KEY = '663...' # app key
		APP_SECRET = '2fc....' # app secret
		CALLBACK_URL = 'https://api.weibo.com/oauth2/default.html' # set callback url exactly like this!
		AUTH_URL = 'https://api.weibo.com/oauth2/authorize'
		USERID = 'w...4' # your weibo user id
		PASSWD = 'w....' #your pw
	
		client = APIClient(app_key=APP_KEY, app_secret=APP_SECRET, redirect_uri=CALLBACK_URL)
		referer_url = client.get_authorize_url()
		print "referer url is : %s" % referer_url
	
		cookies = urllib2.HTTPCookieProcessor()
		opener = urllib2.build_opener(cookies)
		urllib2.install_opener(opener)
	
		postdata = {"client_id": APP_KEY,
					"redirect_uri": CALLBACK_URL,
					"userId": USERID,
					"passwd": PASSWD,
					"isLoginSina": "0",
					"action": "submit",
					"response_type": "code",
					}
		headers = {"User-Agent": "Mozilla/5.0 (Windows NT 6.1; rv:11.0) Gecko/20100101 Firefox/11.0",
					"Host": "api.weibo.com",
					"Referer": referer_url
				}
	
		req  = urllib2.Request(
		   url = AUTH_URL,
		   data = urllib.urlencode(postdata),
		   headers = headers
		   )
		try:
			resp = urllib2.urlopen(req)
			print "callback url is : %s" % resp.geturl()
			code = resp.geturl()[-32:]
			print "code is : %s" %  code
		except Exception, e:
			print e
	
	r = client.request_access_token(code)
	access_token1 = r.access_token # The token return by sina
	expires_in = r.expires_in
	
	print "access_token=" ,access_token1
	print "expires_in=" ,expires_in   # access_token lifetime by second. http://open.weibo.com/wiki/OAuth2/access_token
	
	"""save the access token"""
	client.set_access_token(access_token1, expires_in)

### Get the number of retweets
Step 1. Assume that you have had a list of tweet ids, you want to get the number of repsots. Thus, you can have the distribution of the size of diffusion.

	''' Step 1 Get the number of reposts'''
	"""get the user ids"""
	dataReader = csv.reader(open('C:/Python27/weibo/sampledRtIds2.csv', 'r'), delimiter=',', quotechar='|')
	ids = []
	for row in dataReader:
	    ids.append(int(row[0]))  # modify the number to get the diffusers' ids
	
	file = open("C:/Python27/weibo/repostsRT300000m2.csv",'wb') # save to csv file
	
	start = clock()
	print start
	
	for seqNum in range(1500, 2999):
		id = ids[(0 + 100*seqNum) : (100+100*seqNum)]
		id = str(id).strip('[]').replace('L', '')
		rate = client.get.account__rate_limit_status()
		sleep_time = rate.reset_time_in_seconds + 300
		remaining_ip_hits = rate.remaining_ip_hits
		remaining_user_hits = rate.remaining_user_hits
		if remaining_ip_hits >= 10 and remaining_user_hits >= 5:
			rtc = client.get.statuses__count(ids = id) # mid, 100
			for n in range(0, len(rtc)): # 0-99
				mid = rtc[n]['id']
				reposts = rtc[n]['reposts']
				comments = rtc[n]['comments']
				attitudes = rtc[n]['attitudes']
				timePass = clock()-start
				if round(timePass) % 10 == 0:
					print mid, reposts, len(rtc), "I have been working for %s seconds" % round(timePass)
				print >>file, "%s,%s,%s,%s" % (mid, reposts, comments, attitudes)
		elif remaining_ip_hits < 10 or remaining_user_hits < 5:
			print "Python will sleep %s seconds" % sleep_time
			time.sleep(sleep_time+60)
	
	file.close()

### Get the list of diffusers

Step 2. If you want to step further and get the list of diffusers for a list of weibos. Thus, you will know how many reposts or retweets have been deleted by the website.

	# '''Step 2 Get the diffusers'''
	"""read ids"""
	dataReader = csv.reader(open('C:/Python27/weibo/repostsSample3.csv', 'r'), delimiter=',', quotechar='|')
	ids = []
	for row in dataReader:
	    ids.append(int(row[1]))  # get the number to get the mid
	
	addressForSavingData= "C:/Python27/weibo/diffsersSave.csv"
	file = open(addressForSavingData,'wb') # save to csv file
	
	start = clock()
	print start
	
	lenid = len(ids) # lenid = 8 # test with the first two cases
	
	for n in range(0, lenid+1):  # the 78 should be 77 here
		rate = client.get.account__rate_limit_status()
		sleep_time = rate.reset_time_in_seconds + 300
		remaining_ip_hits = rate.remaining_ip_hits
		remaining_user_hits = rate.remaining_user_hits
		if remaining_ip_hits >= 10 and remaining_user_hits >= 3:
			if reposts[n]%200 == 0:
				pages = reposts[n]/200
			else:
				pages = reposts[n]/200 + 1
			try:
				for pageNum in range(1, pages + 1):
					r = client.get.statuses__repost_timeline(id = ids[n], page = pageNum, count = 200)
					if len(r) == 0:
						pass
					else:
						m = int(len(r['reposts']))
						for i in range(0, m):
							"""1.1 reposts"""
							mid = r['reposts'][i].id
							text = r['reposts'][i].text.replace(",", "")
							created = r['reposts'][i].created_at
							"""1.2 reposts.user"""
							user = r['reposts'][i].user
							user_id = user.id
							user_name = user.name
							user_province = user.province
							user_city = user.city
							user_gender = user.gender
							user_url = user.url
							user_followers = user.followers_count
							user_bifollowers = user.bi_followers_count
							user_friends = user.friends_count
							user_statuses = user.statuses_count
							user_created = user.created_at
							user_verified = user.verified
							"""2.1 retweeted_status"""
							rts = r['reposts'][i].retweeted_status
							rts_mid = rts.id
							rts_text = rts.text.replace(",", "")
							rts_created = rts.created_at
							"""2.2 retweeted_status.user"""
							rtsuser_id = rts.user.id
							rtsuser_name = rts.user.name
							rtsuser_province = rts.user.province
							rtsuser_city = rts.user.city
							rtsuser_gender = rts.user.gender
							rtsuser_url = rts.user.url
							rtsuser_followers = rts.user.followers_count
							rtsuser_bifollowers = rts.user.bi_followers_count
							rtsuser_friends = rts.user.friends_count
							rtsuser_statuses = rts.user.statuses_count
							rtsuser_created = rts.user.created_at
							rtsuser_verified = rts.user.verified
							timePass = clock()-start
							if round(timePass) % 10 == 0:
								print mid, rts_mid, "I have been working for %s seconds" % round(timePass)
								time.sleep( random.randrange(3, 9, 1) )  # To avoid http error 504 gateway time-out
							print >>file, "%s,'%s','%s',%s,'%s',%s,%s,%s,'%s',%s,%s,%s,'%s',%s,%s,'%s',%s,'%s',%s,%s,%s,'%s',%s,%s,%s,%s,%s"  % (mid, created, text, # 3 # "%s,%s,|%s|,%s,|%s|,%s,%s,%s,|%s|,%s,%s,%s,%s,%s,%s,%s,%s,|%s|,%s,%s,%s,|%s|,%s,%s,%s,%s,%s" % (mid, created, text, # 3
												user_id, user_name, user_province, user_city, user_gender,  # 5 --> 5
												user_url, user_followers, user_friends, user_statuses, user_created, user_verified,  # rts_text, # 6 --> 9
												rts_mid, rts_created, # 2
												rtsuser_id, rtsuser_name, rtsuser_province, rtsuser_city, rtsuser_gender, # 5 --> 18
												rtsuser_url, rtsuser_followers, rtsuser_friends, rtsuser_statuses, rtsuser_created, rtsuser_verified)  # 6  --> 22
			except Exception, e:
				print >> sys.stderr, 'Encountered Exception:', e, ids[n]
				time.sleep(120)
				pass
		elif remaining_ip_hits < 10 or remaining_user_hits < 3:
			print "Python will sleep %s seconds" % sleep_time
			time.sleep(sleep_time+60)
	
	file.close()


### Get the following relationships

Step 3. Now, you may want to get the social graph for all the diffusers.

	'''Step 3 Get the social graph'''
	"""read ids"""
	dataReader = csv.reader(open('C:/Python27/weibo/SocialGraphIdsForStepThree.csv', 'r'), delimiter=',', quotechar='|')
	ids = []
	for row in dataReader:
	    ids.append(int(row[0]))  # get the number to get the mid
	
	ids = ids[188648:697060]
	
	addressForSavingData= "C:/Python27/weibo/socialgraphSave142_2.csv"
	file = open(addressForSavingData,'wb') # save to csv file
	
	addressForSavingError = "C:/Python27/weibo/socialgraphSaveError142_2.csv"
	errorlog = open(addressForSavingError,'w')
	errorlog.close()
	
	start = clock()
	print start
	
	for id in ids:
		try:
			rate = client.get.account__rate_limit_status()
			sleep_time = rate.reset_time_in_seconds + 300
			remaining_ip_hits = rate.remaining_ip_hits
			remaining_user_hits = rate.remaining_user_hits
			if remaining_ip_hits >= 10 and remaining_user_hits >= 3:
				cursor = -1
				fids=[]
				while cursor != 0:
					response = client.get.friendships__friends__ids(uid=id, count= 5000, cursor=cursor)  # the biggest count is 5000
					fids	+= response.ids
					cursor = response.next_cursor # previousCursor = response.previous_cursor
					timePass = clock()-start
					if round(timePass) % 10 == 0:
						print id, "I have been working for %s seconds" % round(timePass)
						# time.sleep( 0.01 * random.randrange(0, 5, 1) )  # To avoid http error 504 gateway time-out
					if cursor == 0:
						totalNum = response.total_number
						for fid in fids:
							print >>file, "%s,%s,%s"  % (id, fid, totalNum)
						break
			elif remaining_ip_hits < 10 or remaining_user_hits < 3:
				print "Python will sleep %s seconds" % sleep_time
				time.sleep(sleep_time+60)
		except Exception, e:
			print >>sys.stderr, 'Encountered Exception:', e, id
			errorlog = open(addressForSavingError, 'a')
			print >>errorlog, "%s,%s"  % (id, e)
			errorlog.close()
			print 'When the error happens, the id is:', id
			time.sleep(60)
			pass
	
	file.close()


### Get the diffusion network

Step 4. Given the collected data of retweets, we can get the diffusion path by parsing the text of weibo.

	import re
	import sys
	from time import clock
	
	reload(sys)
	sys.setdefaultencoding('utf-8')
	
	'''
	Convert "Thu Aug 04 11:39:32 +0800 2011" to the ISO format: YYYY-MM-DD H:M:S
	Refer to: http://stackoverflow.com/questions/15727510/using-python-regex-to-identify-retweeters-from-tweets-with-chinese-characters
	'''
	
	file = open("D:/chengjun/New/repostsReSampleClean.csv", 'r')
	lines = file.readlines()
	
	addressForSavingData= "D:/chengjun/New/diffusion_path6.csv"  
	file = open(addressForSavingData,'wb') # save to csv file 
	
	addressForSavingError = "D:/chengjun/New/Error.csv"  
	errorlog = open(addressForSavingError,'w')
	errorlog.close
	
	start = clock()  
	print start
		
	for line in lines:
		list = line.split(',')
		rtsmid = list[15].strip()  #rmid
		userName = list[5].strip().replace("'", "") # username
		submitterName = list[18].strip().replace("'", "")
		tweet = list[3].replace(',','')
		RTpattern = r'''//?@(\w+)'''
		rt = re.findall(RTpattern, tweet.decode("utf-8"), re.UNICODE)
		if rt == None or len(rt)==0:
			target = userName
			source = submitterName
			print >>file, "%s,%s,%s"  % (rtsmid, source, target) 
		elif rt != None and len(rt) != 0:
			rt.insert(0, userName) # 
			for i in xrange(len(rt) - 1):
				target = rt[i].encode('utf-8')
				source = rt[i + 1].encode('utf-8')
				print >>file, "%s,%s,%s"  % (rtsmid, source, target) 
