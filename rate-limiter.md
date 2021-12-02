---
layout: default
title: Rate Limiter
---

# Rate Limiter
{: .no_toc }

1. TOC
{:toc}

[Rate limiters][RateLimiter] allow you to control the rate of executions as a way of preventing system overload. Failsafe provides two types of rate limiting: *smooth* and *bursty*, which are discussed below.

## How It Works

When the number of executions through the rate limiter exceeds the configured rate at the current point in time, further executions will either fail with `RateLimitExceededException` or will block and wait until permitted.

## Smooth Rate Limiting

*Smooth* rate limiting evenly spreads out execution requests over-time, when possible, effectively smoothing out bursty executions. Creating a smooth [RateLimiter] is straightforward:

```java
// Permits an execution every 10 milliseconds
RateLimiter<Object> limiter = RateLimiter.builder(Duration.ofMillis(10)).build();
```

Executions are permitted with no delay up to the given `executionRate`.

## Bursty Rate Limiting

*Bursty* rate limiting allows potential bursts of executions to occur, up to a configured max per time period. Creating a bursty [RateLimiter] is also straightforward:

```java
// Permits 10 executions every 1 second
RateLimiter<Object> limiter = RateLimiter.builder(10, Duration.ofSeconds(1)).build();
```

Executions are tracked per the configured time period, and are permitted with no delay up to the given `maxExecutions` for the current period.

## Waiting

By default, when a [RateLimiter] is exceeded, further executions will immediately fail with `RateLimitExceededException`. A rate limiter can also be configured to block and wait for execution permission if it can be achieved within a `maxWaitTime`:

```java
builder.withMaxWaitTime(Duration.ofSeconds(1));
```

Actual wait times for a rate limiter can vary depending on busy the rate limiter is. Wait times will grow if more executions are consistently attempted than the rate limiter permits. Since execution threads block while waiting for a rate limiter, a `maxWaitTime` should be reasonable enough to avoid blocking too many threads.

## Rate Limiters with Retries

As an alternative to configuring a `maxWaitTime`, which may block an execution while waiting for permission, you can also wrap a [RateLimiter] with a [RetryPolicy] and perform an async execution:

```java
RateLimiter<Object> limiter = RateLimiter.builder(Duration.ofMillis(10)).build();
RetryPolicy<Object> retryPolicy = RetryPolicy.builder()
  .handle(RateLimitExceededException.class)
  .withDelay(Duration.ofSeconds(1))
  .build();
  
Failsafe.with(retryPolicy, limiter).runAsync(this::sendMessage);
```

If the execution attempt exceeds the rate limit, it will be retried asynchronously without blocking a thread while waiting. This approach does require that a rate limiter permit is free at the time of the execution though, since it does not wait for permission.

## Event Listeners

[RateLimiter] supports that standard [policy listeners][policy-listeners] that indicate when an execution attempt through the rate limiter succeeds or fails.

## Best Practices

A rate limiter can and *should* be shared across code that accesses common dependencies. This ensures that if the rate limit is exceeded, all executions that share the same dependency and use the same rate limiter will either wait or fail until executions are permitted again. For example, if multiple connections or requests are made to the same external server, typically they should all go through the same rate limiter.

## Standalone Usage

A [RateLimiter] can also be manually operated in a standalone way:

```java
if (rateLimiter.tryAcquirePermit()) {
  doSomething();
}
```

## Performance

Failsafe's internal [RateLimiter] implementation is efficient, with _O(1)_ time and space complexity.

{% include common-links.html %}