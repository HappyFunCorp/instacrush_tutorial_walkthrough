# Deploying to Heroku

OK, so we're in a good place now to try and deploy this thing to heroku.  What we need to do is:

- Install the Heroku Toolbelt
- Create the heroku app
- Setup the heroku database
- Setup and configure the instragram APP
- Setup and configure Google Analytics
- Setup DNS

Let's get going on that now!

## Heroku Toolbelt

https://toolbelt.heroku.com

## Create the Heroku App

I'm going to call this app "happyfancyname".  You'll need to pick your own.

Lets type in the command now:
```
$ heroku create happyfancyname
Creating happyfancyname... done, stack is cedar-14
https://happyfancyname.herokuapp.com/ | https://git.heroku.com/happyfancyname.git
Git remote heroku added
$ git remote -v
```
