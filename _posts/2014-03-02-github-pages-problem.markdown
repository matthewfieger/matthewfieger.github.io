---
layout: post
title:  "Hosting with Github Pages"
date:   2014-03-02 19:05:49
categories: posts me
tags: ["programming", "github pages", "jekyll"]
---

I have been hoping to write a tutorial explaining how to configure Github Pages but unfortunately I have been having some trouble with it.  Occasionally, the page and/or page resources fail to load.

To inspect the HTTP response codes, we can use cURL to send a HEAD request:

	~ matthewfieger$ curl -I "http://matthewfieger.com"
	curl: (56) Recv failure: Connection reset by peer

	~ matthewfieger$ curl -I "http://matthewfieger.com"
	curl: (56) Recv failure: Connection reset by peer

	~ matthewfieger$ curl -I "http://matthewfieger.com"
	HTTP/1.1 200 OK
	Server: GitHub.com
	Date: Sun, 02 Mar 2014 18:55:49 GMT
	Content-Type: text/html; charset=utf-8
	Connection: keep-alive
	Content-Length: 6299
	Last-Modified: Fri, 28 Feb 2014 18:35:52 GMT
	Expires: Sun, 02 Mar 2014 19:05:49 GMT
	Cache-Control: max-age=600
	Vary: Accept-Encoding
	Accept-Ranges: bytes
	Vary: Accept-Encoding

On some requests, the connection drops before the appropriate handshake can take place.  The site's DNS looks like this:

	20-c9-d0-7a-8f-93:~ matthewfieger$ dig matthewfieger.com

	; <<>> DiG 9.8.3-P1 <<>> matthewfieger.com
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1680
	;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

	;; QUESTION SECTION:
	;matthewfieger.com.		IN	A

	;; ANSWER SECTION:
	matthewfieger.com.	600	IN	A	192.30.252.154
	matthewfieger.com.	600	IN	A	192.30.252.153

	;; Query time: 35 msec
	;; SERVER: 155.247.225.230#53(155.247.225.230)
	;; WHEN: Tue Aug  5 12:11:55 2014
	;; MSG SIZE  rcvd: 67

I think my DNS is configured properly, so I'm not sure what the problem is.  I'll continue to investigate further.