---
layout: post
title: User-Agent有趣的历史
tags: [WebKit]
keywords: UA, user-agent, useragent
description: user-agent有趣的历史
comments: true
share: true
---

In the beginning there was [NCSA Mosaic](http://www.ncsa.illinois.edu/enabling), and Mosaic called itself NCSA_Mosaic/2.0 (Windows 3.1), and Mosaic displayed pictures along with text, and there was much rejoicing.  

And behold, then came a new web browser known as [“Mozilla”](http://en.wikipedia.org/wiki/Mozilla), being short for “Mosaic Killer,” but Mosaic was not amused, so the public name was changed to Netscape, and Netscape called itself Mozilla/1.0 (Win3.1), and there was more rejoicing. And Netscape supported frames, and frames became popular among the people, but Mosaic did not support frames, and so came “user agent sniffing” and to “Mozilla” webmasters sent frames, but to other browsers they sent not frames.   

And Netscape said, let us make fun of Microsoft and refer to Windows as “poorly debugged device drivers,” and Microsoft was angry. And so Microsoft made their own web browser, which they called Internet Explorer, hoping for it to be a “Netscape Killer”. And Internet Explorer supported frames, and yet was not Mozilla, and so was not given frames. And Microsoft grew impatient, and did not wish to wait for webmasters to learn of IE and begin to send it frames, and so Internet Explorer declared that it was “Mozilla compatible” and began to impersonate Netscape, and called itself Mozilla/1.22 (compatible; MSIE 2.0; Windows 95), and Internet Explorer received frames, and all of Microsoft was happy, but webmasters were confused.    

And Microsoft sold IE with Windows, and made it better than Netscape, and the first browser war raged upon the face of the land. And behold, Netscape was killed, and there was much rejoicing at Microsoft. But Netscape was reborn as Mozilla, and Mozilla built Gecko, and called itself Mozilla/5.0 (Windows; U; Windows NT 5.0; en-US; rv:1.1) Gecko/20020826, and Gecko was the rendering engine, and Gecko was good. And Mozilla became Firefox, and called itself Mozilla/5.0 (Windows; U; Windows NT 5.1; sv-SE; rv:1.7.5) Gecko/20041108 Firefox/1.0, and Firefox was very good. And Gecko began to multiply, and other browsers were born that used its code, and they called themselves Mozilla/5.0 (Macintosh; U; PPC Mac OS X Mach-O; en-US; rv:1.7.2) Gecko/20040825 Camino/0.8.1 the one, and Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.8.1.8) Gecko/20071008 SeaMonkey/1.0 another, each pretending to be Mozilla, and all of them powered by Gecko.    

And Gecko was good, and IE was not, and sniffing was reborn, and Gecko was given good web code, and other browsers were not. And the followers of Linux were much sorrowed, because they had built Konqueror, whose engine was KHTML, which they thought was as good as Gecko, but it was not Gecko, and so was not given the good pages, and so Konquerer began to pretend to be “like Gecko” to get the good pages, and called itself Mozilla/5.0 (compatible; Konqueror/3.2; FreeBSD) (KHTML, like Gecko) and there was much confusion.    

Then cometh Opera and said, “surely we should allow our users to decide which browser we should impersonate,” and so Opera created a menu item, and Opera called itself Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; en) Opera 9.51, or Mozilla/5.0 (Windows NT 6.0; U; en; rv:1.8.1) Gecko/20061208 Firefox/2.0.0 Opera 9.51, or Opera/9.51 (Windows NT 5.1; U; en) depending on which option the user selected.     

And Apple built Safari, and used KHTML, but added many features, and forked the project, and called it WebKit, but wanted pages written for KHTML, and so Safari called itself Mozilla/5.0 (Macintosh; U; PPC Mac OS X; de-de) AppleWebKit/85.7 (KHTML, like Gecko) Safari/85.5, and it got worse.    

And Microsoft feared Firefox greatly, and Internet Explorer returned, and called itself Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.0) and it rendered good code, but only if webmasters commanded it to do so.    

And then Google built [Chrome](https://www.google.com/chrome/browser/), and Chrome used Webkit, and it was like Safari, and wanted pages built for Safari, and so pretended to be Safari. And thus Chrome used WebKit, and pretended to be Safari, and WebKit pretended to be KHTML, and KHTML pretended to be Gecko, and all browsers pretended to be Mozilla, and Chrome called itself Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US) AppleWebKit/525.13 (KHTML, like Gecko) Chrome/0.2.149.27 Safari/525.13, and the user agent string was a complete mess, and near useless, and everyone pretended to be everyone else, and confusion abounded.



<div style="display:none" itemscope itemtype="http://schema.org/Product">
   <span itemprop="name">fenesky</span>
   <div itemprop="aggregateRating" itemscope itemtype="http://schema.org/AggregateRating">
     Rated <span itemprop="ratingValue">4.8</span>/5 based on <span itemprop="reviewCount">10</span> reviews
</div>