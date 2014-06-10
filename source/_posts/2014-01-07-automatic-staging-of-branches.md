title: "Automating staging of feature branches"
date: 2014-01-07 11:52
comments: true
categories: [git, php]
author: Mark Sticht
---

Have you found team trying to swtich your staging server from one branch to  another for a demo of feature X or Y? Why limit your project to just one staging site? Just make all your branches available on your stage server as soon as they are pushed to github.

We have a simple to use script that can quickly pull all of your branches down to your server. Throw in a little server and DNS configuration and each new branch will be accessable as its own staging site.

Lets say you work on example.com and have the following branches - social, login  and master. Using [4gitphull](https://github.com/deseretdigital/4gitphull) you'll be able to access social.example.com, login.example.com and master.example.com. 

4gitphull can also generate a static html file that contains a list of commits that are on a branch, but not in master. Providing that your site has a page that  will show the hash of the current deployed code, a second file can be generated  that lists the commits that are on master, but are not yet live. Who wouldn't want to know what would go live if you were to deploy right now?

See more detail on github [4gitphull](https://github.com/deseretdigital/4gitphull)
