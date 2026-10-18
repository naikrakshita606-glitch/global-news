# global-news

## News collection and API flow

This document describes the intended architecture for a news collection service
that fetches articles from an external news API, stores them in MongoDB, and
serves them through a REST API. The current repository contains this
documentation only; it does not include the Python application, scheduler,
database configuration, or API implementation. Consequently, the implementation
details below are guidance, not claims about code currently present here.

### 1. End-to-end project flow

1. A scheduler waits for the configured collection time.
2. The scheduled Python job calls the external news API over HTTP.
3. Python validates the response, maps articles into the application's data
   format, and handles malformed records or request failures.
4. Python writes articles to MongoDB, ideally using idempotent upserts to avoid
   creating duplicates.
5. A REST API endpoint reads articles from MongoDB and returns them, typically
   as JSON.
6. Postman sends an HTTP request to the REST API and displays the response.

Postman is a client for testing the REST API. It does not normally fetch and
store the scheduled news itself.

### 2. Why use a scheduler?

A scheduler automates recurring collection so the job runs at configured
times without a person manually invoking it. For example, an in-process
scheduler such as APScheduler can register a Python callable (for example,
`fetch_and_store_news`) with a time-based trigger. The scheduler calculates
when the next run is due and invokes that callable from the running application
process. The trigger usually calls the function directly; it does not need to
send an HTTP request to it.

The exact scheduler, trigger, timezone, and execution settings depend on the
application configuration.

### 3. What happens if the service is down at the scheduled time?

An in-process scheduler cannot execute a job while its process is stopped. If
the service stops at 6 AM and the job is due at 7 AM, the 7 AM execution is
missed while the service is down.

After restart, what happens depends on scheduler configuration: it may skip the
missed run, run it as a misfire/catch-up, or apply a configured grace period.
A persistent job store can preserve job definitions, but persistence by itself
does not guarantee that missed executions will be replayed. Configure and test
the scheduler's misfire policy, or use an external scheduler/queue when durable
catch-up is required.

### 4. How to prevent duplicate articles in MongoDB

Use a stable article identity, preferably an ID supplied by the news provider.
If one is not available, a normalized canonical URL can be used. Create a
MongoDB unique index on that identity and write with an upsert so a repeated
article updates the existing document instead of inserting another.

An application-level "look up, then insert" check is not sufficient by itself:
concurrent jobs can both pass the check. The unique index provides the
database-level guarantee. Handle duplicate-key errors if concurrent writes
race, and decide whether repeated records should update fields or leave the
first stored version unchanged.

### 5. How news travels from the external API to Postman

```text
External news API
        | HTTP response
        v
Python fetcher and processor
        | validated, mapped, deduplicated writes
        v
MongoDB
        | query
        v
REST API endpoint
        ^ HTTP request
        |
     Postman
        |
        +-- displays the REST API's JSON response
```

The Python job fetches and stores articles. Separately, Postman requests a REST
endpoint; the API reads the stored data from MongoDB and returns it. Postman
shows that response, which helps verify the API.
