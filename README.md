# courier-mta-srs
A very simple and minimalist glue for Courier-MTA and SRS

## What is this?
This is the Github half of a [blog posting](http://matt.matzi.org.uk/2018/07/30/forwarding-external-mail-to-external-addresses-spf-srs-dkim-and-courier-mta/) over at matt.matzi.org.uk

### In brief:
Anyone who knows me knows how utterly frustrating I find the state of email today.

It's inherently insecure, absolutely relied upon, and there's always someone who "didn't get your email".

For larger businesses the offerings by Microsoft and the alike (Office365 for example) make sense.  For me however - it's Courier-MTA on a Debian box.  So when things go wrong, the buck stops with me.

(As an interesting aside: it seems even O365 is playing catch-up on this subject.  Writing this on the 30th July 2018: https://blogs.technet.microsoft.com/exchange/2018/06/15/sender-rewriting-scheme-srs-coming-to-office-365/ )

The "Sender Policy Framework" was an attempt to make email more secure and less abusable.  I assume you're already familiar with it - and that you're reading this because you, like me, are having issues forwarding 3rd party emails to 3rd party aliases.

Because the way Courier (and indeed most MTAs) handle aliases, an email address originating from external address user@gmail.com is received and then literally resent (fairly verbatim) to the external alias another@yahoo.co.uk.  When this happens, Yahoo observes the state of the email's SPF and correctly identifies that your forwarding mail server (plfc.org.uk in my case) is not an authorised sender for gmail.com.

SPF breaks the forwarding.

Aliases continue to work if the aliased address or the source sender are local.  It's specifically external to external that causes the issue.

Sender Rewriting Scheme is an (in my opinion) "attempt" to resolve this issue.  By clearly recognising that you are intentionally forwarding an email from another domain by kludging the originating address, SPF can pass while the forwarding can still continue.

Sadly it seems pretty well nothing natively supports SRS except Postfix (which even then requires manual compilation on several major Linux distributions today).  Courier-MTA is one of the least supported.

We've successfully patched around this by artificially gluing Debian's `srs` utility with Courier:

```
$ sudo dd if=/dev/urandom bs=1 count=32 2>/dev/null | base64 -w 0 > /etc/courier/srssecret
$ sudo chown root:mail /etc/courier/srssecret
$ sudo chmod 640 /etc/courier/srssecret

$ sudo apt-get install srs

$ cat /usr/local/bin/srs-forwarder 
#!/bin/bash
SRS_RETURN="$(/usr/bin/srs "$SENDER" --secretfile /etc/courier/srssecret --alias "$HOST")"
EMAIL="$(</dev/stdin)"
echo "$EMAIL" | /usr/sbin/sendmail -f "$SRS_RETURN" "$1"

$ sudo cat /etc/courier/aliases/system
srs-test: | /usr/local/bin/srs-forwarder someexternaladdress@example.com  ## Example SRS forwarded alias
```

Ensure that srs-forwarder is executable by Courier, and that `$SENDER` and `$HOST` are environment variables set by Courier when performing pipe-to-command aliases.  We're relying on Courier's implementation of sendmail for this, which is being used to rewrite the necessary SRS related headers (Return Path I believe).

Modified Courier alias entries can now re-queue and forward externally bound aliases successfully!

For a very lightly used mail server, the above is sufficient.  For anyone running a mail server with higher throughput, various improvements can be made which are included in this Github project.

## What about bounced/failed delivery notices?
In theory, I believe, a bounce message for a SRS forwarded email should be sent back to the kludged email address (that is, a fake email address belonging to your domain).

We can either capture these ourselves (i.e. forward them to `postmaster`) or pass them back to the true origin inserting a "default" handler in Courier's `aliasdir`:

```
$ cat /etc/courier/aliasdir/.courier-srsbounce-default
postmaster  ## to keep them ourselves ... OR:
| /usr/local/bin/srs-reverser  ## to forward the bounce back to the true origin
```

In this situation you would need to add a `srsbounce` prefix to the `srs-forwarder` script - see the `srs-forwarder` script for further details.  (It's a minor change.)

## Not yet implemented
In theory calling `srsd` (i.e. via the supplied `srsc` client) is significantly faster for higher-throughput mail servers.
