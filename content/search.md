---
title: "Search" # in any language you want
layout: "search" # necessary for search
# url: "/archive"
# description: "Description for Search"
summary: "search"
placeholder: "placeholder text in search input box"
params:
  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    distance: 1000
    threshold: 0.4
    minMatchCharLength: 0
    # limit: 10 # refer: https://www.fusejs.io/api/methods.html#search
    keys: ["title", "permalink", "summary", "content"]
---
