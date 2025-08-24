---
layout: post
title: Method Timing in Java 25 with JFR and OpenTelemetry
tags: [java, profiling, jfr, open-telementry]
---
Recently, I [asked whether the Elastic APM agent could support Micrometer’s `@Timed` annotation](https://github.com/elastic/apm-agent-java/issues/4180) for method-level timings. That feature would let me avoid using AspectJ solely for this purpose. Of course, I could add manual instrumentation, but I’d rather not pollute my business logic with observability concerns.

[JEP 520](https://openjdk.org/jeps/520), arriving with Java 25, introduces `jdk.MethodTiming` and `jdk.MethodTrace` events in the Java Flight Recorder (JFR). Earlier, [JEP 349](https://openjdk.org/jeps/349), included in Java 14, added JFR Event Streaming. I’ve experimented with JFR Event Streaming before, but if you haven’t, I recommend reading [Inside Java’s introduction post](https://inside.java/2022/10/31/sip070/) for a solid overview.

These features can be combined to export timing metrics via OpenTelemetry. Initially, I experimented with `MethodTiming`, which provides aggregated metrics out of the box. My goal was to record average execution time per invocation using a simple `for` loop. However, both invocation counts and averages are cumulative. While it’s possible to calculate deltas for counts, deriving deltas from cumulative averages isn’t feasible.

That led me to switch to `MethodTrace`. This event type, especially when used with stack traces disabled, turns out to be even more powerful: each method call can be directly recorded in an OpenTelemetry histogram. This provides richer, invocation-level metrics instead of relying solely on pre-aggregated averages.

You’ll find a complete working example below. JDK 25 is [scheduled for release on 2025-09-16](https://openjdk.org/projects/jdk/25/).

```java
package com.kelunik;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.LongHistogram;
import jdk.jfr.consumer.RecordedMethod;
import jdk.jfr.consumer.RecordingStream;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.time.Duration;
import java.util.Map;

public class MethodTrace {
    public static void main(String[] args) {
        new Thread(() -> {
            while (true) {
                doSomeWork();
            }
        }).start();

        var settings = Map.of( //
                "jdk.MethodTrace#enabled", "true", //
                "jdk.MethodTrace#stackTrace", "false", //
                "jdk.MethodTrace#filter", "@" + Timed.class.getName());

        LongHistogram methodTiming = GlobalOpenTelemetry.getMeter("jdk.MethodTiming") //
                .histogramBuilder("method_timing")  //
                .setUnit("ns") //
                .ofLongs() //
                .build();

        try (RecordingStream rs = new RecordingStream()) {
            rs.onEvent("jdk.MethodTrace", event -> {
                Duration duration = event.getDuration("duration");
                RecordedMethod method = event.getValue("method");

                String methodName = method.getName();
                String methodType = method.getType().getName();

                Attributes attributes = Attributes.builder() //
                        .put("method", methodType + "." + methodName) //
                        .put("method_type", methodType) //
                        .put("method_name", methodName) //
                        .build();

                methodTiming.record(duration.toNanos(), attributes);

                System.out.println(methodType + "." + methodName + " called (" + duration + ")");
            });

            rs.setSettings(settings);
            rs.enable("jdk.MethodTrace");
            rs.startAsync();

            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }

            System.out.println("Stopped");
        }
    }

    @Timed
    private static void doSomeWork() {
        System.out.print(".");

        try {
            Thread.sleep(300);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    // Don't forget the runtime retention, thanks ChatGPT!
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Timed {}
}
```