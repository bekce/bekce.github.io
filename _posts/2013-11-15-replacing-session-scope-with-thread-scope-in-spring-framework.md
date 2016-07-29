---
layout:     post
title:      Replacing Session Scope with Thread Scope in Spring Framework
date:       2013-11-15
categories: spring
---

If you develop a web application with Spring Framework, your session scoped beans hold various session bound attributes like logged user. The advantage of storing these values in Spring Application Context is that your service layers (SOA) can benefit from it without getting the current user object from every request against the servlet.

### The Problem

However easy to use it is, there is a problem with session scoped beans when your application also needs other integration points other than web servlets. Consider you are implementing a FTP server that will serve requests on port 21\. If you try to get session scoped `SessionData` bean from the context in ftp server code, spring will give you an error like “_Scope ‘session’ is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton_“. This will also happen while unit testing. So what should we do here? Note that we want that our singleton SOA instances remains clean and intact.

We should change `SessionData` bean to thread scope. Thread scope is a custom scope that we must define in our application context whose implementation is included as a support class in spring framework libraries.

### How does it work?

![session data chart](images/image-e1384537975318.png)

In this example, non-web part is a FTP server. For the sake of simplicity I assume it uses a single thread for each client.

Normally, session scope puts the bean in the http session. In this case, while the servlet instance for a client remains still, application container may (and will) assign different threads for every request to the same servlet instance. So we should set our `SessionData` instance in the spring context with the one that resides in `HttpSession` (if found) at the beginning for every request. After the request is processed, we should then set the `sessionData` in `HttpSession` with the one in application context. That way, we know for sure that in every request, we have the valid `sessionData` object.

### How to do it?

Follow the steps in [http://www.springbyexample.org/examples/custom-thread-scope-module.html](http://www.springbyexample.org/examples/custom-thread-scope-module.html). You must download sources or add it as a maven dependency to your project, then add your new scope to your `applicationContext.xml` file because it is not included in Spring as default.

Suppose you have a very simple `SessionData`, which must hold the logged in user if any. You were using this bean with “session” scope before, but now you will be serving a non-web service and you want to take advantage of spring’s injection also.

```java
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
@Scope("thread")
public class SessionData {

	private User currentUser;
	public User getCurrentUser() {
		return currentUser;
	}
	public void setCurrentUser(User currentUser) {
		this.currentUser = currentUser;
	}
	public void set (SessionData other){
		if (other != null) {
			this.currentUser = other.currentUser;
		}
	}
	@Override
	public SessionData clone() {
		SessionData clone = new SessionData();
		clone.set(this);
		return clone;
	}
}
```

We’re going to store `SessionData` in `HttpSession`. In your Servlet class (you should extend the Servlet you use if you haven’t done yet), add the following:

```java
@Override
protected void service(HttpServletRequest request, HttpServletResponse response){
	SessionData sessionData = (SessionData) request.getSession().getAttribute("sessionData");
	if (sessionData != null) {
		SessionData sessionDataFromContext = applicationContext.getBean(SessionData.class);
		sessionDataFromContext.set(sessionData);
	}
//
// do your work here
//
	SessionData sessionData = applicationContext.getBean(SessionData.class);
	request.getSession().setAttribute("sessionData", sessionData.clone());
	sessionData.setCurrentUser(null);
}
```

What we are doing here is that we pull the `sessionData` attribute that we wrote to `HttpSession` and set it to current thread’s `SessionData` bean. At the end, we set the `SessionData` bean in spring to the `HttpSession` as it may have changed.

Above Servlet code is for any servlet application. However, this modification may not be efficient for some frameworks. For example if you’re using Vaadin, you don’t need to extend Servlet, just override `onRequestStart` and `onRequestEnd` in your `Application` class and you’re done:

```java
@Override
public void onRequestStart(HttpServletRequest request, HttpServletResponse response) {
	SessionData sessionData = (SessionData) request.getSession().getAttribute("sessionData");
	if (sessionData != null) {
		SessionData sessionDataFromContext = applicationContext.getBean(SessionData.clas);
		sessionDataFromContext.set(sessionData);
	}
}

@Override
public void onRequestEnd(HttpServletRequest request, HttpServletResponse response) {
	SessionData sessionData = applicationContext.getBean(SessionData.class);
	request.getSession().setAttribute("sessionData", sessionData.clone());
	sessionData.setCurrentUser(null);
}
```

Good luck!
