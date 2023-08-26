---
layout: post
title: "Generating data transform code with LLMs"
date: 2023-08-22T10:00:00Z
authors: ["Vijay"]
categories: ["Data Engineering", "LLM", "AI"]
description: "Put to shame by an LLM copilot"
thumbnail: "/assets/images/gen/blog/data-transform-with-llm.jpg"
image: "/assets/images/gen/blog/data-transform-with-llm.jpg"
comments: true
meta_og_image: "/assets/images/gen/blog/data-transform-with-llm.jpg"

---
As data engineers or analysts, we often receive unsuitable data formats. This article explores transforming data from the Met Office API using a Chat LLM Model (gpt-3.5-turbo) as a copilot. 

To be clear, I am not going full-blown no-code with this - it still involves generated code. In a [later post](https://www.spiritedtechie.com/blog/2023-08-25-data-transform-with-an-llm/), I will experiment with LLM's ability to transform data without code.

# Raw data

The data has a few challenges:
- Verbosity
- Meaningless fields
- Unfiltered
- Uncombined data

Here is a simplified version of the data. You can see it is still a relatively complex format, with field names mapped in a separate data structure (_SiteRep.Wx.Param_)

{% highlight json %}
{
   "SiteRep":{
      "Wx":{
         "Param":[
            {
               "name":"F",
               "units":"C",
               "$":"Feels Like Temperature"
            },
            {
               "name":"G",
               "units":"mph",
               "$":"Wind Gust"
            },
            {
               "name":"H",
               "units":"%",
               "$":"Screen Relative Humidity"
            },
            {
               "name":"T",
               "units":"C",
               "$":"Temperature"
            },
            {
               "name":"V",
               "units":"",
               "$":"Visibility"
            },
            {
               "name":"D",
               "units":"compass",
               "$":"Wind Direction"
            },
            {
               "name":"S",
               "units":"mph",
               "$":"Wind Speed"
            },
            {
               "name":"U",
               "units":"",
               "$":"Max UV Index"
            },
            {
               "name":"W",
               "units":"",
               "$":"Weather Type"
            },
            {
               "name":"Pp",
               "units":"%",
               "$":"Precipitation Probability"
            }
         ]
      },
      "DV":{
         "dataDate":"2023-08-22T10:00:00Z",
         "type":"Forecast",
         "Location":{
            "i":"351033",
            "lat":"52.3676",
            "lon":"-1.4702",
            "name":"COVENTRY AIRPORT",
            "country":"ENGLAND",
            "continent":"EUROPE",
            "elevation":"80.0",
            "Period":[
               {
                  "type":"Day",
                  "value":"2023-08-22Z",
                  "Rep":[
                     {
                        "D":"SSW",
                        "F":"16",
                        "G":"13",
                        "H":"91",
                        "Pp":"7",
                        "S":"0",
                        "T":"16",
                        "V":"GO",
                        "W":"7",
                        "U":"1",
                        "$":"360"
                     },
                     {
                        "D":"WSW",
                        "F":"18",
                        "G":"20",
                        "H":"70",
                        "Pp":"4",
                        "S":"9",
                        "T":"19",
                        "V":"VG",
                        "W":"7",
                        "U":"3",
                        "$":"540"
                     },
                     {
                        "D":"WSW",
                        "F":"19",
                        "G":"16",
                        "H":"56",
                        "Pp":"4",
                        "S":"9",
                        "T":"21",
                        "V":"VG",
                        "W":"7",
                        "U":"5",
                        "$":"720"
                     },
                     ...
                  ]
               },
               ...
            ]
         }
      }
   }
}

{% endhighlight %}

Note: The most complex part of this transformation is the _SiteRep.DV.Location.Period[0].Rep[0].$_ field. This location represents the "minutes from midnight" from the Period date. Therefore, each Rep is a 3-hourly block under that Period date. This fact is difficult to determine from the data (and indeed, the LLM needed prompting around this).

# Transformed data

I wanted to transform it to a single row representing the forecast matching the current date time:

{% highlight csv %}
Datetime, Feels Like Temperature - C, Wind Gust - mph, Screen Relative Humidity - %, Temperature - C, Visibility - , Wind Direction - compass, Wind Speed - mph, Max UV Index - , Weather Type - , Precipitation Probability - %
2023-08-22 09:00:00,18,20,70,19,VG,WSW,9,3,7,4
{% endhighlight %}

I could then load this to a database, or prompt a large language model for some generative AI use case without using up tokens unnecessarily.

# Two paths
1. Code manually and then use an LLM to optimise it
2. Use the LLM to write and optimise the code

My instinct was an LLM would struggle with path 2, but I was very wrong. I tested both paths to capture observations.

# Results
Overall, the LLM could generate new or optimise existing code with minimal need for changes once integrated into the codebase.

## Path 1
- Time to write manually: 5 minutes
- Time for LLM optimisation: 1 minute
- Total time: 6 minutes

I overengineered some things: separating parts into functions and not encapsulating the abstraction properly. I also favoured a more compositional approach to functions, which was overkill. I could have refactored my code to collapse complexity - but this would have added time (let's say another 5 minutes).

**Before**: 
[Source code link - manually written](https://github.com/spiritedtechie/weather-sage/blob/pre-llm-optimisation/api/transform/transform_forecast_data.py){:target="_blank"}

**After**: 
[Source code link - LLM optimised](https://github.com/spiritedtechie/weather-sage/blob/02b0172d93690d3a7d8049483f4548a4a0f86cad/api/transform/transform_forecast_data.py#L48C37-L48C37){:target="_blank"}

A few design-level issues remain since the LLM only optimised each function individually.

{% highlight python %}
def driver():
    object_list, object_keys = transform_to_list_of_json(api_response.json())
    object_list = filter_list_to_date_time(object_list, date_time)
    if not object_list:
        raise Exception(f"Data not found for date/time {date_time}")
    csv = convert_to_csv(object_list, object_keys)


def transform_to_list_of_json(data):
    def parse_datetime(date_text, mins_from_midnight):
        return datetime.strptime(date_text, "%Y-%m-%dZ") + timedelta(
            minutes=mins_from_midnight
        )

    code_mappings = _forecast_code_mappings(data)

    object_list = (
        {
            "Datetime": parse_datetime(period["value"], int(block["$"])).strftime(
                "%Y-%m-%d %H:%M:%S"
            ),
            **{
                code_mappings[key]["name"]: value
                for key, value in block.items()
                if key != "$"
            },
        }
        for period in data["SiteRep"]["DV"]["Location"]["Period"]
        for block in period["Rep"]
    )

    object_keys = ["Datetime"] + [value["name"] for value in code_mappings.values()]

    return list(object_list), object_keys


def filter_list_to_date_time(object_list, date_time: datetime):
    target_date_time = date_time

    matching_object = next(
        (
            obj
            for obj in object_list
            if _is_datetime_within_3_hours_from(
                reference_datetime=datetime.strptime(obj["Datetime"], DATE_TIME_FORMAT),
                datetime_to_check=target_date_time,
            )
        ),
        None,  # Default value if no match is found
    )

    return [matching_object] if matching_object else None


def _is_datetime_within_3_hours_from(
    reference_datetime: datetime, datetime_to_check: datetime
):
    window_start = reference_datetime
    window_end = window_start + timedelta(hours=3)
    return window_start <= datetime_to_check < window_end

{% endhighlight %}


## Path 2
- Time to prompt the LLM to write code: 2 minute
- Total time: 2 minutes
- Reduced lines of code versus path 1: ~100 (and removed 1 module/file)

The result was usable with only two minor tweaks to resolve some bugs. More importantly, the LLM did not over-engineer or overthink the task. It produced a single function that balanced readable code and performance.
 
[Source code link - LLM generated](https://github.com/spiritedtechie/weather-sage/blob/llm-generated/api/transform/transform_forecast_data.py){:target="_blank"}

{% highlight python %}
import csv
from datetime import datetime, timedelta
from io import StringIO


def transform(data):
    # Extract parameters from data
    parameters = data["SiteRep"]["Wx"]["Param"]
    parameter_names = [
        f"{param['$']} - {param['units']}" for param in parameters
    ]

    # Get the current datetime
    current_datetime = datetime.now()

    # Extract periods from data
    location = data["SiteRep"]["DV"]["Location"]
    periods = location["Period"]

    # Define a generator function to yield CSV rows
    def generate_csv_rows():
        yield ["Datetime"] + parameter_names
        for period in periods:
            period_date = datetime.strptime(period["value"], "%Y-%m-%dZ")
            for rep in period["Rep"]:
                period_datetime = period_date + timedelta(minutes=int(rep["$"]))
                if period_datetime <= current_datetime < period_datetime + timedelta(hours=3):
                    values = [rep.get(param["name"]) for param in parameters]
                    yield [period_datetime.strftime("%Y-%m-%d %H:%M:%S")] + values

    # Use the generator to create an in-memory string buffer
    csv_buffer = StringIO()
    writer = csv.writer(csv_buffer)
    writer.writerows(generate_csv_rows())

    # Get the CSV data as a string
    csv_data = csv_buffer.getvalue()

    return csv_data

{% endhighlight %}



# Conclusions
Overall, I am impressed with the LLM copilot for helping me with my data transformation code. It is an objective tool that can generate and optimise code based on short prompts and, at the very least, provide alternatives. It is a great way to learn a new programming language or features you may not know about in an existing language. 

Copiloting is much easier if you have written code before - it's a  much faster feedback loop as you scan the returned code for each iteration and decide on the next prompt.

I've always been a fan of pair programming, yet copilot replaces the knowledge transfer and smaller-scoped, code-level problem-solving traditionally performed during pair programming. Pair programming morphs into collaborations focused on contextual understanding, higher-level problem-solving, gathering domain expertise, solution design, strategic decisions and quality review.

Code can be subjective: LLM-generated code might spark debate over coding styles and what constitutes maintainability. In the previous examples, the LLM created code that utilised Python generators - a Pythonic approach - some will disagree that this is the most readable form. Production codebases should continue to be team-reviewed and alignment reached so that teams can support LLM-generated code.

# Conversations
I won't share all the raw chat conversations as they are lengthy.

For path 1 (optimising existing code), the prompts were entirely variations of: _Can you improve the code?_

For path 2 (LLM generated code), here is the ordered list of prompts. All were successful the first time, except one.

> Write some Python code that converts this JSON to CSV

> Can you improve it?

> Can you include the datetime of the period?

<span style="color:red">Almost: Did not include the Z in the date parser's formatter, so manually added this.</span>

> Thanks! Can you ensure the datetime includes the time? The time is defined by the $ field in each Rep, which indicates the minutes from midnight from the period's date.

> Can you improve this function?

> Can you further improve this function?

> Can you change it so it only includes the row matching the current date time?

> Can you ensure the current date time is within a three-hour window from the the period datetime?

<span style="color:red">Failed: I could not articulate the windowing concept, so manually corrected this.</span>

> Can you improve readability and performance?

> Can you change this slightly so CSV is output to a string instead of a file?

