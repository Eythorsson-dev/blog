---
date: 2024-09-26
title: Tauri IOS app
---

Last night I, was working on a side project I started a week ago. I was attempting to setup the tauri iOS app, when I stumbled into a few issues. Here is a summary of what I needed to do in order to resolve the issues.

First, I was not able to setup the iOS project. After searching a few hours, I stumbled upon [this example](https://github.com/edsky/tauri_yew_ios). These commands worked, whereas the commands on the official documentation did not work

Next, after setting up the application, I got another issue. This issue was resolved by opening the Xcode project, generated using the `cargo tauri ios init`, and configuring the signing of the project. In my case, I needed to log in + set the correct team on the application.
