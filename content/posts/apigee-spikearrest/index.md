+++
date = '2026-08-14'
draft = false
title = 'Apigee Spike Arrest: How the Bucket Actually Works'
description = 'A visual explanation of Apigee SpikeArrest, 429 responses, rate normalization, and the token bucket that determines burst capacity.'
summary = 'What a 429 means, why equivalent throughput rates can behave differently, and how clients should respond when Apigee SpikeArrest rejects a request.'
tags = ['apigee', 'api-management', 'traffic-management']
categories = ['Architecture']
+++

*A visual guide to 429s, rate normalization, and the token bucket behind Apigee SpikeArrest.*

Apigee SpikeArrest is easy to describe as a request limit, but that shorthand hides the detail that matters when traffic arrives in bursts. A policy can allow the same average throughput while producing very different results depending on whether its rate is expressed per second, per minute, or per hour.

This interactive deck walks through what a `429 Too Many Requests` response means, how clients should retry with backoff and jitter, and how Apigee derives burst capacity from the configured rate. The central idea is simple: there is no separate `MaxBurst` setting to tune. The bucket is shaped by the rate and its unit, so understanding those units is the key to predicting the behavior.

{{< deck src="/decks/spike-arrest.html" title="Apigee Spike Arrest: how the bucket actually works" >}}
