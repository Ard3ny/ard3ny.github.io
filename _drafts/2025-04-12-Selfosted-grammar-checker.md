---
title: Selfhosted grammar checker
date: 2025-04-20 20:00:00 +0100
categories: [Homelab]
tags: [homelab-2.0, cloudflare, pangolin, immich]
math: false
mermaid: false
series: homelab-2.0
---

https://github.com/languagetool-org/languagetool

https://hub.docker.com/r/erikvl87/languagetool


# Introduction


I completed the Firefox add-on installation after which I went to the Preferences > Experimental settings in order to add my own server URL. If you are using a custom URL make sure to add the /v2 path at the end like this : http://customlocalserver.lan:8010/v2 Keep in mind that Safari needs HTTPS, so you might need a reverse proxy to run it, oh Apple...

This tool does automatic spell checking whenever you are writing something in a multiline box on your browser, where admittedly most writing is happening nowadays, that includes Gmail or in my case Nextcloud. The interface is clean and comprehensive, giving you one click suggestions and a nice circular checkmark icon on the bottom right of text boxes that turns red once you make a mistake.



### Install
curl -L https://raw.githubusercontent.com/languagetool-org/languagetool/master/install.sh | sudo bash



CHECK API
http://10.1.1.23:8010/v2/check?language=en-US&text=my+text
https://www.youtube.com/watch?v=ca2JmSZEjfk&t=97s



### Vscode
LanguageTool Linter
https://marketplace.visualstudio.com/items?itemName=davidlday.languagetool-linter
https://github.com/davidlday/vscode-languagetool-linter


## Conclusion


## Check out other posts in the series
{% include post-series.html %}