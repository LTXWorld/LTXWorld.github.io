+++
date = '2025-12-23T10:23:41+08:00'
draft = true
title = 'Debug2512'
+++

# Debug2512

This is a storage about issues that I met in December 2025.

## Pop-up windows Blocked

### Description

I use Safari as my default web browser.

When I *visited* the Duckcoding website and *attempted* to register using my Linux.do account, the redirect to the Linux.do authentication page failed.~~I can't jump to the Linux.do register page(the third-party register).~~ Instead,a "Pop-up Window Blocked" notification *appeared at* the top of my browser.

### Steps to Reproduce

1. Open a website that requires third-party authentication(OAuth).
2. Click "Register/Login via by Linux.do"
3. Observe that the browser blocks the new window from opening

### Expected Behavior

The third-party login window should open immediately without being blocked, allowing for a successful redirect to the authentication page.

### Solution

Initially,I *suspected* "uBlock" extension was the **culprit**. However, even after disabling it, ~~when I deny it~~,the issue **persisted**。I searched for the error on Google and located  the [apple support documentation regarding this behavior](https://support.apple.com/en-us/102524) .

Following the guide, I updated my settings to allow pop-up windows specifically for the Duckcoding domain. ~~to close the pop-up window blocked settings about the Duckcoding websites.~~

Finally,the issue was resolved, and the login flow worked as expected.

### 原因

> Pop-ups can be ads, notices, offers, or alerts that open in your current browser window, in a new window, or in another tab. Some pop-ups are third-party ads that use phishing tactics such as warnings or prizes to trick you into believing they’re from Apple or another trusted company, so that you’ll share personal or financial information. Or they might claim to offer free downloads, software updates, or plug-ins to try to trick you into installing unwanted software.

