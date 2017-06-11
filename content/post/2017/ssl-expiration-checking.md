---
categories:
- programming
comments: false
date: 2017-06-11T11:00:00-06:00
keywords:
- python
- aws
- lambda
- api gateway
- ssl
showPagination: false
summary: Put stuff here
tags:
- python
- aws
- lambda
- api gateway
- ssl
title: SSL Expiry Quick and Simple
thumbnailImagePosition: left
autoThumbnailImage: true
coverSize: partial
coverImage: https://c1.staticflickr.com/9/8297/7899423426_74a5510d2a_b.jpg
---

Ever have that feeling you are forgetting something right as you leave work? You are probably thinking about your keys or your lunch box but I am talking about your SSL certificate.  They don't last forever, we know this when we setup SSL but that doesn't stop it from sneaking up on us.  It has happened to the big guys like [Instagram](https://thenextweb.com/apps/2015/04/30/oops-instagram-forgot-to-renew-its-ssl-certificate/#.tnw_9kCFpT58) and [Google](http://www.pcworld.com/article/2906216/expired-google-certificate-temporarily-disrupts-gmail-service.html), at [Teem recently](http://status.teem.com/incidents/ddtys5ss4mw5), and of course for myself with my own home server.
<!--more-->

{{< image classes="fancybox center clear" src="https://media.giphy.com/media/joV1k1sNOT5xC/giphy.gif" title="Everything is fine">}}

I got tired of this and hacked together a tool to warn when a cert is going to expire: [ssl-expiry-check](https://github.com/LucasRoesler/ssl-expiry-check).

## The script
Checking SSL expiration is actually relatively simple, all of the information is public, you just need to grab it and validate it. My little project is a Python 3 project and uses only the standard library so that it is easy to use in any Python 3 environment, in particular AWS Lambda.

So, what does it take to validate an SSL cert?

```
import datetime
import logging
import socket
import ssl


logger = logging.getLogger('SSLVerify')


def ssl_expiry_datetime(hostname: str) -> datetime.datetime:
    ssl_date_fmt = r'%b %d %H:%M:%S %Y %Z'

    context = ssl.create_default_context()
    conn = context.wrap_socket(
        socket.socket(socket.AF_INET),
        server_hostname=hostname,
    )
    # 3 second timeout because Lambda has runtime limitations
    conn.settimeout(3.0)

    logger.debug('Connect to {}'.format(hostname))
    conn.connect((hostname, 443))
    ssl_info = conn.getpeercert()
    # parse the string from the certificate into a Python datetime object
    return datetime.datetime.strptime(ssl_info['notAfter'], ssl_date_fmt)


def ssl_valid_time_remaining(hostname: str) -> datetime.timedelta:
    """Get the number of days left in a cert's lifetime."""
    expires = ssl_expiry_datetime(hostname)
    logger.debug(
        'SSL cert for {} expires at {}'.format(
            hostname, expires.isoformat()
        )
    )
    return expires - datetime.datetime.utcnow()


def test_host(hostname: str, buffer_days: int=30) -> str:
    """Return test message for hostname cert expiration."""
    try:
        will_expire_in = ssl_valid_time_remaining(hostname)
    except ssl.CertificateError as e:
        return f'{hostname} cert error {e}'
    except ssl.SSLError as e:
        return f'{hostname} cert error {e}'
    except socket.timeout as e:
        return f'{hostname} could not connect'
    else:
        if will_expire_in < datetime.timedelta(days=0):
            return f'{hostname} cert will expired'
        elif will_expire_in < datetime.timedelta(days=buffer_days):
            return f'{hostname} cert will expire in {will_expire_in}'
        else:
            return f'{hostname} cert is fine'
```

At the core of this script is the `SSLSocket.getpeercert` method, using Python's `ssl` and `socket` modules, we connect to the provided `hostname` on port 443 and then `getpeercert` returns the SSL certificate as a dictionary. At this point you want the `notAfter` value.

For the `ssl-expiry` tool, we parse the `notAfter` value into a `timedelta` object because we want to know if the certificate will expire soon (defaults to 30 days).

If you checkout the source for `ssl-expiry` you can quickly check if a certificate is expiring soon using

```
$ echo "google.com\nfacebook.com" | python ssl_expiry.py
> google.com cert is fine
> facebook.com cert is fine
```

Depending on your environment, you may need to explicitly call to `python3`

```
$ echo "google.com\nfacebook.com" | python3 ssl_expiry.py
```

## SSL Expiry in Action
At this point, you could easily create a `cron` job to run this regularly and notify you if the output contains `"expire"`.  This is what I did for my home server. At Teem we use [New Relic](https://newrelic.com/) which provides an awesome tool called [Synthetics](https://newrelic.com/synthetics) that provides a cron-like tool that allows you to ping or GET against an API on regular intervals.  It will then trigger alerts that can be forwarded to your favorite on-call to (:cough:PagerDuty:cough:) if the request fails or the response does not contain an expected sub-string. With this tool available, I decided to deploy `ssl-expiry` as an AWS Lambda function an expose it via an AWS API Gateway api.

To simplify this deployment, I created a script specifically for Lambda, cleverly called [`ssl-expiry-lambda`](https://github.com/LucasRoesler/ssl-expiry-check/blob/master/ssl_expiry_lambda.py) This script simply wraps `ssl-expiry` so that the output is specialized to the format that Lambda and Gateway need so that I can

- Return a 200 if all of the requested hosts are valid for the next 30 days, or
- Return a 400 and the list of certificate test strings if at least one of the requested hosts is expired, will expire in the next 30 days, or is in some other way currently invalid.

With this setup, it is very easy to use New Relic Synthetics to alert the on-call developers to any SSL certificate issues.  This also implicitly alerts us if any of the domains are inaccessible.

I should note that New Relic Synthetics does have a "Validate SSL" option for any ping type checks that you configure.  However, this option does not include the preemptive "will expire soon" validation that I am looking for.

Configuring Lambda and API Gateway was actually more time consuming that writing the script, mostly due to trying to get the API response just right. To deploy to Lambda, create a zip that contains `ssl_expiry.py` and `ssl_expiry_lambda.py` and then follow the normal instructions to setup and configure a Lambda function. The `ssl_expiry_lambda` will use, if they exist, two env parameters:

- `HOSTLIST`: a comma separated string of `hostnames` to validate, and
- `EXPIRY_BUFFER`: an int that represents the days prior to expiration that the script will alert for, i.e. alert if the expiration is within `EXPIRY_BUFFER` days.

The entry point for the Lambda will be `ssl_expiry_lambda.main`.  Set the `HOSTLIST` env variable and you now have a default list of hostnames that will always be checked when calling this Lambda.

AWS API Gateway was the trickiest to configure. The important parts that are not obvious from the API Gateway admin UI are as follows:

You will need to create a new "Integration Response" for the exception that is raised when the check finds a failing or soon to fail certificate.

I configured this new Integration Response with a regex of

```
.*Cert Errors.*
```

and a "Body Mapping Template" with content type `application/json` and the template:

```
#set($inputRoot = $input.path('$'))
$input.path('$.errorMessage')
```

With this configuration, the exception raised by the `main` method will be parsed and returned as the body of the response. The HTTP status code will be a 400.

Additionally, in the "Method Request" section, I declared URL Query String Parameters for `host_list` and `expiry_buffer`.  This is helpful for testing the API, since the AWS UI will provide form fields for setting these parameters while testing it.

Finally, you should also define a "Method Response" for the 400 status. This can be left with all for the default empty values for response headers and response body.

With all of this in place, you now have an API that with a simple GET request, will validate the SSL certificates for a list of hostnames, including a warning for when certificates are about to expire. You can now sit back and relax knowing that you will be abruptly alerted as soon as any of your certificates are even close to expiring.

---

Thank you to
<a data-flickr-embed="true"  href="https://www.flickr.com/photos/cleanslatephotography/7899423426/in/photolist-d33BnC-5osrTL-9cuiGG-ehWXkg-nRdREH-ey8CR5-nRacoG-4qDmXP-a25CPx-8MqVJU-qmXRpL-dU81E6-oicDNZ-fu8kkk-2fvgQ1-qBtgR4-pEgwwC-hdKjjd-9Z4bzn-4kMSQK-qvyyF9-q3EyRX-9A6kY9-p5vhpk-TiDVoG-obiQrS-9uKVDe-fFjqjy-8z7vh2-o1ApPq-6vaRSf-e14EyV-cqWEpd-cvgD9f-cavamS-oKiPxn-9qeT5j-9ZMc4V-e6k5Ak-bD1SrM-qxMZWj-dJBsh9-c1Q6Fj-9oQTJP-jvWBA7-65A1Yh-qdLuiW-bq7vVh-brsYNu-bkuUnp" title="Unlocked">Sam Stockton</a> for the CC BY 2.0 image I used as the cover on this post.