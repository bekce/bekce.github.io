---
layout:     post
title:      SuxChat – A Nifty Java Client-Server Chat Tool
date:       2012-05-08
categories: 
---

Back in the days when we were giving consultancy service to a company which limited the internet usage of its employees, I’ve developed a simple solution for our inside-company basic communication problem with a nifty tool written in Java. This tool have some capabilities compared to other similar solutions. It has the following features.

![sux chat ss](http://sebworks.com/wp-content/uploads/2012/05/sux-chat-ss.png)

- Keep alive mode, which prevents connection loss due to inactivity for a long period.  
- Operator mode to kick out users, which is enabled with a special password.  
- Entrance password mode can be enabled to lock out strangers.  
- Normally all messages are broadcasted to all clients and private messaging feature allows secret communication between any two client.  
- Listing of currently logged on users.  
- A nice Swing GUI for the client, which flashes its icon in dock when it receives a new message.  
- Uses UTF-8 as its string encoding.  
- Messages are time stamped by the server.  
- Communication is done via plain sockets.  
- Configuration is done via properties files.  
- Unlike many other tools, there’s no bug and runs perfectly.

The source code is available with Apache License 2.0\. Its good reference for a Java learner.

Instructions for server:
After you downloaded the executable, open command prompt and type in:  
`java -jar <path to server jar file>``

Instructions for client:
Open with Java Jar Launcher (a.k.a double-click).

Downloads:

- [ChatClient3.7](/static/ChatClient3.7.jar) (binary executable)
- [ChatServer3.7](/static//ChatServer3.7.jar) (binary executable)
- [Sources](/static//suxchat-3.7.zip)

Good luck.
