---
title: "Introducing Sprinkles, a Utility to Safely Share Secrets in Dev Environments"
date: 2020-09-08T04:31:15+03:00
draft: false
---

We've all been there. Setting up your dev environment was meant to be a single
command away. And yet, here you are, hours later, just on the cusp of victory,
before it turns out that you're missing yet another API key. You know you
probably shouldn't, but you DM your teammate, let's call her Fatima, asking
her to please send you the credentials. You'll clean up after yourself,
deleting the message as soon as you add the credentials to your environment
configuration. But it still feels dirty. You have the nagging feeling that
they're still lurking about somewhere in the ether, vulnerable to the next
data breach. Sure, you know you should avoid sharing credentials to begin
with. You know that, if you must, plain text is the worst possible way to
go about it. You've taken the 12 factor methodology to heart, so much so that
even your local dev set ups are configurable using environment variables. But
it's been a long day. You're tired, but you'll be careful.
InfoSec will understand.

I've had to deal with different iterations of this problem more times than I'd
like in the course of my career. Well, really, it's two, maybe three,
intertwined problems. You have a somewhat repeatable dev environment. This could
be your package.json/package-lock.json combo, your docker-compose.yml, your
pom.xml... you name it. But you want to have enough flexibility to allow
everyone to customise their environments to their heart's content. So you have
a .env file at the root, or an application-dev.properties file, ignored in
your VCS config. You have a few different iterations of this file doing the rounds
inside the team, full of implicit knowledge. You don't want it in VCS, since it
might contain some sensitive info, but also because it's intended to allow for
differences between set ups.

Sprinkles is an attempt to solve both problems. It uses templates that you
can safely track in VCS, eliminating the dependence on implicit knowledge, without
the risk of accidentally exposing sensitive information.
The secrets are stored in AWS Secrets Manager. This allows for them to be encrypted,
and managed within your organisation's IAM policy constraints.
The templates are defined in the commonly used Jinja templating language.

Let me run you through a brief demo.

To install sprinkles, run `pip install sprinkles-config`. Create a `.sprinklesrc` file
in your project root, in TOML format.

*.sprinklesrc*

```toml
[secret]
arn = "arn:aws:secretsmanager:<region>:<account-id-number>:secret:<secret-name>"

[files]
  [files.docker-env]
    template = ".env.sprinkles"
    target = ".env"
```

*.gitignore*
```gitignore
.env
```

*.env.sprinkles*
```shell script
THIRD_PARTY_API_ENDPOINT=https://example.com/api/v1
THIRD_PARTY_API_KEY={{THIRD_PARTY_API_KEY}}
```

`THIRD_PARTY_API_KEY` is defined as a key in an AWS Secrets Manager secret, identified by the ARN set in
`.sprinklesrc`.

To initialise a functioning `.env` file, you simply need to run `sprinkles` in the project root containing
`.sprinklesrc`.

The templating system allows for arbitrary text-based config formats, so you can just as easily adapt
it to `application.properties`, for instance. It enables the team to confidently share implicit
configuration knowledge. A full dev set up is one more command, rather than 10 Slack messages,
a Zoom call, and two existential crises away.

For more details, please check out the PyPI page, here: https://pypi.org/project/sprinkles-config/.

