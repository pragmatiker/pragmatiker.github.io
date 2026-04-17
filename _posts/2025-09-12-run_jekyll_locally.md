---
layout: post
title: "Running Jekyll locally"
date: 2025-09-12
author: "TimDude"
categories: ["Tech",  "Jekyll"]
tags: ["proxmox", "linux", "routing"]
---

To delevop faster I want to run jekyll locally and push to GitHub Pages

# Install ruby toolchain
```
sudo apt install -y ruby-full build-essential zlib1g-dev
```

# (Optional but nice) install gems to your home dir:
```
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

# Clone you GitHub Pags repo an enter it
```
git clone 
cd pragmatiker.github.io
```

# Check gemfile
I use Chirpy theme so it should look like this
```
# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.0", ">= 7.0.1"

group :test do
  gem "html-proofer", "~> 5.0"
end
```

# Install deps with bundler
Inside the repo dir do:
```
gem install bundler
bundle install
```

# Run the site
```
bundle exec jekyll serve --livereload
```
