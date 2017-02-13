---
layout: post
title: "Testing Dropwizard UI Apps in Memory"
date: 2017-02-13T15:28:08+11:00
---

If you're in [javaland](https://darrenhobbs.com/2006/04/22/a-bad-citizen-in-javaland/), you can do a lot worse than [Dropwizard](http://www.dropwizard.io/1.0.6/docs/).

A simple, lightweight web service framework built on top of Jersey with an all-star cast of mature java libraries in supporting roles.

__In-process testing__ is *the* essential technique for fast feedback when writing software. It involves having a test process (e.g. Junit) start your application and hosting it in-memory, firing requests at the app directly from an in-memory test client. 

No packaging of the application.
No deployment.
No network IO for requests.
Direct access to the application to mock external dependencies.
Mock behaviour defined in code, sitting with your tests.

You can have a comprehensive, large-scale test suite run in *seconds*, which when multiplying that by the team size and number of pushes a day, is a backbone of time-saving techniques that power high-performing teams.

Dropwizard already has out-of-the-box support for in-process testing, but:

- testing UIs and user journeys is hard (sequential actions, form posts, persistent cookies, dealing with html, does the javascript work etc)
- no support for dependency switching

Here's a tutorial for how to set up a DI-switchable integration test harness with Dropwizard and Guice, and as an added bonus, how to set that up for UI apps so you can use webdriver in-memory too.

Set-up:

`

	compile "com.hubspot.dropwizard:dropwizard-guice:1.0.0.3",
            "io.dropwizard:dropwizard-core:1.0.6",
            "io.dropwizard:dropwizard-views:1.0.6",
            "io.dropwizard:dropwizard-views-freemarker:1.0.6",
            "io.dropwizard:dropwizard-client:1.0.6",
            "io.dropwizard:dropwizard-assets:1.0.6",
            "io.dropwizard:dropwizard-metrics-graphite:1.0.6",
            "com.palominolabs.metrics:metrics-guice:3.1.3"
			
    testCompile "junit:junit-dep:4.11",
                "org.mockito:mockito-core:1.9.5",
                "io.dropwizard:dropwizard-testing:1.0.6",
                "com.thoughtworks.inproctester:jerseytester-webdriver:1.1.0",
                "com.thoughtworks.inproctester:jerseytester-htmlunit:1.1.0"
`

1. have tests use custom rule builder

``` java

package com.testupstream.app.integration;

import com.testupstream.app.integration.steps.IntegrationTests;
import org.junit.Test;
import org.openqa.selenium.By;

import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertThat;


public class ExampleIntegrationTest extends IntegrationTests {

    @Test
    public void homepageShouldHaveGoodbyeWorld() throws Exception {
        driver.get("http://localhost:9998/");
        assertThat(driver.findElement(By.tagName("h1")).getText(), is("Goodbye, World!"));
    }

}

```


Enter [jerseytester](https://github.com/aharin/jerseytester). This library provides a webdriver interface on top of JerseyTest, jersey's own in process test harness that dropwizard builds its test tools upon.
