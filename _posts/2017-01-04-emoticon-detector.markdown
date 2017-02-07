---
layout:     post
title:      "Detecting Emojis in a String by a State Machine"
date:       2017-01-04 01:02:58 +0530
comments:   true
---
The message texts on social media contain emojis most of the time. And these emojis help identify the emotions and sentiment conveyed in the message. Not only to identify the sentiment we need to extract emojis from the text but to replace the emoji's symbol with actual emoji's gif or png (image) we need to detect the emoji symbols present in a given string.

Detecting Emojis is not as simple problem as just checking if any particular emoji is present in the text that we can do by contains check. So I created [EmoticonDetector][emoticon-detector] in java.

The usage is simple. We have set of emoji symbols we need to detect. We construct EmoticonDetector for those emojis. And we can use the detector to fet the emoji details.

{% highlight java %}
Set<String> emojisToDetect = new HashSet<>; //All the emojis we want to detect
EmoticonDetector detector = new EmoticonDetector(emojisToDetect);
Set<Emoticon> detectedEmoticons = detector.detect("Hello World! :) :D");
{% endhighlight %}

The `detectedEmoticons` will have the emoji label as well as the starting and ending position index on the input string. Just like a regex matcher would return matched group.

The Emoticon entity looks like:
{% highlight java %}
public class Emoticon implements Serializable {
    private static final long serialVersionUID = 42L;

    /**
     * Symbol
     */
    private String value;
    /**
     * The starting position in the string for this entity
     */
    private int start;
    /**
     * The ending position in the string for this entity
     */
    private int end;
    ...
}
{% endhighlight %}

The EmoticonDetector is similar to creating a regex for matching `emojisToDetect`, but as this is dedicated for detecting keywords it is faster than regex matching. Internally it computes using a state machine only.

State machine is dynamically created. We first figure out the _character set_ we would need to create state machine. We also need to label all _states_ we would need for given input set. One we have the _character set_ and _states_ identified we are good to create _transition map,_ that is the definition with each state with information for what would be the next state for each input char. Typical automata stuff.

This state machine would help to identify all the emoticons present in the text in **only one iteration.**

[emoticon-detector]:https://github.com/yogin16/emoticon-detect
