---
sidebar_position: 1
---

# Introduction

Within the [Getting Started](https://docs.cosmwasm.com/1.0/getting-started/intro) section
we demonstrated and performed all necessary setup that is required to use and develop
Cosmwasm smart contracts. In this tutorial we will undertake writing our own smart contract
from scratch.

## Aims of the Tutorial

Within this tutorial we will be building a smart contract for a polling application (similar to [strawpoll](https://www.strawpoll.me/)).
Its functionality will be:

- Any user can create a poll
  - Polls are identified by a unique string, (this could be a UUID or a slug)
- Any user can vote on a poll
  - Ballots are stored per poll per address, meaning a user's vote can be updated any time
- Polls can be defined with many different options
  - The maximum options defined will be 10 due to gas limitations
- Any user can query and retrieve all the polls
- Any user can query and retrieve one poll by its unique identifier

A walkthrough of this flow is provided below:

1. User A creates a poll it has the following details:

   - "What is your favourite programming language?"
     - Rust
     - Go
     - JavaScript
     - Haskell

2. User A decides to vote on their own poll, they vote for Rust
   - Rust now has 1 vote, the rest have 0 still
3. User B also decides to vote, they vote for Go
   - Go now has 1 vote, Rust also has 1 vote, the rest have 0 votes
4. Some time passes and User A decides to change their vote to Haskell
   - Rust now has 0 votes (as User A's vote has been changed), Haskell has 1 vote (User A's new vote),
     Go still has 1 vote and the rest are on 0
5. User C decides to query the poll and check the results they see the following:
   - "What is your favourite programming language?"
     - Rust - 0
     - Go - 1
     - JavaScript - 0
     - Haskell - 1

TODO: Github link

:::tip Reminder
Ensure your environment is setup accordingly by following the [Getting Started](https://docs.cosmwasm.com/1.0/getting-started/intro)
section. Also verify that your chosen editor is setup for Rust development correctly using the [Setting up your IDE](https://docs.cosmwasm.com/1.0/getting-started/installation#setting-up-your-ide) section.
