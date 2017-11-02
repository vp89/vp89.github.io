---
layout: post
title:  "Configuring Logstash to use multiple inputs"
date:   2017-11-01 16:50:00 -0500
categories: programming
---

A simple Logstash config has a skeleton that looks something like this:

{% highlight ruby %}
input {
    # Your input config
}

filter {
    # Your filter logic
}

output {
    # Your output config
}
{% endhighlight %}

This works perfectly fine as long as we have one input. Every single event comes
in and goes through the same filter logic and eventually is output to the same
endpoint.

What happens if we want to use the same Logstash instance to process other inputs?

Because the structure and content of these events may vary from input to input,
we probably don't want to put them through the same filter logic, and they may
even need to go to different outputs.

One way to solve this is to "decorate" each event with a type, depending on the
input it came in from. We can then have branches inside our filter and output
blocks that allow us to control how each of these types are processed.

{% highlight ruby %}
input {
    beats {
        type => "beats_events"
        # Add rest of beats config
    }
    jdbc {
        type => "jdbc_events"
        # Add rest of jdbc config
    }
}

filter {
    if [type] == "beats_events" {
        # Add beats event filter logic
    }

    if [type] == "jdbc_events" {
        # Add jdbc event filter logic
    }
}

output {
    if [type] == "beats_events" {
        # Add beats event output config
    }
    if [type] == "jdbc_events" {
        # Add jdbc event output config
    }
}
{% endhighlight %}

This solution may lead to large config files, but that is a fine trade off
for not having to manage multiple Logstash instances. You will also want to
test carefully especially when introducing new types and branches to ensure
you are not breaking existing pipeline functionality.