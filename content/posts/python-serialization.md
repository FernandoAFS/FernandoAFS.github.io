+++
title = 'Python Serialization'
date = 2023-12-25T20:22:55+01:00
type = "post"
draft = true
+++

# Motivation

## Pydantic and pickle

While working on a few projects I had to make a choice between attrs/cattrs, dataclasses and pydantic. I've been intrigued by the following line:

> Pydantic models support efficient pickling and unpickling [[1]](https://docs.pydantic.dev/latest/concepts/serialization/#pickledumpsmodel).

## Attrs, cattrs and pydantic

Learning how to use attrs I've comed upon [an article comparing attrs/cattrs with pydantic](https://threeofwands.com/why-i-use-attrs-instead-of-pydantic/).

The only benchmark provided is about class initialization.

> Instantiating these classes on my machine; attrs takes 953 ns +- 20 ns, while pydantic takes 3.06 us +- 0.07, so around ~3x slower.  What's worse, if we adopt a better approach - using Mypy to do the validation statically - the attrs approach can drop the validator to drop down to 387 ns +- 11 ns, while Pydantic needs to switch to using PydanticDatetime.construct (awkward to use) which still takes ~1.36 us, so again ~3.5x slower.

Aside from this there is a point made about how validation is opt-in


This post was written in 2021, before the release of pydantic v2. pydantic should perform much better now. In the [pydantic v2 plan](https://docs.pydantic.dev/latest/blog/pydantic-v2/) we can find:

> The core validation logic of pydantic V2 will be performed by a separate package pydantic-core which I've been building over the last few months. pydantic-core is written in Rust using the excellent pyo3 library which provides rust bindings for python.


>As a result of the move to Rust for the validation logic (and significant improvements in how validation objects are structured) pydantic V2 will be significantly faster than pydantic V1.

> Looking at the pydantic-core benchmarks today, pydantic V2 is between 4x and 50x faster than pydantic V1.9.1.

> In general, pydantic V2 is about 17x faster than V1 when validating a model containing a range of common fields.












