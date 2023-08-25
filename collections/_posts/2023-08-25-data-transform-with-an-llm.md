---
layout: post
title: "Data transform directly with an LLM"
date: 2023-08-25T10:00:00Z
authors: ["Vijay"]
categories: ["Data Platform", "LLM", "Data Engineering"]
description: "Using an LLM to transform JSON to CSV"
thumbnail: "/assets/images/gen/blog/data-transformation-with-llm.jpg"
image: "/assets/images/gen/blog/data-transformation-with-llm.jpg"
comments: true
---
**Warning:** I do not necessarily advocate this approach unless accompanied by appropriate quality testing and risk analysis. LLM performance will improve over time, but there is always a chance of non-deterministic behaviour - that's inherent in their design. An optimised prompt could work 95% of the time and then randomly throw out garbage despite heavy prompt engineering.

## Intro
In the following post, I explore the opportunity to do no-code conversions and basic transforms on data with the gpt-3.5-turbo LLM. I'll take some data in JSON form and convert it to CSV, and translate some of the data along the way to something more useful - all via an LLM prompt.

My overall conclusion from this experiment: currently, I would prefer to use prompt-generated code that can be proven to be deterministic and then use that code for transforms. See my [previous article](https://www.spiritedtechie.com/blog/2023-08-22-data-transforms-code-gen-with-llms/).

## Source code
Can be found here - it's pretty small.

[Github](https://github.com/spiritedtechie/weather-sage/tree/main/api/experiments){:target="_blank"}

## Transform weather forecast data
TLDR: The transform to CSV worked! I needed a prompt with one-shot examples to guide the LLM in places.

### Pass 1:
Dump a full version of the JSON in the prompt context. It looked like [this](https://github.com/spiritedtechie/weather-sage/blob/main/api/data/met_office/sample_forecast_data.json){:target="_blank"} - you can see it has many days of data. 

However, there is an immediate problem: many rows were missing along with columns - this was due to token limitations (4,096 tokens for gpt-3.5-turbo). I suspect gpt-3.5 returns its best effort under these circumstances.

The results:

{% highlight csv %}
Day,Feels Like Temperature (C),Wind Gust (mph),Screen Relative Humidity (%),Temperature (C),Visibility,Wind Speed (mph) <-- MISSING COLUMNS
...
2023-08-16,17,9,73,17,VG,4
2023-08-17,15,9,82,16,VG,4
2023-08-17,14,9,92,15,VG,4
2023-08-17,14,13,94,15,GO,7
2023-08-17,18,18,77,19,VG,9
2023-08-17,20,22,57,22,VG,11
2023-08-17,22,20,48,24,VG,11
2023-08-17,22,20,60,22,VG,11
--> MISSING ROW
2023-08-18,14,18,87,16,VG,9
2023-08-18,13,18,92,15,VG,9
2023-08-18,14,20,92,16,GO,11
...
{% endhighlight %}

### Pass 2:
To solve the previous problem, reduce the JSON to only a single day's data. The rows were all there, but some columns were still missing.

[Here](https://github.com/spiritedtechie/weather-sage/blob/main/api/experiments/sample_forecast_data_slim.json){:target="_blank"} is the slimmer JSON used.

### Pass 3:
Next, the focus was on prompt tuning to solve the problem in pass 2, and develop a more complex transformation:

- Ask to include all columns
- Ask to calculate the date-time

Note: I included a one-shot example in the prompt to get the date-time calculation consistently accurate.

[Here](https://github.com/spiritedtechie/weather-sage/blob/main/api/experiments/sample_forecast_data_slim.json){:target="_blank"} is the JSON used (same as pass 2).

The results:

{% highlight csv %}
DateTime,Feels Like Temperature (C),Wind Gust (mph),Screen Relative Humidity (%),Temperature (C),Visibility,Wind Direction,Wind Speed (mph),Max UV Index,Weather Type,Precipitation Probability (%)
2023-08-25 06:00:00,10,13,90,10,MO,WSW,2,1,3
2023-08-25 09:00:00,16,11,70,16,VG,WSW,4,3,0
2023-08-25 12:00:00,17,13,51,18,EX,WSW,9,4,5
2023-08-25 15:00:00,16,18,56,19,VG,SW,11,2,31
2023-08-25 18:00:00,14,13,85,15,GO,WSW,7,1,81
2023-08-25 21:00:00,13,11,91,14,VG,WSW,7,0,50
{% endhighlight %}

### Final prompt
{% highlight text %}
Here is some JSON data.
Each Rep becomes a row.
Each Rep has a '$' field which represents the "minutes from midnight" from the Period date.
You have to calculate the actual date-time using the Period date and "minutes from midnight".
For example, if the Period date is 2023-07-10, and the $ value is 540, this represents 2023-07-10 09:00:00.
---------
{json}
---------
Include the CSV on a single line.
Include the header row with all field names and units.
Include the calculated DateTime for each row.
{% endhighlight %}

With this, we have a concise natural language prompt that could transform the weather forecast JSON to CSV without code. More complex transforms on the data could easily be prompted with one or more examples in the prompt if needed.

## Limitations
An immediate limitation was the context length (gpt-3.5-turbo = 4096 tokens). For lots of data, the process would be to chunk the data, process each chunk and recombine. Langchain has a pattern for this: [MapReduceChain](https://api.python.langchain.com/en/latest/chains/langchain.chains.mapreduce.MapReduceChain.html){:target="_blank"}.

Here were the average per-request costs:
- 6 rows of forecast data
- Tokens Used: 1046
- Prompt Tokens: 804
- Completion Tokens: 242
- Total Cost (USD): $0.00169

Let's do some rough math to understand scale.

Let's assume we continued with gpt-3.5-turbo. If there were 1000 rows of data (versus the 6 in my experiment), the costs are (1000 / 6)*$0.00169 = $0.28. 

If there were 100,000 rows, this would cost $28. Things start pricing up quickly on a relatively small dataset - this is still less than 20MB in size. 

There are higher token context options like gpt-3.5-turbo-16k, which scale well:

- 4K context	$0.0015 / 1K tokens	$0.002 / 1K tokens
- 16K context	$0.003 / 1K tokens		$0.004 / 1K tokens

The prices would soon add up if running this on much larger event data with wide columns and over 100,000 rows generated regularly. Private LLMs become the go-to option in these cases.

## Er, wait though ...

Could the LLM be performing well because this Met Office data schema is public knowledge and well-documented? At the very least, 'weather' is a widely documented concept. The prompt was not exactly light, but the LLM might have been consistently doing well because it knows the structure and public-knowledge domain to make the right leaps. 

I ran another experiment on a similarly structured dataset with an entirely made-up (meaningless) domain.

## Transform custom domain data
[Here](https://github.com/spiritedtechie/weather-sage/blob/main/api/experiments/sample_data_madeup.json){:target="_blank"} is the customised JSON used. All the JSON field names have been changed.

The LLM needed slightly more prompt guidance on mapping field code names to meaningful column names. Also, the prompt includes mapping the 'H' field's code values to word values. Besides this, the prompt changed very little, and the LLM successfully performed the transformation.

The prompt:

{% highlight csv %}
Here is some JSON data.
Each Block becomes a row.
Each Block field code can be mapped to a meaningful label in the Dict Map.
For example, field with code 'A' becomes column 'Feels'.
Each Block has a 'tm' field which represents the "minutes from midnight" from the Segment date.
You have to calculate the actual date-time using the Segment date and "minutes from midnight".
For example, if the Segment date is 2023-07-10, and the 'tm' value is 540, this represents 2023-07-10 09:00:00.
Map the 'H' field like follows: 
- 'GO' maps to 'Good'
- 'VG' maps to 'Very Good'
- 'P' maps to 'Poor'
- 'A' maps to 'Average'
- 'F' maps to 'Unknown'
---------
{json}
---------
Include the CSV on a single line.
Include the header row with all field names.
Include the calculated DateTime for each row.
{% endhighlight %}

The results:

{% highlight csv %}
Feels,Bend,Creep,Definition,Each,Finder,Grip,Rating,Indicator,Jump,DateTime
5,15,20,Apple,90,4,15,Good,15,1,2023-08-14 06:00:00
6,15,20,Banana,90,4,15,Very Good,15,1,2023-08-14 09:00:00
8,15,20,Pear,90,4,15,Poor,15,1,2023-08-14 12:00:00
12,15,20,Raspberry,90,4,15,Good,15,1,2023-08-14 15:00:00
11,15,20,Blueberry,90,4,15,Average,15,1,2023-08-14 18:00:00
8,15,20,Melon,90,4,15,Unknown,15,1,2023-08-14 18:00:00
{% endhighlight %}


## Non-determinism
The more you prompt an LLM, the more deterministic the results. The problem is that you never really know how deterministic it is - i.e. the probability. This works for many use cases where the results can be creative.

LLM usage needs a high amount of critical thinking. Much of the cool stuff you see promoted works because it applies LLMs over publicly documented domains with well-known language and concepts. LLMs are trained on a vast corpus of this knowledge. And for many use cases, this is great - it will drive automation and productivity with low effort. 

A public LLM could perform less well for domain-specific things where language is more customised and contextualised. Is a __dog__ an animal or a brand of beer? - If a non-contextualised LLM incorrectly thinks it's an animal, imagine what kind of response it could give. It may do well, it may not.

With context lengths growing, the contextualisation can be done at runtime (via the prompt) to a limit. For large domains, with lots of domain-specific data, we are better off fine-tuning LLMs before deployment. 


## Conclusion
For now, though, I'll be sticking to having transforms in deterministic code. However, a friendly LLM can certainly help me write that code. I fully expect my current understanding will get blown away as LLMs evolve.

It would be super cool to explore fine-tuning an LLM to domain-specific data and see what powerful features could result. I could also extend the transformation problem and use a more powerful LLM like gpt-4 to see how it performs.

In my next project, I will create a small app that can upload some data and use an LLM to identify patterns of data quality problems in the data. Such a tool could be useful as part of a data platform to flag potential data problems for upstream action.