---
layout: post
title: 'Migrate Code Snippets in Xcode'
categories: [iOS dev]
tag: [swift, xcode]
---

We developers all want to boost our productivity and snippet gives us a hand. You can simply type some keywords to retrieve a bunch of boilerplate code. If you have no idea how to create and edit snippets, I recommend you read this [article](https://sarunw.com/posts/how-to-create-code-snippets-in-xcode/). What I want to talk about is how to import and export your custom snippets once you have made them.

It's a bummer that we lose our snippets whenever we change a device or update a new Xcode version. In fact, it's pretty easy to migrate your snippets. Open the terminal and run this command:
 
 ```cd ~/Library/Developer/Xcode/UserData/CodeSnippets``` 
 
 Under the directory, you will see a list of files with `.codesnippet` filename extension. These are custom snippets you have created. Feel free to copy and paste them to another device and make sure to put them under the same directory. You can also use `git` to manage them. After you import the snippets, you need to restart Xcode so those files can be properly detected.  