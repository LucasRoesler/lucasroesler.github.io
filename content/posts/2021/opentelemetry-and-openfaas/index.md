---
title: "A foray into OpenTelemetry with OpenFaaS"
description: "Introducing a walk-through for adding OpenTelemetry to your Python functions"
date: 2021-12-12T00:00:00+02:00
tags:
  - programming
  - openfaas
  - serverless
  - python
  - tracing
  - opentelemetry
  - observability
  - kubernetes
draft: false
# Photo by <a href="https://unsplash.com/@artnok?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Nicolas Picard</a> on <a href="https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
coverImage: images/nicolas-picard-small-unspash.jpg
---

I have always been a fan of tracing. My first taste of it was with NewRelic, but the development of [OpenTracing](https://opentracing.io/) and more recently [OpenTelemetry](https://opentelemetry.io/) have made it an easy must have in every project I start. I have created a [new walk-through: Tracing and Observability with OpenFaaS](https://github.com/LucasRoesler/openfaas-tracing-walkthrough) to show you how to add OpenTelemetry to your Python Flask functions.

---

This post won't go through the walk-through, the walk-through already does that. Instead, I will offer some commentary on the experience and some next steps we should take.

## Set up and configuring OpenTelemetry

OpenTelemetry provides a set of vendor agnostic APIs, SDKs, and tooling to provide telemetry data for you applications. This means: tracing, logging, and metrics. This is an ambitious project to standardize the way we monitor and debug our applications in almost every popular language. This is great but has has certainly made the initial configuration slightly more complex. The minimal configuration of the Flask app looks something like this

```py
resource = Resource(attributes={SERVICE_NAME: NAME})
provider = TracerProvider(resource=resource)

exporter = OTLPSpanExporter()
provider.add_span_processor(BatchSpanProcessor(exporter))

trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

FlaskInstrumentor().instrument_app(app)
```

Previously, I would have used Jaeger and OpenTracing and it would look something like this

```py
config = Config(config={}, service_name=NAME, validate=True)
tracer =  config.initialize_tracer()
tracing = FlaskTracing(tracer, True, app)
```

It is slightly simpler, because OpenTracing, is slightly less agnostic putting more work onto the tracing provider. This means that the OpenTracing Jaeger provider does a little bit more work, whereas OpenTelemetry needs to expose more configuration options. OpenTelemetry introduces or exposes the concepts of the Resource, Exporter, and Provider that would previously be rolled together as just the Tracer.

My first impression was "oh no! what is all of this new stuff?" Fortunately, OpenTelemetry is very [12 Factor App](https://12factor.net/config) friendly, meaning most or all of the options can be configured via the environment variables. This also means that once you a familiar with what options you need to configure, you can probably hide this configuration behind a simple utility function. Of course, your milage may vary, these configuration options are exposed for a reason and may need to be tweaked based on your preferred language and observability stack.

## Backwards compatibility for OpenTracing

In Python, I was able to completely customize the configuration with the correct `pip install` and environment variables. Notably, the walk-through integrates traces from NGINX and an OpenFaaS function. Currently NGINX can produce _OpenTracing_ tracing headers with support for [Jaeger](https://www.jaegertracing.io/), [Zipkin](https://zipkin.io/), and Datadog. The default configuration in the Python OpenTelemetry SDK does not parse these headers, but it is very easy to enable it.

```sh
pip install opentelemetry-propagator-jaeger

```

and then set this environment variable

```yaml
OTEL_PROPAGATORS: tracecontext,baggage,jaeger
```

This works really well in Python because it can dynamically import the required `opentelemetry-propagator-jaeger`, so we don't need to modify our function to enable this compatibility.

## Integrating into OpenFaaS templates

The OpenTelemetry project, NGINX, and Grafana have made it very easy to install and integrate OpenTracing/OpenTelemetry with minimal changes to your code. If you are writing OpenFaaS functions, this flexibility will depend greatly on your template.

The default [Flask template](https://github.com/openfaas/python-flask-template) is a bit harder to integrate, primarily because it does not directly expose the request headers to your function implementation. We need access to the headers to initialize the tracing context. For my walk-through, I forked the original template to add an `initialize` hook so that the function can modify the Flask app initialization. In the template, I added

```py
# the template
if hasattr(handler, "initialize"):
    app = handler.initialize(app)
```

then in handler I can add

```py
def initialize(app: Flask) -> Flask:
    FlaskInstrumentor().instrument_app(app)
```

This actually seems like a good pattern that we should consider adopting in all of the base OpenFaaS templates. As it stands, we could probably standardize on OpenTelemetry in all templates because it is agnostic of your preferred vendor. Prior to OpenTelemetry, we would have had to pick a preferred SDK, it is much easier to abstract and integrate tracing into each template without adding this `initialize` hook. However, the pattern would be very useful in general, for example, to customize the logger or other Flask middlewares.

On the other hand, the base templates are exactly that, generic and basic. I generally advise teams adopting OpenFaaS to take advantage of the custom template repository support. This allows teams to standardize and pre-configure their preferred libraries, utils, and configuration. This is how I work, a repository of private custom templates using my preferred libraries configured in the "correct" way. Correct for your team might be slightly different, so fork the base templates and "fix" them to your liking.

## Next steps and tracing OpenFaaS components

The obvious next steps are to create more walk-throughs for other languages. Go is my preferred language, so that might happen before the end of the year. Check out the [template store repo](https://github.com/openfaas/templates/tree/master/template), for your favorite language. If you want to give OpenTelemetry a try, then [fork my walk-through](https://github.com/LucasRoesler/openfaas-tracing-walkthrough) and provide a new sample function for your favorite language/template. The walk-through already provides all of the setup and configuration for the cluster and tracing, we only need to build out more sample functions.

Part of the reason that I included NGINX in my walk-through is the (1) it is a very common component and (2) I wanted to have multiple components producing traces and showing those traces connecting together to provide a more complete picture of the system. Currently, OpenFaaS does not integrate any tracing. Of course, it does not prevent any tracing, it is very good about ensuring that headers are preserved both on the request and the response. This is true for both synchronous and asynchronous function calls.

One area that would be nice to add tracing is in the OpenFaaS Gateway. The Gateway process is pretty simple and light weight, but it would be nice to have an explicit tracing span showing the Gateway timing, similar to thr tracing that NGINX provides. It can obviously be inferred from the existing traces, but explicit is better than implicit, as the [Zen of Python](https://www.python.org/dev/peps/pep-0020/) says.

{{<image src="images/connected-traces.png" title="Connected Traces: we can estimate that the Gateway adds ~10-15ms of overheard." >}}

OpenTelemetry's flexibility makes this much easier add to OpenFaaS while also providing maximum flexibility. We started looking into awhile ago with OpenTracing, but never made much progress: https://github.com/openfaas/faas/issues/1354. If this is something you are interested in or what to see, let me know on the [Github issue](https://github.com/openfaas/faas/issues/1354) or [Twitter](https://twitter.com/Theaxer).
