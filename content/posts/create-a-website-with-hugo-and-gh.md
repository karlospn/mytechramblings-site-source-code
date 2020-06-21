---
title: "Create and host a blog with Hugo and GitHub Pages in 20 minutes"
date: 2020-06-21T18:28:38+02:00
draft: false
---


Do you want to start a blog and don't want to lose a lot of time setting it up?  
That was exactly my thought! I wanted to build a blog to write about my ramblings and I didn't want to spend long hours settings things up.  
After doing a little bit of research about what options are available nowadays I found that Hugo could be a good fit.

# What's Hugo?

Hugo is another static HTML and CSS website generator, it is written in Golang and it relies on markdown files.  

What I liked about it is how few steps you need to perform to have a blog ready to go. 

You just need to:  

+ Create a new static site
+ Choose a theme from his website  (https://themes.gohugo.io/)
+ Do some tinkering on how the theme is going to work in your website 
+ Choose hosting provider
+ Publish it

And that's it! You are ready to go!

There are a lot of options for hosting an static website but I chose GitHub Pages because it is free, easy to work with and I already have an existing account. 


# Steps

After reading a little bit about how to set everything up I begin building my site

## Install Hugo

I work mainly with Windows so I use *Chocolatey*.

> Chocolatey it's just a package manager for Windows if you are interested in learning more about check it out its website:  https://chocolatey.org/  

You can install Hugo with the following one-liner:

```bash
choco install hugo -confirm
```

If you use another OS just check out the official docs about how to install it:   
https://gohugo.io/getting-started/installing/

After you install Hugo you can use it via command line, just type <i>hugo -help</i> to list all the options available

## Create a new site

We are going to create a new Hugo site in a folder named dotnetramblings.

```bash
hugo new site dotnetramblings
```

## Add a theme

Next step is to style our site. We can choose an existing theme from the website: https://themes.gohugo.io/   
Pick the theme that you prefer and clone it inside the /themes folder

```bash
cd dotnetramblings
git clone https://github.com/panr/hugo-theme-hello-friend.git themes/hello-friend
```

You can see how your website currently looks by typing:

```bash
hugo server -D
```

## Create a GitHub repository and modify the config.toml file

Go to github and create a new repository. I create the repository "dotnetramblings".   
Once the repository is created you **HAVE** to modify the config.toml file found on the root directory of your site.   

You need to update the baseUrl property found in the config.toml file to point to your github pages:   

```toml
baseurl = "https://myuser.github.io/dotnetramblings/"
```


## Build your site

After tweaking our theme we are going to build our site. We have to execute 

```bash
hugo
```

The output will be in **./public/** directory  
The **./public** directory is what we are going to publish into github   


## Push your code into GitHub

After that you just have to set the repository in the public folder

```bash
cd dotnetramblings/public
git init
git remote add origin https://github.com/myuser/dotnetramblings.git
git add .
git commit -m "Initial commit"
git push
```

After pushing the code into gitHub you go into the repository settings and change the setting Settings > Options > GitHub pages > Source  to "master branch"

## Test your webpage

After a couple of minutes if you browse into https://myuser.github.io/dotnetramblings you will see your website up and running.

And that's it! You have a complete running website in less than 20 minutes. 


## Finally automate everything

Be aware that right now your source code is only on your local machine.  
So I'm going to create another github repository and put the source code in there and then i'm going to build a github action to publish the site automatically.

I create the repository dotnetramblings_source, after that I create a .gitignore file in my dotnetramblings site and push everything there

```bash
cd dotnetramblings
echo "public/" >> .gitignore
git init
git remote add origin https://github.com/myuser/dotnetramblings_source.git

```