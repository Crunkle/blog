---
layout: post
title: Optimizing the Minecraft server
published: false
---

As communities grow, more processing power is needed to hold large amounts of players on servers. It is sometimes more beneficial to optimize the server software instead of upgrading your hardware. I wanted to write up some of the changes I made to the Minecraft server to allow an extremely large amount of entities at a single point in time. Everything here needs to be taken with a pinch of salt as nothing was independently benchmarked, but the result was almost a 6x increase in entity performance (based on number of entities able to load before TPS drop).
