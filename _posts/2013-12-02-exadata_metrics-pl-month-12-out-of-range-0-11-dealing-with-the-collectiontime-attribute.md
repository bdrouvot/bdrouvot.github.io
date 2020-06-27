---
layout: post
title: 'exadata_metrics.pl: "Month ''12'' out of range 0..11" dealing with the collectionTime
  attribute'
date: 2013-12-02 07:24:05.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
- Perl Scripts
- Presentations
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/0onsTbOAcI
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5813160432374284288&type=U&a=fEPO
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/12/02/exadata_metrics-pl-month-12-out-of-range-0-11-dealing-with-the-collectiontime-attribute/"
---

Yesterday was the first of December and I detected an issue with the [exadata\_metrics.pl](http://bdrouvot.wordpress.com/2013/11/01/exadata_metrics-pl-new-feature-to-collect-real-time-metrics-extracted-from-cumulative-metrics/ "exadata_metrics.pl: New feature to collect real-time metrics extracted from cumulative metrics") script when dealing with the [collectionTime attribute](http://bdrouvot.wordpress.com/2013/09/13/exadata-cell-metrics-collectiontime-attribute-something-that-matters/ "Exadata Cell metrics: collectionTime attribute, something that matters").

<span style="text-decoration:underline;">It produced:</span>

    Month '12' out of range 0..11 at ./exadata_metrics.pl line 255

This is due to the fact that when using perl localtime/timegm the valid range for a month is 0-11 with 0 indicating January and 11 indicating December (while the collectiontime attribute is using 1-12)

The exadata\_metrics.pl script **has been updated to take care of this rule** (It can be downloaded from [this repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit)).

<span style="text-decoration:underline;">**Remarks:**</span>

1.  Without this update the script did not produce wrong values for the DELTA(s) field (It was simply not possible to launch the script during December).
2.  Guess when I discovered the issue? When I was speaking about my [exadata\_metrics.pl](http://bdrouvot.wordpress.com/2013/11/01/exadata_metrics-pl-new-feature-to-collect-real-time-metrics-extracted-from-cumulative-metrics/ "exadata_metrics.pl: New feature to collect real-time metrics extracted from cumulative metrics") script during the [UKOUG TECH13](http://www.tech13.ukoug.org/) conference. Live demo never works as expected :-), but with the help of some attendees (Big thanks to [Martin Nash](https://twitter.com/mpnsh)) we managed to bypass the issue during the live demo.
