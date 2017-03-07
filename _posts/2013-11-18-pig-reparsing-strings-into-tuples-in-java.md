---
layout:     post
title:      "Pig: Reparsing Strings into Tuples in Java"
date:       2013-11-18 23:16:00
categories: java, pig
---

Recently, I needed to read text which is stored with PigStorage. The text also had internal bag and tuple structures so I didn’t want to reinvent the wheel. However there is no direct documentation about that, so you have to dig into the Pig source to find how does Pig itself read it. Luckily enough, I’ve found it.

```java
import org.apache.pig.ResourceSchema.ResourceFieldSchema;
import org.apache.pig.builtin.Utf8StorageConverter;
import org.apache.pig.data.DataBag;
import org.apache.pig.data.Tuple;
import org.apache.pig.newplan.logical.relational.LogicalSchema;
import org.apache.pig.impl.util.Utils;
```

Let’s say your string to be parsed is this:

```java
String tupleString = "(quick,123,{(brown,1.0),(fox,2.5)})";
```

First, parse your schema string. Note that you have an enclosing tuple.

```java
LogicalSchema schema = Utils.parseSchema("a0:(a1:chararray, a2:long, a3:{(a4:chararray, a5:double)})");
```

Then parse your tuple with your schema.

```java
Utf8StorageConverter converter = new Utf8StorageConverter();
ResourceFieldSchema fieldSchema = new ResourceFieldSchema(schema.getField("a0"));
Tuple tuple = converter.bytesToTuple(tupleString.getBytes("UTF-8"), fieldSchema);
```

Voila! Check your data.
```java
assertEquals((String) tuple.get(0), "quick");
assertEquals(((DataBag) tuple.get(2)).size(), 2L);
```

[StackOverflow Question Link](http://stackoverflow.com/questions/15676470/pig-reparse-strings-into-tuples-in-java/19765535#19765535)
