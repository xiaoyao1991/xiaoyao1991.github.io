---
layout: post
title: "Inserting data to MySQL with many-to-many relationship"
date: 2013-12-18 00:55:29 -0500
comments: true
categories: CS, MySQL
---
One of my recent work involves parsing a large XML file(DBLP), and for each data record parsed, it needs to be inserted into the db. The data would be populating into tables, and they are correlated with many-to-many relationship.   

So, obviously there will be 3 tables:  

- publications
- authors
- author_publication(with Foreign Key constraints on authors table and publications table)

The first try:  

1. Insert publication to 'publications', take the last insert id
2. Query an id from 'authors'
3. If step 2 returns an id(which means the author exist in the table), take that id; otherwise, insert the author to 'authors', take the last insert id
4. Take both id's and insert into 'author_publication' table.

This trial runs very slow... Because for every author, we will be executing at most 2 queries to the db, and the queries can be expensive as the db size grows larger. So, instead of executing 2 queries, I utilized the fact that author must be UNIQUE inside of 'authors' table. Here's the second trial:  


1. Insert publication to 'publications', take the last insert id
2. Insert author to 'authors' table and `ON DUPLICATE KEY UPDATE id=LAST_INSERT_ID(id);`, take the last insert id
3. Take both id's and insert into 'author_publication' table.

In this case, the second step would ensure that at most 1 query to the db. The details on `ON DUPLICATE KEY UPDATE` can be read from [The MySQL Manual Page](http://dev.mysql.com/doc/refman/5.0/en/insert-on-duplicate.html)

I'm not sure if the second trial provides the best performance here, but it indeed improve the performance significantly as compared to the first trial. 