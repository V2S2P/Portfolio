---
title: "Dify auto scraper"
date: 2026-04-17
draft: false
description: "Third lesson, Dify auto scrape Hugo site"
---

## What did I do?

We used the service Dify, an external service that allows us to play with LLMs, and build our own RAG-chatbot. After setting up Dify, we built an app that allows Dify to auto scrape our Hugo site, learn all the information on our site, and add it to our RAG-chatbot. 

This process will happen whenever we update our Hugo site so that it will stay updated along with the site.

## Components

- [Dify service](https://cloud.dify.ai/apps) (Service to easily create RAG-chatbot)
- Our own backend app
- An API key from Dify used in our backend