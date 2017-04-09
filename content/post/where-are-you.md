+++
date = "2016-04-04T15:08:46-07:00"
title = "Where are you?"
tags = [  "AWS", "Python" ]
topics = [ "Projects" ]
+++

After a discussion with co-workers (and numerous plane flights from Seattle, WA to Iowa for weddings), they remarked that they never knew which weekends I'd be in Iowa. So the result was this website!

<figure class="center-text">
  <a href="http://isderbyiniowa.com"><img src="/img/where-are-you/iowa.png"></a>
  <figcaption>The horrible photoshopped icon that Facebook uses when shared.</figcaption>
</figure>

Powered by [AWS Route 53](https://aws.amazon.com/route53/), Static Website Hosting via [AWS S3](https://aws.amazon.com/s3/), and dynamic updates via [AWS Lambda](https://aws.amazon.com/lambda/) which periodically check my Google Calendar to update the site accordingly based on the presence of an event (in Iowa) or not (not in Iowa). It also comes with a convenient API (for no good reason): [http://isderbyiniowa.com/api/](http://isderbyiniowa.com/api/) The cool thing with this simple website is there are no backend servers maintained by me since it is all hosted via S3!

## Architecture

<figure class="center-text">
  <img src="/img/where-are-you/isderbyiniowa-aws-arch.png">
  <figcaption>What AWS Services are used to operate the website.</figcaption>
</figure>

The key thing to note is the power behind the site is Lambda, triggered by an hourly CloudWatch event. Everything else is standard work to set up a [static website in S3](http://docs.aws.amazon.com/gettingstarted/latest/swh/website-hosting-intro.html).

The bucket contents is pretty basic:

<pre>
- justinderby.com/
  - api/
    - no.html
    - yes.html
  - no.html
  - yes.html
</pre>

The trick to making this work is pointing the index and error pages in the S3 Static Website Hosting configuration to `yes.html` or `no.html`, instead of what most people set it to: `index.html`. Hitting the `api` subpage will also give you the correct JSON since the files within it mimic the `yes.html` and `no.html` structure. It doesn't vend HTML though :)

<figure class="center-text">
  <img src="/img/where-are-you/static-hosting-s3-config.png">
</figure>

This is the magic to this site - so now we need to find a way to dynamically change it...

## Making Static Actually Dynamic

When the lambda function is invoked, it contacts the Google Calendar API. It then looks to see, on a specific calendar I have, if there is an event going on. An event going on means that it will know I am in Iowa - I only put an event on this calendar when I actually am in Iowa. Based on what it determined, it then contacts AWS's S3 API and updates the `WebsiteConfiguration` map accordingly:

* In Iowa? `yes.html`
* Not in Iowa? `no.html`

*WARNING: Python code written in under 2 hours ahead!*

<pre>
<code class="language-python">from __future__ import print_function

import boto3
from datetime import datetime, timedelta
import json
import logging
import urllib
import urllib2

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Settings
# Google
calendarId = "FILL_ME_IN"
apiKey = "FILL_ME_IN"
# AWS
s3Bucket = "FILL_ME_IN"


def lambda_handler(event, context):
    logger.info('got event: {}'.format(event))

    # Timing stuff
    if event["time"]:
        now = datetime.strptime(event["time"], "%Y-%m-%dT%H:%M:%SZ")
        timeZoneOffset = "00:00"
    else:
        datetime.utcnow()
        timeZoneOffset = "07:00"
    tomorrowNow = now + timedelta(days=1)
    timeMin = "{0}T00:00:00-{1}".format(now.date(), timeZoneOffset)
    timeMax = "{0}T00:00:00-{1}".format(tomorrowNow.date(), timeZoneOffset)

    response = getEvents(timeMin, timeMax)

    location = None
    if "items" in response and len(response["items"]) > 0:
        firstEvent = response["items"][0]
        if "location" in response:
            logger.error("In Iowa, at {0}".format(firstEvent["location"]))
            location = firstEvent["location"]

        inIowa = True
    else:
        inIowa = False

    updateInIowa(inIowa)

    return {"result": {
            "inIowa": inIowa,
            "location": location,
            }}


def getEvents(timeMin, timeMax):
    logger.info("Grabbing events from {0} to {1}".format(timeMin, timeMax))

    url = "https://www.googleapis.com/calendar/v3/calendars/{0}/events?timeMax={1}&timeMin={2}&key={3}".format(
            urllib.quote(calendarId),
            urllib.quote(timeMax),
            urllib.quote(timeMin),
            urllib.quote(apiKey))

    data = urllib2.urlopen(url)
    response = data.read()
    logger.info(response)

    return json.loads(response)

def updateInIowa(yesNo):
    if yesNo == True:
        update = "yes.html"
    elif yesNo == False:
        update = "no.html"
    else:
        logger.error("Unknown value: {0}".format(yesNo))

    logger.info("Updating website to {0}".format(update))

    client = boto3.client('s3', region_name='us-east-1')
    response = client.put_bucket_website(
            Bucket=s3Bucket,
            WebsiteConfiguration={
                'ErrorDocument': {
                    'Key': update
                },
                'IndexDocument': {
                    'Suffix': update
                },
            }
        )
    logging.info("AWS S3 CLI Response: {0}".format(response))
</code>
</pre>

## Where To Go From Here

Currently, the bucket is not being fronted by [CloudFront](https://aws.amazon.com/cloudfront/). If the website got really popular, I would front it with a CloudFront distribution to cut down on costs. With it getting very few hits (usually when I send it out to continue to joke), it costs fractions of a penny to operate today.

## Ending Thoughts

For my safety, the people visiting the website only know that I'm in *Iowa* or not, but not the exact location.  That would get weird, fast...

[So...am I in Iowa?](http://isderbyiniowa.com)