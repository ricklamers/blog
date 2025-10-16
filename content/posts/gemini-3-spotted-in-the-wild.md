---
title: "Gemini 3.0 Spotted in the Wild Through A/B Testing"
date: 2025-10-16T12:00:00+02:00
tags: ["ai", "llms", "gemini"]
author: "Rick Lamers"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Testing Google's highly anticipated Gemini 3.0 through AI Studio's A/B feature using SVG generation as a quality proxy"
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

So I kept reading rumors that Gemini 3.0 is accessible through Google AI Studio through A/B testing and the SVGs folks were posting (of Xbox controllers in particular) made me think that they might be right.

Gemini 3.0 is one of the most anticipated releases in AI at the moment because of the expected advances in coding performance.

Evaluating models is a difficult task, but surprisingly the SVG generation task seems to be a very efficient proxy for gauging model quality as [@simonw](https://simonwillison.net/) has shown us using his "pelican riding a bicycle" test.

Lo and behold, after trying a couple of times I got the A/B screen and got an SVG image of an Xbox 360 controller that looked VERY impressive compared to the rest of the frontier.

![Gemini 3.0 Xbox controller output](/gemini-3-xbox-controller.png)

The exact prompt I used:

````
Create an SVG image of an Xbox 360 controller. Output it in a Markdown multi-line code block.
Like this:
```svg
...
```
````

For what it's worth the model ID for "Gemini 3.0" was `ecpt50a2y6mpgkcn` which doesn't really help understand which version of the model it is. Perhaps since I user selected Gemini 2.5 Pro it is actually Gemini 3.0 Pro that it is pitted against, as comparing Gemini 3.0 Flash to Gemini 2.5 Pro in an A/B test makes less sense to me. Also, it had about 24s higher TTFT and output length was about 40% longer (this includes reasoning tokens AFAICT), but that doesn't say much other than it's likely not a "GPT-5 Pro" type answer that uses significant test time compute.

## Appendix

"Gemini 3.0" A/B result versus the Gemini 2.5 Pro model:

![Gemini 3.0 vs Gemini 2.5 Pro comparison](/gemini-3-vs-2-5-comparison.png)

