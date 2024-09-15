---
title: Journal - The worlds best note taking application
date: 2024-09-15
---

The last year, i have worked exclusivly on a note taking application. This application, is no longer a meer side project. It is something i have to accomplish, something I need in my life. And to be honest, something I truly believe will outbeat the competition - normal, boring, paper.

The journey started as a asp.net core web app (as it was at the time the language i was the most familiour with). After months of hard work, i got to a point where i most of the major features i wanted. It had supported indent, lists and inline tools such as bold, italic, underline and of corse a method of linking to other notes / resources. It had a canvaas mind map and a customisable table. I pretty happy with what I had created. Sure, it had some bugs, but what application does not have those..? 

This was around the time I started looking into TDD. and boi, did that change things. I now had a reliable way to get rid of bugs. I managed to add tests to my application, but, it was very difficult, and the boiler plate was creasy. I did not know much about dependency incjection, only that it was hated at the job i worked at. Luckely for me, i had just quit that job, and was looking for a new one. After a few weeks of searhing and applying, i finally got a few interviews (i was not expecting it to be this hard). After interviewing on the first, i realised that i would be working with a bunch of retierees in a legacy code base. In this company i would be the first hire, and groomed to become the project lead. 

I had no garantee that i would get this job, and i therefore continued my search. That is when i stumbeld oppon another position. I created my application and got ready to apply, only to realise that the application process had just closed. I picked up my phone, diled the phone number given in the post, and jokeingly asked the reqruiter if they would like to hire me. He told me to send in my application, on another position they had open, and that he would take a look at it. a few days later, he called me back, and asked if i was interested in an interview. Seeing the chance, i told him yes, and that i would fly down to meet them (i lived in a small city, this job was in the capital). A few days later, i was on the morning plane to the capital. When i landed, i thought to myself. What on earth should i do now? i had 6hrs untill my interview, and no plans. I ended up walking from the city centre to their HQ, and waited the rest of the time on a bench close the the building.

I was so excited after the interview. I could feel i would get this job, or at very least, that i wanted it. A week or so later, my phone rang, it was the boss of the first company i interviewd at. They offered me a job. And boy, the pay was creasy. it blew me away. I told him that i needed to think about ut. Because really, I needed. But, i could not shake the feeling, that this was not what i realy wanted. it felt as if i was setteling for a job. a lifestyle. i saw my life unfold, and i did not like what i saw. 

long story short, i ended up taking the second job. Accepting a step pay cut, figuring that i would learn, far more in that job than in the first. And so for, 1 year leater. I now feel like a worse developer, than i did 1 year ago. I now work with a bunch of code geeks (in the good sense). My reality shifted taking this job. I think in a new way, and struggle to understand everything i have learned - they have become that second nature. I can imagine working in a different way.

What i have learend:
- Scrum
  - The value of having daily cordination meethings
  - To descuss what went well, and how we can improve the next two weeks (retrospective)
  - The importance of having everyone onboard, of what needs to be done, and how we should do it
- Code Infrastructure
  - CQRS - Being able to create bite sized, reusable components that can have multiple entry points (REST, GraphQL, Internal Jobs, etc.)
  - Domain Driven Developemnt - Although our code base is not DDD, i am lucky to work with colleagues with years of experianse (allowing me to pick their brains)
  - Event Driven Development
    - When the project started, the goal was to have it event driven. However, as the development processed, we ended up doing it synchronosly (due to debug difficulties). However, this weekend, i have come up with a POC (Proof of concept) that would allow us to refactor our codebase to allow it to be fully Event Driven, without any of the difficulties we previously encountered.
  - Test Driven Development (TDD)
  - Dependency Injection
  - Clean Architecture
