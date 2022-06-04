+++
title = "Snowplow to Segment"
date = 2021-09-10T10:24:28+02:00
description = "How we've decided and executed migration from snowplow to segment"
tags = [
    "thumbnail",
]
thumbnail = "images/snowplow-to-segment.png"
+++

## Introduction

It's common for companies to use tracking systems to enable analytics. While sometimes positive, this practice is often detrimental — Google, we're looking at you. If you want to know your customer and their needs without bothering each one of them, you can turn to analytics tools that, thankfully, don’t have to be built in-house. Many SaaS products rise to the challenge of indicating customer preferences. It's good to have a myriad of options, as competition demands the best.

## Basics Snowplow and Segment

When selecting a SaaS tool for customer analytics tracking, two sit top of mind — Snowplow and Segment. Snowplow is oriented on the engineering side of the market and has features that enable contract testing through the user’s SDK. Snowplow provides quick and robust transfers of event data from source to destination and offers reliable delivery guarantees.

Another contender, Segment, is geared towards less tech-savvy, more managerial-focused individuals. On the surface, Segment boasts a fancy interface and real-time event analytics. Upon closer inspection, however, you’ll see how powerful the system is. It contains some cool features for developers, including a really expressive API for interacting with their schemas and data validation in real-time.

## Why we choose Segment

For years, my Contentful team and I used Snowplow. Only recently did we begin evaluating and considering a jump to Segment. In the beginning, those around us weren't sure about why we chose one over the other, but our team knew. Certain features really impressed us. So much so that we made the costly but entirely correct decision to migrate to Segment. Among its killer features, I’d like to call out the following:

- Real-time data validation against the schemas (with flexible configuration on how to react to any of the violations)
- User-friendly interface that allows our marketing team to run quick experiments without involving engineers
- Native connection to dozens of the services — we were even able to find Google Analytics

## What we've built to migrate smoothly

### CLI to programmatically work with Segment APIs

When we were working with Snowplow, the way we managed event schemas was through a git repository. That's the way we used to do it, and since people responsible for event data and its consistency are developers and analysts, who are comfortable and like working with git. And since we wanted to continue doing that — we thought about the way of doing it.

Segment exposes a nice API to work with tracking protocols. We decided to build a small tool that allows us to manage schemas. The tool is a CLI that has four commands:

- `pull-schemas`: pulls schemas from Segment into a folder, parsed as separate JSON files
- `push-schemas`: transforms JSON files into tracking plans and pushes them into Segment
- `test-schemas`: tests JSON files for validity (runs in the CI, so we do not push to Segment invalid schemas)
- `from-snowplow`: transforms Snowplow schema into Segment one.

We worked out a pretty good way of finding collaborators by making it in Typescript, which also enabled us to define types for schemas and events in particular, e.g.:

```
export interface TrackingPlan {
  created_time: string // date
  display_name: string
  id: string
  name: string
  rules: TrackingPlanRules
  updated_time: string // date
}

export interface TrackingPlanRules {
  events: Array<Event>
  global: null | Record<string, unknown>
  group: null | Record<string, unknown>
  group_traits: Array<string>
  identify: null | Record<string, unknown>
  identify_traits: Array<string>
}

export interface Event {
  description: string
  name: string
  rules: EventRules
  version: number
}

export interface EventPropertyObject extends Record<string, unknown> {
  [PropertyKey: string]: EventProperty
}
```

This one is convenient, and we plan to open source it when it’s groomed. The main purpose of this CLI is to be used by our CI pipelines to maintain our event tracking schema registry. When people want to add/remove/update some tracking plan, they go to our registry repo, do necessary changes there — and the CLI tests whether the changes are valid JSON schemas and deploys to our workspace in the Segment app.

### Segment Test Server

Small services, something that our teams spin up in the docker, runs in the end-to-end tests. It pulls schemas from the Segment, exposes the REST endpoint and validates payloads against the pulled schemas. Tiny and lightweight, this thing is handy during migration and increases the stability of our services. The idea is pretty simple and sums up in JSON validations using the ajv JS library, which is also used underneath by Typewriter. That enables us to emulate real work of Segment and make us sure that data will be accepted or rejected by it. The example of validation is following:

```
export const validate = (msg: object, schema: object): string => {
  const ajv = new Ajv();
  const validate = ajv.compile(schema);

  const valid = validate(msg);
  if (!valid) {
    console.log(`validate.errors: `, validate.errors);
    return JSON.stringify(validate.errors);
  }
  return 'valid!';
}
```

### Segment Alerts

Segment offers a validation of events, which is important. Receiving notifications when any violations occur is of equal importance. Segment has email notifications built-in and, while useful for some users, they may be ignored by those who let emails pile up or simply ignore them. With the practices of these users in mind, we made an alerting service that aggregates alerts over a small window of time and sends them to slack, tagging the responsible for the event in question.

Aggregated alerts are more informative — and hopefully less ignored — than those baked into Segment. With these alerts, people see what’s wrong and their responsibility in the matter in a format that’s more accessible.

## What were the challenges

![Challenges are everywhere](https://memegenerator.net/img/instances/75574353/challenges-challenges-everywhere.jpg)

### Dashboards

During our Snowplow years, we built dashboards and analytics on the data coming from there. We had more than 500 queries that used Snowplow data, and we had to migrate those somehow. And even though we were able to partially automatize that process using the API of our analytics tool, there were challenges since we've decided to take an opportunity and make our tracking better during the migration. Many events changed the schema and, besides, we decided to get rid of generic events by carefully splitting them into proper schemas. That made the data model way better, but it required a lot of manual work to migrate all the dashboards.

That people were relying on those dashboards, making it even more complicated. In many cases, issues that were brought by migration were treated as urgent bugs and because they blocked everything else.

### Product App Tracking

Since tracking was happening in many places, migration to the new library was complicated. Many teams work on the same project and, sometimes, they have different approaches to solving challenges. We had to come up with some sort of mid-ground solution. Since migrating from one SDK to another is a challenging and long-lasting process, we need to make it smooth. To make it smoother, we've merged both SDKs into a package and, for each call to that package, we were sending tracking events to Snowplow and Segment simultaneously. That made migration easier for our product team but harder for the data team since we were ought to maintain (and maintain for a long time) both pipelines. Those pipelines sometimes produced different results and, in many cases, it was kinda expected. Neither service guarantees delivery order of the events. In the end, it was fine.

### Learnings

**Avoid writing dirty code.** Ok, sometimes you can. Sometimes it's unavoidable, or the counter costs are too high. Sometimes you're in the rush of delivering a POC. But if it's something you will live with for years, invest time into making it clean and structured. Data is more about structure than any other software engineering branch. If you start piling various things into some generic table, you will end up with having people build some logic on top of it and, when you'll want to refactor it, the cost will be sky-high.

**Avoid extra mental load for users.** Data is widely used across the whole company. HR, SEs from other departments, and especially people in marketing use it daily. If the tool has a steep learning curve, it will make the life of the data department more miserable. You will need to invest in educating your users, helping them out, or even doing things that you usually shouldn't do if you had the right tool. So think about your end users when choosing the tool to use.

**Do not ignore warnings.**
![Warnings](https://i.redd.it/vinhvmejba331.jpg)

Engineers tend to ignore warnings because in many cases those are either hard to fix or do not influence anything actually and our brain just starts filtering out those yellow lines in the console. But with data, it's a bit different. With data, those warnings will live with you till the end of time (or till the end of life of your persistence layer) unless you'll fix them. And sooner or later you will need to fix them and that's what we were ought to do with our tracking data since we were not strict on passing the data and allowed systems to send us malformed/incomplete data and in some cases, it lasted for months, until we've started treating _just warnings_ as real errors.

## Conclusion

In the end, we're happy with the decisions made. While we could've made them sooner, which would have made migration easier, the most important thing is that we now have a reliable system that offers insights into customer behaviors. We have a schema registry managed in git, good and spot-on alerts on the violation, and a reliable mechanism to test all changes.

Besides that, Segment provides a nice tool, called Typewriter, that does type checking and autocompletion, that our product engineers happily use.
