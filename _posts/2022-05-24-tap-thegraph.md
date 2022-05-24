---
layout: post
title: "Build blockchain data pipelines with The Graph & Singer"
---

<!-- TODO: remove TLDR -->

## TL;DR

[The Graph](https://thegraph.com/en/) is an indexing protocol that makes structured blockchain data available via GraphQL APIs called subgraphs.

[Singer](https://www.singer.io/) is an open-source standard for writing data extraction scripts (taps) and data loading scripts (targets).

I wrote a Singer tap for The Graph ([`tap-thegraph`](https://github.com/superkeyio/tap-thegraph)) to allow extracting blockchain data from subgraphs and loading it into any destination with a target (CSV, Postgres, Snowflake, & more).

## Why?




