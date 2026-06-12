# Project Notes

## Context

This file is meant to help with early project setup. We can use it for task lists,
design notes, status and bookkeeping, etc., but it is not meant to remain a permanent
part of the project. It's fine for now, though.

This project is about creating a ToDo service, to help me organize. There are hundreds
of ToDo services and tools already, but I want to create a new one. Here are some of
the reasons why:

 * control over feature set and code base
 * ensure my data stays private
 * work with ToDo tasks in various contexts (probably by working with labels)
 * have a REST API for easy integration into other tools
 * have an MCP server for easy integration into AI tools
 * minimal footprint, reduce attack surface
 * easy to backup, easy to restore, easy to deploy

## Approach

I want to approach this project looking at various aspects. Each should be discussed
and documented, describing the decisions, so that together they give enough context
to implement all the parts of the project, in the way I want them.

Of the top of my head, here is what we should look at:

 * technologies to use
 * code architecture
 * feature set
 * design principles
 * project goals
 * quality assurance
 * build automation
 * CI/CD
 * deliverables and artifacts
 * data migration
 * REST API design
 * MCP server design
 * security architecture
 * code reviews
 * dependency management
 * watchdog mechanisms
 * encryption
 * additional information for tasks, including BLOBs
 * labels / topics
 * task contexts
 * auto-generated tasks
 * manual and automatic task reviews (e.g. classification, status updates, etc.)
 * web frontend
 * Signal integration
 * Claude Code integration
 * command line interface
 * authentication
 * priorities
 * triage / curation
 * auto-closing tasks
 * supply chain security
 * AI assisted tasks
 * duplication detection
 * secret management
 * acceptance criteria
 * task comments and/or notes
 * code coverage
 * concurrency testing
 * locking logic verification
 * gerkin specs

## AI Tooling

 * Skills
   * Dependency Updates, including Golang (do we need AI for this?)
   * Run tests (do we need AI for this?)
   * Build everything (do we need AI for this?)
   * Checklist for commits (tests ran, documentation is updated, version is bumbed, backwards compatiblity, ...) (do we need AI for this?)
   * Bundle a release and publish (do we need AI for this?)
   * Security analysis, including CVE scan (do we need AI for this?)
   * Documentation Review
   * Search for missing tests
   * Spec vs Code Comparer
 * Agents
   * Architect discussion partner
   * Security analyst
   * Quality Assurance engineer
   * Technical Writer
 * Bots
   * Daily Maintenance
   * 

