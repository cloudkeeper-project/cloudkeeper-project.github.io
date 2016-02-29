# cloudkeeper.xyz

This repository contains the source of the [cloudkeeper.xyz](http://cloudkeeper.xyz) website. It’s built using [Jekyll](http://jekyllrb.com) and designed with [Bootstrap](http://getbootstrap.com).

In order to preview the site locally, follow the instructions on GitHib, titled [“Setting up your Pages site locally with Jekyll”](https://help.github.com/articles/setting-up-your-pages-site-locally-with-jekyll/). Basically, what it comes down to is installing [Bundler](http://bundler.io) once with

```sh
gem install bundler
```

and use

```sh
bundle exec jekyll serve
```

to start a local webserver that automatically refreshes as you update the local files. It uses the same versions as GitHub Pages in order to compile the site (this is configured in file `Gemfile`). Initially, and whenever versions used by GitHub Pages are updated, run:

```sh
bundle install
```
