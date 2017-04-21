---
layout: post
title: "Handling mails in non-production environments"
---

What options do we have to prevent such issues?

- Don't use production data outside production at all. If you need to, obfuscate email
  addresses (and other personal data). From the security point of view it is
  useful (required?) anyway. You can create email addresses, that all points to
  a single mailbox, where every developer has access too.

- Use a [fake SMTP server](http://nilhcem.com/FakeSMTP/). Just let the application send
  "real" (see first bullet point) mails, that will all end in a sink. This is especially
  useful during development, because you can easily check, what was sent.
  
- Use the [`delivery_address`](http://symfony.com/doc/current/email/dev_environment.html#sending-to-a-specified-address-es)
  and configure it in the `parameters.yml`. In my opinion the last option, because it
  is not as failsafe as the both options above. Still it works.
