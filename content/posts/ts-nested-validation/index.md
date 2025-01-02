+++
title = 'How to validate deeply nested structures in Typescript, a modest proposal'
description = 'Or how to have fun with generics and generators'
date = 2024-12-30T18:08:37+01:00
type = "post"
draft = false
image = "free-photo-of-traffic-jam-on-highway-in-city.jpeg"
+++

_All the code plus unit-tests are available in the [github repo](https://github.com/FernandoAFS/ts-deep-validation)_.

<!--https://www.pexels.com/photo/traffic-jam-on-highway-in-city-20530375/-->

![Trucks neatly parked](pexels-photo-2800121.jpeg)
_Ready, set, go! ([pexels](https://www.pexels.com/photo/aerial-photography-of-trucks-parked-2800121/))_

# Introduction

No user is trustworthy. Every input users make must be double checked. React has multiple ways to go about this. There are the traditional [html client side validation](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Forms/Form_validation). In react there is [react controlled input](https://react.dev/reference/react-dom/components/input#controlling-an-input-with-a-state-variable). Plus other third party libraries such as [react-hook-forms](https://react-hook-form.com/) or [formik](https://formik.org/). In full-stack [zod](https://zod.dev/) for schema validation... 

<!--That try to expand on the limitations of html and to make the controlled component code more declarative and easier to maintain plus a ton of optimizations and extra features.-->

All this patterns and libraries are helpful but I found them lacking when the user had to interact with a more complex data structure. Let me show you with an examples.

# A real-world scenario

Congratulations, you have been tasked with writing the frontend of a truck company tool. The user must define routes.

There are the following entities with restrictions:

- **Village**: They are points in the map. There are source, sink and transit villages.
- **Road**: They are a list of villages connected with each other.
- **Route**: It's the trajectory of a truck. It's made of slices of roads.
- **Trip**: It has one main route and multiple alternative routes. It also has links between the two. Each sub-entity has the following restrictions:
    - Main route must start in a source and end in a sink village.
    - Alternative routes must end in a sink village
    - A point in the main route must one or none alternative routes links from it.

![Trip from Bilbao to Málaga](entitiesInMap.svg)
*Example of a trip with a main route from Bilbao to Málaga and two alternative routes with multiple links*

Let's say that the user defines the villages first, the roads seconds and then the Trips. 

# High level solution

1. Define data structures in detail.
2. Define error structure in detail. Let's call it `TripErrors`
This 
3. Write a function that receives a `Trip` data structure and returns a `TripErros`.

There are more ways about this. You may design the whole application around this requirements. Helping users not make mistakes is good practice but relying on that alone is dangerous since application design at large is much harder to test and will change over time.

## Main data types

This are definition of the structures:


```typescript
/**
 * Types required by faked API.
 */
export type VillageType = "transit" | "source" | "sink"

/**
 * Effectively, a point on the map
 */
export type Village = {
  uuid: string
  name: string
  villageType: VillageType
  // Would include coordinates...
}

/**
 * A list of villages connected.
 */
export type Road = {
  uuid: string
  villages: Village[]
}

/**
 * A segment of a road. Does not Segments are not re-usable. Belong only to a
 * single route (this is important for point identification.)
 */
export type RoadSegment = {
  uuid: string
  road: Road
  ndx0: number
  ndxF: number
}

/**
 * Path to be taken from point A to point B. Effectively, list of road segments
 */
export type Route = {
  uuid: string
  segments: RoadSegment[]
}

/**
 * Point in a given route. Segments are unique to a 
 */
export type RoutePoint = {
  route: Route
  segment: RoadSegment
  ndx: number
}

/**
 * Link used between a village in the main route and contingency route.
 */
export type RouteLink = {
  uuid: string
  route: string
  // Include condition to switch
}

/**
 * Top level definition of a given trip.
 */
export type Trip = {
  mainRoute: Route
  alternativeRotues: Route[]
  alternativeRouteslinks: RouteLink[]
}

```

## Error type

I will only make 1 assumption. A validation error happens when a property value is wrong, either because the value itself is not valid or because the combination of multiple values is not valid.

It's important to inform exactly what is wrong and why and this is specially true for relationships between properties. When two values that cannot be valid at the same time both are wrong, at least potentially wrong.

An error therefore is why the value is wrong and where the value is in the data structure.

A generic error type would have the same structure as the type that refers to but every value would be a string explaining the error and every key would be read-only and optional. 

```typescript
/**
 * Turn the nested object into a list of optional strings. Lists are turned
 * into an object with an optional "overall" error and a list of "values"
 * errors.
 */
export type ErrorType<T> = 
    T extends string ? string // TRICK TO AVOID TREATING STRING AS OBJECT
  : T extends number ? string
  : T extends boolean ? string
  : T extends symbol ? string
  : T extends Array<infer V>
    ? { [K in string]: ErrorType<V> } & { "overall": string }
  : T extends { [K in keyof T]: T[K] }
    ? { readonly [K in keyof T]?: ErrorType<T[K]> }
  : T extends { [K in string]: T[keyof T] }
    ? { readonly [K in keyof T]?: ErrorType<T[keyof T]> }
  : string;

type TripError = ErrorType<Trip>
```

The only exceptions I've made where to lists. 

I've included a potential `overall` key to the list errors since lists may have errors about it's length. 

Every list translates to an object instead of other list. This is fine as long as every list object is identifiable by some sort of primary key. An alternative would be to return another list of errors, to make every element nullable and to fill with nulls (or worse, empty objects) every time an element is fine. I find this solution much more inelegant and it could be harder to know when an object has no errors.

## Let's get to the logic

![Closed road](pexels-photo-9811329.jpeg)
_Don't try the spaghetti way, you have been warned! ([pexels](https://www.pexels.com/photo/road-sign-in-the-middle-of-the-road-9811329/))_

We have the source and the destination, now... How to get there?

What I don't want to do:

- Make the logic tightly coupled with the data structure. The logic should scale as the complexity of the data grows.
- We can only show one error per property at a time but tests should be able to see multiple errors at a time. A property may be wrong in more ways than one.
- Basic errors may prevent the fine-grained validation from taking place since wrong data will not be valid for any business-case requirement and trying to validate it may even make us throw an error.
- Make validation logic not related to object instantiation. This is more abstract but I would try to avoid to mix complex object creation and validation logic.

## Consider the following solution...

What I'm trying to do is good old _divide and conquer_.

We can create multiple [generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) for different entities and maybe even different error groups. Each `yield` would carry a chain of keys or _ids_ and a string error payload. If we have any very basic error just hit a `return` and wait for the user to input something else.

![Diagram of flattening the structure into tuples](diagram.svg)
_Going from data structure to a list of tuples with list of keys-errors and then finally create the error structure_

With generators we would solve all the pain points. We now need to:

- Define an `ObjecTuple` type... This will involve generics.
- Write our validation logic in `function*` type generators.
- Define a process to go from a bunch of tuples to the original `ErrorType`.

### Flattening objects. From nested structures to tuples.

```typescript
/**
 * Flattening of a type into a array of tuples.
 */
export type ObjTuple<T> = 
    T extends string ? [string]
  : T extends number ? [number]
  : T extends boolean ? [boolean]
  : T extends symbol ? [symbol]
  : T extends Array<infer V> ? [string, ...ObjTuple<V>]
  : T extends { [K in keyof T]: T[keyof T] }
    ? [keyof T, ...ObjTuple<T[keyof T]>]
  : T extends { [K in string]: T[keyof T] } ? [string, ...ObjTuple<T[keyof T]>]
  : never; // MAY GET VERY COMPLEX OTHERWISE.

/**
 * Shortcut for tuples of error type of a given generic. Used extensively in this case.
 */
export type ErrorTuple<T> = ObjTuple<ErrorType<T>>;

```

Simply put, this turns a nested object into a bunch of tuples, just like the diagram shows.

I would love to remove the `never` at the end of the type definition but in my experience I found that ending with a `[T]` results in may cases where the compiler is not sure about type safety. If find this implementation good enough for most cases. If I'm missing something please let me know.

### Validation logic as such

This should now be straight forward.

Let's start small. If a `RoadSegment` index is below 0 or points to a village out of it's road lists bounds then it must raise an error.

```typescript
/**
 * Checks that segment is valid for it's given road.
 * This is meant to be used to every route segment available.
 */
export function* validateSegment(
  segment: RoadSegment,
): Generator<ErrorTuple<RoadSegment>> {
  if (segment.ndx0 < 0) {
    yield ["ndx0", SEGMENT_NDX_BELOW_0_ERROR];
  }

  if (segment.ndxF < 0) {
    yield ["ndxF", SEGMENT_NDX_BELOW_0_ERROR];
  }

  if (segment.ndx0 >= segment.road.villages.length) {
    yield ["ndx0", SEGMENT_NDX_OUT_OF_BOUNDS_ERROR];
  }

  if (segment.ndxF >= segment.road.villages.length) {
    yield ["ndxF", SEGMENT_NDX_OUT_OF_BOUNDS_ERROR];
  }
  // NDX0 > NDXF IS ALLOWED SINCE ROADS CAN BE TRAVERSED BOTH WAYS.
}
```


Now, Every `Route` need to have validate every `RouteSegment`. On top of that the end of a road segment must be the same village as the beginning of the next route segment except the end of the route itself.

```typescript
/**
 * Validates basic validity of segment indexes and coherency of each segment
 * with the next one.
 */
export function* routeValidation(route: Route): Generator<ErrorTuple<Route>> {
  let ndxValid = true;

  for (const segment of route.segments) {
    for (const error of validateSegment(segment)) {
      yield ["segments", segment.uuid, ...error];
      ndxValid = false;
    }
  }

  // DON'T PROCEED. NEXT STEPS MAY BREAK IF SEGMENTS ARE INVALID.
  if (!ndxValid) {
    return;
  }

  const decorSeg = route.segments
    .map((s) => ({
      ...s,
      village0: s.road.villages[s.ndx0],
      villageF: s.road.villages[s.ndxF],
    }));

  // List of all the segments except first one with last one included
  const lastSegDecor = decorSeg
    .filter((_, ndx) => ndx > 0)
    .map((seg, ndx) => ({
      segment: seg,
      previousSegment: decorSeg[ndx],
    }));

  for (const { segment, previousSegment } of lastSegDecor) {
    if (previousSegment.villageF.uuid == segment.village0.uuid) {
      continue;
    }
    yield ["segments", previousSegment.uuid, "ndx0", SEGMENT_LINK_BEGIN_ERROR];
    yield ["segments", segment.uuid, "ndxF", SEGMENT_LINK_END_ERROR];
  }
}
```

<!--TODO: COMPLETE-->
We can see one of the superpowers of generators. If there is a broken segment at a logical level then the business logic validation would not take place for a given route. Skipping the broken segments may result in a more complete early error and therefore a better user experience but this keeps the logic simple and concise and would never result in a wrongly valid structure which is the priority. If completeness is important I would include a _set_ with all the wrong segments _uuids_ and i would decorate such segments to ommit every potentially wrong error in the end.

Now, on top of this the _main_ route needs to start in a source and end in a sink while alternate routes will only need to end in a sink.

![Truck moving to the hills](pexels-photo-93398.jpeg)
_This is you on your way to deliver a validated datastructure. ([pexel](https://www.pexels.com/photo/white-truck-on-black-road-1003868/))_

```typescript
/**
 * Validate that function is valid and it begins and ends where it should.
 */
export function* mainRouteValidation(
  route: Route,
): Generator<ErrorTuple<Route>> {
  const routeValidationErrors = routeValidation(route);

  let validRoute = true;
  for (const err of routeValidationErrors) {
    yield err;
    validRoute = false;
  }

  if (!validRoute) {
    return;
  }

  const s0 = route.segments[0];
  const sF = route.segments[route.segments.length - 1];
  const village0 = s0.road.villages[s0.ndx0];
  const villageF = sF.road.villages[sF.ndxF];

  if (village0.villageType !== "source") {
    yield ["segments", s0.uuid, "ndx0", MAIN_ROUTE_START_SOURCE_ERROR];
  }

  if (villageF.villageType !== "sink") {
    yield ["segments", sF.uuid, "ndxF", MAIN_ROUTE_END_SINK_ERROR];
  }
}
```

Once again. Invalid routes will not be further validated.

I won't show _alternate_ routes validation, I think you can all see what to omit from this last example.


Now, we are not validating routes, we have to validate `Trips`.

```typescript
/**
 * Top level trip validator generator
 */
export function* tripValidation(trip: Trip): Generator<ErrorTuple<Trip>> {
  for (const error of mainRouteValidation(trip.mainRoute)) {
    yield ["mainRoute", ...error];
  }

  for (const alternativeRoute of trip.alternativeRotues) {
    for (const error of alternativeRouteValidation(alternativeRoute)) {
      yield ["alternativeRotues", ...error];
    }
  }
}
```

Here we can see how trip validation may grow to include more and more details where more and more generators may be called. No need to be concerned about code-complexity scalability.

We now have a business logic we can test in detail but there is a missing link. We don't want to iterate over tuples on presentation.


### Some recursion to stitch everything together

I will provide the simplest way I came up with. I will iterate the tuples from top to bottom and left-to-right moving from lists to a recursive [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) structure where I keep the first error informed for each property. I assume that the first errors may refer to the most basic validations.

Then I would convert this nested `Map` to a js object recursively. This is done because `Map` is recommended when there is a lot of random key access.

First. This is to go from a tuple to a Map:

```typescript
export type ObjMap<T> = T extends string ? string // DIRTY TRICK...
  : T extends { [K in keyof T]: T[keyof T] } ? Map<keyof T, ObjMap<T[keyof T]>>
  : T;

/**
 * Typeguard that guarantees that the list has no elements
 */
function isEmptyList<T>(l: T[]): l is [] {
  return l.length <= 0;
}

/**
 * Typeguard to discriminate maps from leafs in recursive functions
 */
function notMap<K, V, T>(o: T | Map<K, V>): o is T {
  return !(o instanceof Map);
}

/**
 * Recursively transverse the error tuple.
 */
function recTupleMap<T>(
  tuple: ObjTuple<T>,
  map: ObjMap<T>,
): ObjMap<T> | null {
  // For readability.
  type V = T[keyof T];

  // The key value will always be keyof T when not in leaf condition. i don't
  // know how to coordinate both types in a more type-safe way without using
  // complex function overloading (too complex for typescript)
  type K = keyof T | string | number | boolean | symbol;

  // If it's not a leaf it's a node

  // If there is a leaf already then cancel insert of tuple. This shouldn't happen other than in leaf.
  if (notMap(map)) {
    return null;
  }

  const thisNode = map as Map<K, ObjMap<V>>

  const [k, ...vs] = tuple;

  // End of recursion condition. there is no more tuple to read so this is the
  // leaf.
  if (isEmptyList(vs)) {
    return k as ObjMap<T>;
  }

  const getNextNode = (): ObjMap<V> => {
    const existingNode = thisNode.get(k);
    // Create a new node if none have been there before.
    if (!existingNode) {
      return new Map() as ObjMap<V>;
    }
    return existingNode;
  };

  const populatedNextNode = recTupleMap(vs, getNextNode());

  // Abort on existing leaf, return as is.
  if (populatedNextNode == null) {
    return map;
  }

  thisNode.set(k, populatedNextNode);

  return thisNode as ObjMap<T>;
}
```

This function receives a tuple and a node (or map). If the length of the tuple is only 1 then we are on a leaf and returns it's value as the end of recursion condition. 

If the node it's a primitive then a value was put there before and returns a null to indicate that no writing should take place. Writing the last value would be simpler but potentially less useful.

If the tuple is longer than one then it will proceed to call itself. It will try to use the next node in line if there is one and then it will create a new one otherwise.

Now, the generator as such has it's own entry function:

```typescript
/**
 * Convert list of tuples to an object (top-bottom recursion)
 */
export function tuplesToObject<T>(
  tuples: Iterable<ObjTuple<T>>,
): T {
  type V = T[keyof T];

  const mainM = new Map<keyof T, ObjMap<V>>();
  for (const tuple of tuples) {
    recTupleMap(tuple, mainM as ObjMap<T>)
  }

  return recMapToObj(mainM as ObjMap<T>);
}
```

This is just iterating over every tuple for the same map.

Going from maps to object is:

```typescript
/**
 * Turn map type to object recursively
 */
function recMapToObj<T>(m: ObjMap<T>): T {

  // End of recursion condition.
  if (!(m instanceof Map)) {
    // Lazy casting. Assuming that if it's not a map it's a primitive
    return m as T;
  }

  function* entriesGen(){

    // Type guard...
    if (!(m instanceof Map)) {
      return
    }

    for (const [k, v] of m.entries()) {
      // Recursively generate objects
      yield [k, recMapToObj(v)]
    }
  }
  return Object.fromEntries(entriesGen())
}
```

Finally the final piece of the puzzle would be:

```typescript
/**
 * Generate new error types given a trip.
 */
export function tripErrors(trip: Trip): ErrorType<Trip> {
  const tripGen = tripValidation(trip);
  return validationToErrorObj<ErrorType<Trip>>(tripGen);
}
```

Just calling the top level generator and the function and this conversion from tuples to objects.

You may think that the juice isn't worth the squeeze. All this recursive _boilerplaty_ functions will only be written once but the validation may grow and grow over time. This allows me to focus on validation logic and presentation and that is what my users and customers need. All this glue will only be written once.

# Conclusion

Complex data validation must be treated as it's own library. Requirements will change and error will happen.

You may think the example is overly-convoluted. I can assure you that I've seen much worse and sometimes for good reasons. Re-defining business logic may result in more elegant code but we are not here for that. This may be exactly what the user needs and our job is not to write elegant code is to keep the users happy.

Quality is always a non-negotiable. Testing will result in quality and decoupling will allow for testing. I found that making the code testable fixes the quality problems before the actual testing even begins.

I would generally advice against trying to be too smart but don't fall for the [not invented here](https://www.joelonsoftware.com/2001/10/14/in-defense-of-not-invented-here-syndrome/). It's ok to have some fun as long as you are doing so for the user ;).

Don't be afraid of being creative!
