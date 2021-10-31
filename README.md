KHPH
====
Keep Honest People Honest - String Obfuscation, Storage, & Retrieval

Disclaimer
==========

This module scrambles a string to help keep it private, but of course the scrambled string is inherently vulnerable to being unscrambled by someone other than the owner.  One might ask, "Why even bother employing a scrambling function?"  The pragmatic answer is that simply masking sensitive data from view can prevent many of the exposure scenarios that exist in the real world.

Consider the egregious case where you need to run a program that absolutely requires you to include a *password* in its command line invocation.

    myuid@myserver> /usr/local/bin/srvrconn -acct=USER72 -password=pAsSwOrD57! START INSTANCE ABC

If you run it interactively, your shell history will record the entire command line for posterity, including the exposed password.  Then your system backup will make a copy of that, and who knows where that goes and for how long?  If it were executed via a job scheduler, the password could be exposed in multiple places: crontab, logs, email, backups, etc.

You might consider a solution where you put the secret characters in a file and judiciously apply DAC controls to restrict access (chown/chgrp/chmod).  When it's time to use the password, you could read the secret string from the file and insert it where needed.  But **root** would be able to look at your secret with a quick `cat` command, and then your secret wouldn't be a secret anymore.

This module offers you a way to reduce the likelihood of baring your secret information to curious people who are just poking around.  It helps reduce the number of surfaces where your private data is openly exposed.  It does not purport to fully protect your private information from prying eyes, rather to make it opaque to glances.

> ALWAYS ENCRYPT CUSTOMER DATA.  Customer data and other types of sensitive information warrant real security, not a privacy fence.

Description
===========

This module will scramble a string, stash it wherever you specify, then expose it to you whole again when you ask for it.  **root** can’t expose it directly, unless **root** originally stored it.  `su`’ing into the owner’s account from a different account won’t expose it directly either.  It’s not in the direct line of sight by anyone other than the owner.

Synopsis
========

```raku
use KHPH;

my $userid = 'testid';
my KHPH $secret-string .= new(
    :herald('myapp credentials'),
    :prompt($userid ~ ' password'),
    :stash-path('/tmp/myapp/' ~ $userid ~ '/mysecret'),
    :user-exclusive-at('/tmp/myapp/' ~ $userid);
);
say $secret-string.expose;
```

Methods
=======

.new()
------
Generate a KHPH object

#### :herald?

  * Optional announcement used only when interactively stashing the secret.

#### :prompt?

  * Optional prompt string used only when interactively stashing the secret.

#### :secret?

  * Optionally send the constructor the secret string. No prompting will occur.

#### :stash-path!

  * Specify the path (directories/file) to create or find the stash file.  Always include a subdirectory in the path, as KHPH will `chmod` the directory containing the stash file.

#### :user-exclusive-at?

  * Optionally specify a segment of the :stash-path to exclude all group & other access (0700).

.expose()
---------
Return the secret as a clear-text Str.

Example I
=========

The `myapp-pass.raku` script will manage the password stash of `myapp`.  Run it interactively one time to stash your secret, then you (not someone else) can run it anytime to expose the secret.

The `myapp-pass.raku` script:

```raku
#!/usr/bin/env raku
use KHPH;
KHPH.new(:stash-path('/tmp/.myapp/password.khph')).expose.print;
```

Run ~/myapp-pass.raku once interactively to stash the secret:

    me@mysystem> ~/myapp-pass.raku && echo
    [1/2] Enter secret> aW3S0m3pA55w0rDI'LlN3VeRr3m3mB3R
    [2/2] Enter secret> aW3S0m3pA55w0rDI'LlN3VeRr3m3mB3R
    aW3S0m3pA55w0rDI'LlN3VeRr3m3mB3R
    me@mysystem>

> _Notice how the script dumps the secret when you personally run it?  Have someone else log into the same system, have them run the same script, and see what they get.  Have them `su` to your account and try again.  Have them log in as **root** and give it a go.  Have them `su` from **root** into your account and try.  Have them `sudo su -` into your account and try again._

Then in your application client:

    me@mysystem> /usr/bin/dsmadmc -id=MYSELF -password=`~/myapp-pass.raku` QUERY SESSION FORMAT=DETAILED

The password will be inserted into the command line and authentication will succeed.

> __Note__:  _The above example demonstrates a particular application client (familiar to some backup admins) that is more helpful than most, in that it re-writes the process' args after the program launches.  `ps` will only display the string `-password=*******` instead of the actual password string.  Not all application vendors pay attention to such details, so beware -- `ps` could be displaying the secret despite your efforts to protect it._

Example II
==========

When crafting REST API clients, servers will often issue session tokens for subsequent connections.  These authenticating session tokens remain valid for long intervals of time (hours, days, weeks) and should be protected like passwords.  When stashing a token locally for reuse, minimally use KHPH instead of clear-text so that it isn't easily viewed by passersby.

Usage Recommendation
====================

Since the intent of using this module is to obfuscate, it is recommended to specify a :stash-path that doesn't indicate what's being stored.

This looks innocuous:

    :stash-path($*HOME ~ '/.metrics/' ~ $account ~ '/' ~ $server ~ '/stats')

This wouldn't generate much interest:

    :stash-path('/var/dynaplex/.perf/' ~ $account ~ '/dynaplex.' ~ $server)
    :user-exclusive-at('/var/dynaplex/.perf/' ~ $account)
    
These misleading paths result in added camouflage, and every little bit helps.

Limitations
===========

Only developed on Linux.

Author
===
Mark Devine <mark@markdevine.com>
