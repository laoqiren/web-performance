# 高性能JavaScript

JavaScript诞生于1995年，最开始设计它只是用于网页上简单的处理，并没有考虑其执行速度。然而，由于各大浏览器厂商之间的不断竞争(浏览器大战：[https://en.wikipedia.org/wiki/Browser_wars](https://en.wikipedia.org/wiki/Browser_wars))使得js内核不断优化，以及一群爱折腾的开发者，他们用js去尝试做更多的事，如用Node.js进行服务端开发，用electron进行桌面应用开发，甚至将其用于loT等领域使得人们对js的性能要求越来越高。如此js的性能正不断改善。可以说，JavaScript绝对是无比幸运的。


在之前的网页渲染原理中我们知道，浏览器内核包括了渲染引擎和js引擎，常见的js引擎：
* Rhino，由Mozilla基金会管理，开放源代码，完全以Java编写。
* SpiderMonkey，第一款JavaScript引擎，早期用于Netscape * Navigator，现时用于Mozilla Firefox。
* V8，开放源代码，由Google丹麦开发，是Google Chrome的一部分。
* JavaScriptCore，开放源代码，用于Safari。
* Chakra (JScript引擎)，用于Internet Explorer[11]。
* Chakra (JavaScript引擎)，用于Microsoft Edge。
* KJS，KDE的ECMAScript／JavaScript引擎，最初由哈里·波顿开发，用于KDE项目的Konqueror网页浏览器中。

本章将从js内核原理、编码实践等方面去理解JavaScript性能问题。