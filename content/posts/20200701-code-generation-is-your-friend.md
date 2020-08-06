---
title: "The beauty of formally defined APIs"
author: "Facundo Medica"
type: "post"
date: 2020-07-01T22:23:02-03:00
draft: false
subtitle: "Part 1: Swagger/OpenAPI"
image: ""
tags: []
---

As software developers we need to perform several tasks _at the same time_, coding, writing documentation and writing tests. And for every feature our current project requires we'll perform, to a greater or lesser extent, all of these three things. So we can deduce that if we reduce the time spent in any of these activities we are then reducing the time that it takes our team to finish a feature.

Common cases that this post may be helpful in:

- Backend developers have to communicate the changes of each endpoint to frontend developers, maybe even including payload examples.
- Frontend/backend repeating code or having to write multiple helper functions to aid networking
- API documentation hasn't been updated in a long time, and it may stay that way a little bit longer (_no one has time for that, right?_)
- The backend team rolled their own HTTP framework out and now it's a PITA to maintain. We've all gone through this. We, think that we need our software to be _taylor-fitted_ to our project but in reality a very small number of projects really need custom stuff. If you find something that looks like a special feature that no one ever implemented, then put some extra thought into it, I know you can simplify it!
- The frontend team complains about the backend team not being consistent (this won't be solved that easily, but now you can be inconsistent in an orderly manner 🤣)

I'll be focusing on client-server/frontend-backend apps, more specifically the link between the parts and how we can make this interface as smooth as possible. The two tools described below were chosen by personal experience. **Do your own research to find the best fit for you, as there are hundreds of options.**

## Swagger/OpenAPI

It's an open-source framework that helps us to design, build, document and consume HTTP APIs. It consists in a YAML document which holds all the definitions for our endpoints including their request and response objects.

We can then use our yaml definition to create client and server libraries, static documentation, interfaces for easy manual testing (like [Swagger Editor](https://editor.swagger.io)) and also tools for automated testing. But to be clear, you don't need to use all of the available tools to get benefit from this framework, you can adopt it gradually and start implementing more things as the team becomes more comfortable with it.

A good startpoint in my opinion is having a generated client library and a generated HTTP framework for the backend:

- Best ROI. The amount of effort for this setup is extremely low compared to the amount of time that it'll save the team, even in the short term.
- Get a tremendous jumpstart on documentation and accessibility of your APIs. In some cases it can replace tools like Postman completely.
- In many cases a code generator for the client will turn the API endpoints into native functions, so a `GET /users` becomes `ListUsers()`.
- On the backend you can focus on the business logic only, so forget about reinventing the wheel with a custom HTTP framework.
- Code generators usually provide you with already well tested code, so yay! less time writing boring tests!


But wait! You need to consider the following things before running into the office (or the Zoom/Google Meet/Jitsi/Microsoft Teams meeting) to tell everyone about it:

- **OpenAPI != Swagger** OpenAPI it's like Swagger 3.0, but do your research, take into account the features that each one has (or lack thereof), the availability of code generators for your desired languages and of course their quality. Spending extra time here will spare you from headaches in the future!
- **Just because it's a valid YAML doesn't mean it's the right way.** The job of keeping things consistent or following a structure (like REST) is on you, so don't think this is the magical solution you were waiting for!
- **It's not a drop-in replacement to whatever framework you are currently using.** Switching to a different framework is an always painful and cumbersome task, and this is no exception. But don't be scared! It's absolutely doable if you plan ahead and think critically about your next steps. If you have some kind of versioning along rolling updates then it should be a piece of cake 🍰.

Stay tuned for a gRPC focused post!