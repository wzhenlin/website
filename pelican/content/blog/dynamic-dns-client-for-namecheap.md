---
Title: Dynamic DNS client for Namecheap using bash & cron
Date: 2015-02-26 23:21
Modified: 2015-10-06 22:05
Author: Carey Metcalfe
Tags:
 - shell script
 - code
 - linux
 - website
---

In addition to running this website, I also run a home server. For convenience,
I point a subdomain of `cmetcalfe.ca` at it so even though it's connected using
a dynamic IP (and actually seems to change fairly frequently), I can get access
to it from anywhere.

As a bit of background, the domain for this website is registered and managed
through [Namecheap][]. While they do provide a [recommended DDNS client][] for
keeping a domain's DNS updated, it only runs on Windows.

Instead, after [enabling DDNS for the domain] and reading [Namecheap's article
on using the browser to update DDNS] I came up with the following `dns-update`
script.

```bash
#!/bin/sh

dns=`dig <subdomain>.cmetcalfe.ca @resolver1.opendns.com +short`
curr=`dig myip.opendns.com @resolver1.opendns.com +short`
if [ $? -eq 0 ] && [ "$curr" != "" ] && [ "$dns" != "$curr" ]; then
    curl -s "https://dynamicdns.park-your-domain.com/update?host=<subdomain>&domain=cmetcalfe.ca&password=<my passkey>" | grep -q "<ErrCount>0</ErrCount>"
    if [ $? -eq 0 ]; then
        systemd-cat -t "`basename $0`" /usr/bin/echo "Server DNS record updated ($dns -> $curr)"
    else
        systemd-cat -t "`basename $0`" /usr/bin/echo "Server DNS record update FAILED (tried $dns -> $curr)"
    fi
fi
```

It basically checks if the IP returned by a DNS query for the subdomain matches
the current IP of the server (as reported by an [OpenDNS][] resolver) and if it
doesn't, sends a request to update the DNS. The `systemd-cat` commands are there
just to put some record of the IP changing into the syslog. Maybe I'll do some
analysis of it at some point.

From here, it's as easy as using [cron][] to run this script every 30 minutes
(`/30 * * * * /usr/local/bin/dns-update`) and I can be confident that my
subdomain will always be pointing at my home server when I need access to it.

!!! note
    A previous version of this article used `curl -sf http://curlmyip.com` to
    find the server's current IP address. However, after curlmyip went down for
    a few days, I decided to take the advice in [this StackExchange answer][]
    and use OpenDNS instead.

  [Namecheap]: http://namecheap.com
  [recommended DDNS client]: https://www.namecheap.com/support/knowledgebase/article.aspx/28
  [enabling DDNS for the domain]: https://www.namecheap.com/support/knowledgebase/article.aspx/595
  [Namecheap's article on using the browser to update DDNS]: https://www.namecheap.com/support/knowledgebase/article.aspx/29
  [OpenDNS]: https://en.wikipedia.org/wiki/OpenDNS
  [cron]: http://en.wikipedia.org/wiki/Cron
  [this StackExchange answer]: http://unix.stackexchange.com/a/81699
