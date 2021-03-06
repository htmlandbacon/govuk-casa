[building-blocks-ab]: assets/plan-building-blocks-ab.png "a to b"
[building-blocks-ae]: assets/plan-building-blocks-ae.png "a to e"
[building-blocks-abc-oneway]: assets/plan-building-blocks-abc-oneway.png "a, b, c one-way route"
[building-blocks-route-condition]: assets/plan-building-blocks-route-condition.png "route condition"

# Designing a Route Plan

A _Plan_ in CASA simply describes the sequence of web pages that a user visits during their session.

The simplest plan is a linear, one-page-after-the-other affair. However, you can also go to town and design complex journeys that take the user off on tangents (and back again, or not) depending on the data they are providing during their session.

## Terminology

* **Plan**: The sum of all _Waypoints_, _Routes_ and _Origins_
* **Waypoint**: A visitable point in a user's journey through your plan (e.g. a UI that the user visits)
* **Route**: A connection between any two waypoints in the plan; can be directed either two-ways, or one-way between the waypoints
* **Origin**: A point within the Plan from which traversals will be started; at least one origin must be defined
* **Condition**: A boolean decision of whether a particular route will be followed or not during a traversal
* **Journey Context**: Some state information about the user's interaction with the plan (contains data and validation state)

Whilst the terms _Waypoint_ and _Page_ are interchangeable for the most part, it is not always the case that a _Waypoint_ will lead to one of your page form templates being rendered. For example, the `submit` waypoint in the [example application](deploying.md#1-plan) is actually delivered to the user by some custom route handlers, rather than CASA's default page-handling middleware. It is due to this subtle semantic difference that we tend to prefer the term _Waypoint_ when describing points along a user's journey.

## Simple linear routes

Connect two waypoints with a two-way route as follows:

```javascript
const { Plan } = require('@dwp/govuk-casa');

const plan = new Plan();
plan.setRoute('a', 'b');
plan.addOrigin('main', 'a');
```

> NOTE: an origin is mandatory; however, when only one origin is defined, its name is irrelevant

![simple two-way route between a and b][building-blocks-ab]

Note that the `next` and `prev` route labels are important, and used in certain forwards/backwards traversal scenarios.

For convenience, you can setup a sequence of two-way routes using the `Plan.addSequence()` method:

```javascript
const { Plan } = require('@dwp/govuk-casa');

const plan = new Plan();
plan.addSequence('a', 'b', 'c', 'd', 'e');
plan.addOrigin('main', 'a');
```

![sequence between a and e][building-blocks-ae]

You can also create one-way routes. This can be useful if you want to force a user's journey to go back to a different waypoint than the one just visited:

```javascript
const { Plan } = require('@dwp/govuk-casa');

const plan = new Plan();
plan.setRoute('a', 'b');
plan.setNextRoute('b', 'c');
plan.setPrevRoute('c', 'a');
plan.addOrigin('main', 'a');
```

![simple one-way route between a, b and c][building-blocks-abc-oneway]

## Route conditions

You can attach conditions against routes to control how the user traverses through them. By default, there are conditions attached to each route that will prevent them being traversed unless the "source" waypoint satisfys two conditions:

* The user has provided some data to store against that waypoint (i.e. they've submitted a non-empty form)
* There are no validation errors on that data

However, you can override these defaults with your own functions, which must match the following signature:

```javascript
/**
 * @param {object} route Information about the route being traversed
 * @param {JourneyContext} context Contextual state information
 * @returns {boolean} whether the route should be followed or not
 */
myCondition = (route, context) => {
  // perform checks
  return trueOrFalse;
};
```

See [Journey State](api/journey-state.md) for more information on how to use instances of the [`JourneyContext`](../lib/JourneyContext.js) class.

Here's a simple example that checks the journey's data context to determine whether one route or the other should be followed:

```javascript
const { Plan } = require('@dwp/govuk-casa');

const plan = new Plan();
plan.setRoute('a', 'b', (r, c) => (c.data.a.ticked === true));
plan.setRoute('a', 'c', (r, c) => (c.data.a.ticked !== true));
plan.setRoute('b', 'c');
plan.addOrigin('main', 'a');
```

When `ticked` is `true`, the traversal sequence will be: `a <--> b <--> c`

When `ticked` is `false` the traversal sequence will be: `a <--> c`

![conditional route][building-blocks-route-condition]

## Multiple origins

Origins are a useful tool to effectively segment a user's journey through your service into separate chunks.

For example, you might have a "spine" page that links to a series of separate workflows. In this scenario, you would define origins for the spine and each workflow:

```javascript
const { Plan } = require('@dwp/govuk-casa');

const plan = new Plan();

plan.addSequence('a', 'b', 'c');

plan.addSequence('name', 'age', 'dob');

plan.addSequence('outdoors', 'indoors');

plan.addOrigin('spine', 'a');
plan.addOrigin('user-details', 'name');
plan.addOrigin('hobbies', 'outdoors');
```

Now, a user can visit the following URLs to start traversing from that origin's start waypoint:

* `/spine/a` follows `a <--> b <--> c`
* `/user-details/name` follows `name <--> age <--> dob`
* `/hobbies/outdoors` follows `outdoors <--> indoors`

## Looping routes

TODO: Explain how to possibly use hooks to clear session data prior to sending user back to a waypoint to complete/restart a loop.

