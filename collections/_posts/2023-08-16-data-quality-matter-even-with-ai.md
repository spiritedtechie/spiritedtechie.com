---
layout: post
title: "Yes, Data Quality Still Matters - Even More So With AI"
date: 2023-08-15T12:00:00Z
authors: ["Vijay"]
categories: ["Data", "Data Platform", "Data Quality", "AI", "LLM"]
description: "Manage data debt and improve data quality for more effective AI"
thumbnail: "/assets/images/gen/blog/data-debt-data-quality.jpg"
image: "/assets/images/gen/blog/data-debt-data-quality.jpg"
comments: true
---
Apologies if you know this already, but it is worth repeating: **data quality does not come for free**. As organisations use LLMs to interact with data, this realisation will become front and centre.

Data debt levels across the board are very high and have been for a while (at least in my experience) - it feels neglected.

Data debt is often higher than technical debt typically found in software web applications. And if you are running your own data platform, you could have multi-faceted technical debt spanning software, infrastructure, ops and data (amongst other things!).

Data debt tends to come from the need to unlock 'value' from data faster or scale types of data. Organisations make harsh trade-offs for the short-term gain. Commercial drivers are understandable, but organisations need to be explicit about what the trade-offs mean for them. Perhaps an organisation moves fast now but could be slower on something else later. Maybe all data delivery efforts will slow if the debt is in a core area.

I don't believe anyone wants to be in this position of having mountains of data (or any technical) debt; however, factors such as budgets, goals, tool and skills maturity, and hiring ability fuel increasing data debt and force trade-offs. Data debt transparency and continuous management (not ignorance!) are critical to moving ahead sustainably.

Examples of data debt:

* **Lack of meaning** - contextual knowledge, semantics, relationships & shared language.
* **Siloed meaning** - duplicated or different semantics across roles or departments
* **Low data testing** - no contracts defined or enforced, so low-quality data slips through to downstream consumers

These problems require humans to understand the data deeply and are usually domain-centric. It requires collaborative, cross-functional agreement and governance. The further upstream this is applied, the better.

Don't fall into the trap of thinking value from data comes for free - that technology alone is the silver bullet. Build the pipeline, throw some AI in the mix and data awesomeness will come - unlikely. Data quality is meaning from (and for) humans, documenting that knowledge and upholding adherence.

As AI drives more interest in interacting with data, the elephant in the room is emerging - data debt means AI is currently limited in usefulness until the core data has the appropriate quality standards. Data quality and semantic knowledge become a turbocharge for any AI technique such as LLMs.

As someone who has historically championed software quality, improving data quality is an ever-growing (and interesting) challenge. The renewed interest in interacting with data using AI will push a "back to the basics" approach to data quality in organisations - at least, I hope so. Data still feels like an immature area with so much potential to improve. There is plenty of challenge for any data engineer or analyst!

I like the data mesh model for scaling data quality, consisting of data platform tools that empower distributed data experts to own data quality. These tools allow cleaning & transforming data, capturing semantic knowledge, and governance without relying on centralised data teams. But if you are not there yet, and are a centralised data team, stick to the core principles anyway. Consequently, your ability to use AI with that data will be much smoother.