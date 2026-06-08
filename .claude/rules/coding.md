# Coding Rules

## Language & Coding Style

We use Golang. We use "gofmt" to enforce standard Golang formatting. We prefer
code that is easy to read over code that is "clever" or short. Readability
matters. Cognitive load matters. Keep it simple and straight forward.

## Dependencies

We limit our external dependecies, especially large frameworks. We want to
keep our exposure to external code low, and we don't want to depend on
other projects more than necessary. External dependencies should therefor
be "worth it" by implementing complex functionality, not just trivial
things or fluff, and external dependencies should be mature and reliable.
Gorm is an example for an acceptable external dependency. Anything
that uses node.js is NOT acceptable.

## Code Architecture

We use hexagonal architecture, which (among other things) means that we
add an abstraction layer between external systems and our code. (ports
and adapters)

It also allows us to have mocks and alternative implementations for
component and unit tests.

## Monorepo

This project is in a monorepo. The repo contains code for all related services,
build, test and deployment scripts, frontend, backend and tooling, all in one
place. This make it easier to make changes that touch more than one component
and keeps everything neatly together in one place.

