--- 
layout: post
title: "Golang Snippets: Caching API Calls in the DB"
tags: golang snippets
date: 2024-04-20 0:00:00
---

Say you're going to make a network request, and you want to store the result in the database.  Maybe it's to prevent hammering on an external API.  Maybe you want to seed a different table with the results.  Maybe you want to construct a new model, with the results from the API as your jumping-off point.  There's tons of reasons you'd want to do this.
