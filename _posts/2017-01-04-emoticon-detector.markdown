---
layout: post
title:  "Detecting Emojis in a String by State Machine"
date:   2017-01-04 01:02:58 +0530
comments: true
categories: java state-machine regex
---

The message texts on social media contain emojis most of the time. And these emojis help identify the emotions and sentiment conveyed in the message. Not only to identify the sentiment we need to extract emojis from the text but to replace the emoji's symbol with actual emoji's gif or png (image) we need to detect the emoji symbols present in a given string.

Detecting Emojis is not as simple problem as just checking if any particular emoji is present in the text. That we can do by contains check. So I created [EmoticonDetector][emoticon-detector] in java.

The usage is simple. We have set of emoji symbols we need to detect. We construct EmoticonDetector for those emojis. And we can use the detector to fet the emoji details.

{% highlight java %}
Set<String> emojisToDetect = new HashSet<>; //All the emojis we want to detect
EmoticonDetector detector = new EmoticonDetector(emojisToDetect);
Set<Emoticon> detectedEmoticons = detector.detect("Hello World! :) :D");
{% endhighlight %}

The `detectedEmoticons` will have the emoji label as well as the starting and ending position index on the input string. Just like a regex matcher would return matched group.

[emoticon-detector]:https://github.com/yogin16/emoticon-detect