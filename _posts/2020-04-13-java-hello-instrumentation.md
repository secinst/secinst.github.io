---
date: 2020-04-06 16:32:40
layout: post
title: Hello, Instrumentation (Java)
subtitle: Getting started with instrumentation in Java.
description: A step-by-step tutorial that will help you get started with instrumentation in Java.
image: https://www.wns.com/Portals/0/Images/Articles/DesktopImages/600/159/Digital_ROI_HD-1.jpg
optimized_image: https://www.wns.com/Portals/0/Images/Articles/DesktopImages/600/159/Digital_ROI_HD-1.jpg
category: java
tags:
  - bytebuddy
  - IAST
author: secinst
paginate: true
---

In this article, we will set up a simple "Hello, World" type example for using instrumentation. The canoncial web app for this would look something like this...

```
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<body>

<form>
  <div>Enter name:</div>
  <input name="name">
  <button name="submit" type="submit">Submit</button>
</form>

<h1>Hello, ${param.name}!</h1>

</body>
</html>
```

Try visiting http://localhost:8080/ticketbook/hello.jsp?name=Fred and it works just great. But unfortunately, you just introduced a cross-site scripting vulnerability. This happens anytime you write untrusted data (the "name" parameter) to the HTTP response without escaping it. You could easily fix this in the code. But what if just didn't ever want to think about this again?

Instrumentation to the rescue. Let's instrument a method that will eliminate any dangerous characters from all HTTP parameters. This is definitely overkill, but it's a good demonstration of the power and elegance of the instrumentation approach. Basically, if you can articulate a rule for where certain code should be used, instrumentation is a clean and easy way to make sure it's always there.

[PICTURE: normal development mess VS. software instrumentation separation of concerns.]




