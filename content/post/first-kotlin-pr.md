+++
author = "Stephan Schröder"
title = "The first Kotlin pull request to a Java project shouldn't contain any Kotlin"
date = 2023-08-19T23:26:35+02:00
description = "devs/managers unfamiliar with Kotlin think adding it to a project is risky and therefore delay it. So commit first only configuration changes to use Kotlin in the future."
tags = [
    "kotlin",
]
+++

but only the configuration changes to add the Kotlin compiler next to the Java compiler.
Make the project ready to add Kotlin code in the future, but don't add Kotlin code quite yet.

**I'm meeting Java backend devs all the time who have no idea how smoothly Kotlin code and Java code work together!**
Even some devs who are genuinely interested in Kotlin, but haven't had any real world experience with it yet.
Kotlin courses seem to focus on showing off Kotlin, so students tend to be exposed to Kotlin only codebases exclusively.
Unfortunately this makes the necessary first step of adding Kotlin to a Java project feel bigger than it is and therefore risky.
And risky steps are delayed "until an unspecified time in the future when the team will have enough resources to deal
with all the issues that are assumed to pop up". This can be a long time.

### The first PR

So the first pull request should only contain the configuration changes necessary to add the Kotlin compiler next to
the Java compiler. Since no code changed and the old tests are still green for the old Java code, the PR shouldn't feel
that risky and therefore easy/ier to accept.

### The second PR

Wait until the first has been accepted and merged to *main*. Maybe even wait for the next sprint.
Now convert a single, not to big of a class. I like to start with that class that contains the `main`-method since it's
normally a small file with mostly framework configuration. Additionally, no other class tends to have a reference to
the `main`-class and therefore the changes are truly contained to a single file. Create a pull request.

It's been accepted and merged? Congratulations, you now have a `src/main/kotlin` directory path in your project with a
working Kotlin class (or maybe no class and just a `main`-method) inside it. The adoption barrier has been overcome.
From this point on the percentage of Kotlin code within the project tends to mimic entropy and increase over time on its own.

## Appendix

