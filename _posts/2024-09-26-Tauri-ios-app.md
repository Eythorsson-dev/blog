---
date: 2024-09-26
title: Tauri IOS app
---

Last night I, was working on a side project i started a week ago. I was attempting to setup the tauri ios app, when i stumbled into a few issues. Here is a summary of what i needed to do inorder to resolve the issues.

First, i was not able to setup the ios project. After searching a few hours, i stumbled upon this example. These commands worked, where as the commands on the official documentaion did not work: 
https://github.com/edsky/tauri_yew_ios

Next, after setting up the application, i got another issue. This issue was resovled by opening the xcode project, generated using the `cargo tauri ios init`, and configuring the signing of the project. In my case i needed to login, + set the correct team on the application.
