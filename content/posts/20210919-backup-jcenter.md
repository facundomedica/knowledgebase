---
title: "Backup JCenter/Bintray Android dependency"
author: "Facundo Medica"
type: "post"
date: 2020-09-19T22:23:02-03:00
draft: false
subtitle: "JFrog deprecated Bintray and JCenter, let's create a plan B"
image: ""
tags: ["android","hacks"]
---

A week ago I noticed that our Android builds were failing in the pipeline, the logs stated that it failed to download a couple of dependencies from JCenter with 502s, so I thought _"no worries, I'll just re-run the pipeline and it'll get fixed!"_ **Nope!** The JCenter server was down for a while, so I decided to do some googleing and found out that JFrog had deprecated Bintray and JCenter [earlier this year](https://blog.gradle.org/jcenter-shutdown).


After a while everything was back to normal, but it was clear we had to stop using JCenter and use mavenCentral. This should have been an easy change, but it wasn't. **The main dependency in our app is a closed source library that is exclusively published on JCenter by a trusted (_but with bad dev support_) third-party**. So we couldn't just pull it from Github and use it as a submodule or something like that, we had to use a different approach.

Of course we count on the publisher to migrate to mavenCentral some time in the future, but in the meantime we'll have this plan B ready to deploy whenever things go south (which for a service that's getting deprecated might be more often than usual).

## Presenting the plan B: _scrape the entire maven directory off JCenter for this dependency_.

### Step 1
#### Use wget to download everything from JCenter

```bash
 mkdir localmaven && cd localmaven
 wget --recursive -np -nH -e robots=off -l5 http://jcenter.bintray.com/com/thevendorname/
```

### Step 2
#### Add your local maven repo to your build.gradle

Under `allprojects` -> `repositories` add the following (in the last position):

```
        maven { url rootProject.rootDir.getAbsolutePath() + "/localmaven" }
```

### Step 3
#### Remove jcenter (optional)

Given that the libraries on JCenter won't receive any more updates, you can safely remove it from your build.gradle once you've made a copy of all the required dependencies.


