---
title: "Open-Source Monitoring of Industrial Systems with InfluxDB using OPC-UA, Part 2"
date: 2019-10-28T14:01:49+01:00
draft: false
---

A little over 3 years ago, I released a [blog post]({{< relref "0417_BuildingAnOpenSourceProcessHistorian.md" >}}) about using open source tools like [InfluxDB](https://www.influxdata.com/products/influxdb-overview/) and [Grafana](https://grafana.com/) to replace the functionality of classical Process Historians, and by doing so opening up the collected data to anyone who needed it. The idea was to use the OPC-UA protocol as a way to get the data from industrial systems and send it to InfluxDB. Together with the post I also released the **node-opcua-logger** project on GitHub. 

Since then, a lot has been going on. People around me adopted the idea of using these kinds of open tools for industrial applications. As a result, I quit my job and started a company called [Factry](https://www.factry.io), where amongst other things we built the Factry Historian, a commercial product based on those ideas from 3 years ago. 

While developing the Factry Historian, we learned a lot and have implemented both new functionalities and a lot of improvements compared to the first release. As a lot of people still successfully use the community version of our data collector - and the code was getting a bit outdated - we felt it would be a good idea to trickle down some of our improvements to the original project.

So, long story short, **[v2-alpha](https://github.com/coussej/node-opcua-logger)** has been released on github! Below you can find an overview of the most significant changes.

### Pre-built binaries

You can now download **pre-built binaries for Windows, Linux and MacOS** instead of having to clone the project, which eliminates a lot of the hassle for people who are not experienced with git, nodeJS, ... You can find these binaries under the [releases](https://github.com/coussej/node-opcua-logger/releases) section of the github repository.

### Better buffering

In the previous version, all collected data was first written to disk in an applications specific database format before it was sent to InfluxDB. This is very disk intensive and has proven not to be a sensible approach. Therefore, buffering is now done in-memory as long as connectivity with InfluxDB is available. Only once connection-problems occur, the buffer is dumped on disk in JSON format. 

### Easier configuration
A lot of unnecessary config was replaced with sensible defaults (overridable in the environment), resulting in a simpler and cleaner config file. Furthermore, you can now choose to supply the config in either TOML or JSON, whichever you prefer.

### Cleaner code
NodeJS has also evolved a lot during the last few years, now fully supporting async/await. The opcua-logger code is now heavily using this abstraction from callbacks, resulting in much more readable code. 

### ...and more!
Please check it out for yourself, and perhaps contribute by submitting any issues or suggestions on GitHub!
