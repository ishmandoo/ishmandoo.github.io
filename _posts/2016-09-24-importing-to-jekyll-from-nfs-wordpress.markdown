---
layout: post
title: Migrating from Wordpress on NearlyFreeSpeech to Jeckyl
date: '2016-09-23 18:55:26 -0400'
categories:
- Programming
tags: []
comments: []
---
<p>
  I messed up my Wordpress installation and I don't know how to fix it. It was hosted on <a href="www.nearylfreespeech.net">nearlyfreespeech.net </a>, but their version of the wordpress command line client seems to be either broken or too old to work with an up-to-date Wordpress install. Eiter way, I don't have access to the wp-cli. So, I decided to try something else
</p>

<p>
  I read good things about Jekyll, so I decided to try it. There's a lot there, but one sticking point was migrating the database into Jekyll. I don't think you can access NFS's mysql server from outside, so I downloaded the data using phpMyAdmin, created a database, imported the data with <code>source data.sql</code>, and ran the Jekyll import function. 
</p>

{% highlight bash %}
ruby -rubygems -e 'require "jekyll-import";
    JekyllImport::Importers::WordPress.run({
      "dbname"   => "benwiener",
      "user"     => "username",
      "password" => "password",
      "host"     => "localhost",
      "table_prefix"   => "wp_"
    })'

{% endhighlight %}
