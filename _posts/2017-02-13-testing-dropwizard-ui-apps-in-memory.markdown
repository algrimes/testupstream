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

	{% highlight groovy %}	
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
	{% endhighlight %}


1. In your test code, override your guice bindings in the areas that you want to mock (e.g. time, external dependencies). Use your production guice module as the base (e.g. AppModule)

    {% highlight java %}	
    Injector injector = Guice.createInjector(Stage.PRODUCTION, Modules.override(new AppModule()).with(binder -> {
        binder.bind(ResponseProvider.class).toInstance(mock(ResponseProvider.class));
        binder.bind(DateTimeProvider.class).to(TestDateTime.class);
    }));
    {% endhighlight %}


{:start="2"}
2. Expose a list of resources in your DW appication class, and use ResourceTestRule instead of DropwizardAppRule. Add resource classes from the injector (my app is called App here)

    {% highlight java %}
    ResourceTestRule.Builder builder = new ResourceTestRule.Builder();
    
    for (Class resource : new App().getResources()) {
        builder.addResource(injector.getInstance(resource));
    }
    
    JerseyGuiceUtils.reset(); //Because of https://github.com/HubSpot/dropwizard-guice/issues/95
    
    ResourceTestRule rule = builder.build();
    {% endhighlight %}
	
{:start="3"}
3. Use a Junit ClassRule in your test class (or superclass) to point to your built rule (i happened to wrap that up in a harness)

    {% highlight java %}
    @ClassRule
    public static final ResourceTestRule rule = HARNESS.getRule();
    {% endhighlight %}
	
Enter [jerseytester](https://github.com/aharin/jerseytester)	

This library provides a webdriver interface on top of a Jersey client, adapting requests and responses so you get all the good stuff from HTMLUnitDriver - persistent browser state, cookies, and the page being traversable by that sweet API.
	
{:start="4"}
4. ResourceTestRule exposes a client, use this to bootstrap the webdriver

    {% highlight java %}
    driver = new JerseyClientHtmlunitDriver(rule.client());
    {% endhighlight %}
    
*	Note: calling client() before the test run begins will throw an NPE*

{:start="5"}
5. Profit! 

	Here is an example of using mockito at runtime to manipulate the response of the mock. *(Baseuri for ResourceTestRule is "http://localhost:9998")*

	{% highlight java %}
	
	public void theHomepageIsVisited() {
	    driver.get(baseUri);
	}
	
	public String homepageText() {
	    return driver.findElement(By.tagName("body")).getText();
	}
	
	public void theHomePageResponseIs(String message) {
	    when(injector.getInstance(ResponseProvider.class).get()).thenReturn(message);
	
	}
	{% endhighlight %}

For context, the production code that uses the ResponseProvider is:

	{% highlight java %}
	@Path("/")
	@Produces(MediaType.TEXT_HTML)
	public class HelloWorldResource {

	    private ResponseProvider responseProvider;

	    @Inject
	    public HelloWorldResource(ResponseProvider responseProvider) {
	        this.responseProvider = responseProvider;
	    }

	    @GET
	    public Response getHelloWorld(){
	        return Response.ok(responseProvider.get()).build();
	    }

	}
	}
	{% endhighlight %}

	{% highlight java %}
	private void validateInProcessMocking(String message) {
	    given.theHomePageResponseIs(message);
	    when.theHomepageIsVisited();
	    assertThat(then.homepageText(), is(message));
	}
	{% endhighlight %}

#### Next time....

I'll post about how to get webdriver working with your javascript



