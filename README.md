DDM Tech Blog
=============

The DDM Tech Team Blog is intended to be the place for DDMers to blog about 
solutions and things they find helpful. 

It is a statically generated blog hosted by GitHub so we don't have to worry
about security patches and keeping it up to date like with WordPress. 

## Hexo

We've migrated the blog from Jekyll, which required all sorts of correct gems 
installed, to Hexo which is easier to install and use. The [hexo documentation](http://hexo.io/docs/)
is pretty helpful if you want to make some more significant changes.

## Prereqs

You'll need Node and NPM installed. If you're on a mac and using homebrew you 
can do this easily:

``` bash
brew install node
```

## Setup

Go ahead a clone the blog at [https://github.com/deseretdigital/deseretdigital.github.io](https://github.com/deseretdigital/deseretdigital.github.io). Once you've finished, cd into the directory and then run the following commands:

``` bash
cd deseretdigital.github.io

# Globally install the hexo tool
npm install -g hexo

# Install the local npm dependencies
npm install
```

## Previewing the Site

You can preview any changes you've made by running the ``hexo server`` command:

```bash
JustinCarmony-Macbook in ~/git/ddm/deseretdigital.github.io
± |source ✓| → hexo server
[info] Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```

## Making changes

You'll want to make sure you are on the source branch, and not master. Master 
is where the generated files will be. 

The blog uses GitHub flavored markdown ([basics](https://help.github.com/articles/markdown-basics), [advanced](https://help.github.com/articles/github-flavored-markdown)).

All of the files for the blog's content can be found under the ``[source](source)`` directory. 

## Adding a new post

You can add a new post by running the command:

```bash
hexo new post the-title-you-want-sluggified
```

It will create a file in the right format under ``[source/_posts](source/_posts)``.
Just update the content and it'll show up automatically under the preview.

## Pushing Changes Live

Make sure you commit your changes to the ``source`` branch and then push those up.
Then you just need to generate and deploy the new generated source:

```bash
hexo generate
hexo deploy
```

Then within 10 minutes the github page should update and show your changes. Awesome!

If you have any questions, feel free to ask Justin Carmony.

