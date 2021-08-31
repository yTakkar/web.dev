---
title: Best practices for measuring Web Vitals in the field
subhead: Cómo medir Web Vitals con su herramienta de análisis actual.
authors:
  - philipwalton
description: Cómo medir Web Vitals con su herramienta de análisis actual
date: '2020-05-27'
updated: 21/07/2020
hero: image / admin / WNrgCVjmp8Gyc8EbZ9Jv.png
alt: Cómo medir Web Vitals con su herramienta de análisis actual
tags:
  - blog
  - performance
  - web-vitals
---

Tener la capacidad de medir e informar sobre el rendimiento real de sus páginas es fundamental para diagnosticar y mejorar el rendimiento a lo largo del tiempo. Sin [datos de campo](/user-centric-performance-metrics/#in-the-field) , es imposible saber con certeza si los cambios que está realizando en su sitio realmente están logrando los resultados deseados.

Many popular [Real User Monitoring (RUM)](https://en.wikipedia.org/wiki/Real_user_monitoring) analytics providers already support the [Core Web Vitals](/vitals/#core-web-vitals) metrics in their tools (as well as many [other Web Vitals](/vitals/#other-web-vitals)). If you're currently using one of these RUM analytics tools, you're in great shape to assess how well the pages on your site meet the [recommended Core Web Vitals thresholds](/vitals/#core-web-vitals) and prevent regressions in the future.

While we do recommend using an analytics tool that supports the Core Web Vitals metrics, if the analytics tool you're currently using does not support them, you don't necessarily need to switch. Almost all analytics tools offer a way to define and measure [custom metrics](https://support.google.com/analytics/answer/2709828) or [events](https://support.google.com/analytics/answer/1033068), which means you can likely use your current analytics provider to measure the Core Web Vitals metrics and add them to your existing analytics reports and dashboards.

This guide discusses best practices for measuring Core Web Vitals metrics (or any custom metrics) with a third-party or in-house analytics tool. It can also serve as a guide for analytics vendors wishing to add Core Web Vitals support to their service.

## Use custom metrics or events

As mentioned above, most analytics tools let you measure custom data. If your analytics tool supports this, you should be able to measure each of the Core Web Vitals metrics using this mechanism.

Measuring custom metrics or events in an analytics tool is generally a three-step process:

1. [Define or register](https://support.google.com/analytics/answer/2709829?hl=en&ref_topic=2709827) the custom metric in your tool's admin (if required). *(Note: not all analytics providers require custom metrics to be defined ahead of time.)*
2. Compute the value of the metric in your frontend JavaScript code.
3. Send the metric value to your analytics backend, ensuring the name or ID matches what was defined in step 1 *(again, if required)*.

For steps 1 and 3, you can refer to your analytics tool's documentation for instructions. For step 2 you can use the [web-vitals](https://github.com/GoogleChrome/web-vitals) JavaScript library to compute the value of each of the Core Web Vitals metrics.

El siguiente ejemplo de código muestra lo fácil que puede ser realizar un seguimiento de estas métricas en el código y enviarlas a un servicio de análisis.

```js
import {getCLS, getFID, getLCP} from 'web-vitals';

function sendToAnalytics({name, value, id}) {
  const body = JSON.stringify({name, value, id});
  // Use `navigator.sendBeacon()` if available, falling back to `fetch()`.
  (navigator.sendBeacon && navigator.sendBeacon('/analytics', body)) ||
      fetch('/analytics', {body, method: 'POST', keepalive: true});
}

getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getLCP(sendToAnalytics);
```

## Ensure you can report a distribution

Once you've computed the values for each of the Core Web Vitals metrics and sent them to your analytics service using a custom metric or event, the next step is to build a report or dashboard displaying the values that have been collected.

To ensure you're meeting the [recommended Core Web Vitals thresholds](/vitals/#core-web-vitals), you'll need your report to display the value of each metric at the 75th percentile.

If your analytics tool does not offer quantile reporting as a built-in feature, you can probably still get this data manually by generating a report that lists every metric value sorted in ascending order. Once this report is generated, the result that is 75% of the way through the full, sorted list of all values in that report will be the 75th percentile for that metric—and this will be the case no matter how you segment your data (by device type, connection type, country, etc.).

If your analytic tool does not give you metric-level reporting granularity by default, you can probably achieve the same result if your analytics tool supports [custom dimensions](https://support.google.com/analytics/answer/2709828). By setting a unique, custom dimension value for each individual metric instance you track, you should be able to generate a report, broken down by individual metric instances, if you include the custom dimension in the report configuration. Since each instance will have a unique dimension value, no grouping will occur.

The [Web Vitals Report](https://github.com/GoogleChromeLabs/web-vitals-report) is an example of this technique that uses Google Analytics. The code for the report is [open source](https://github.com/GoogleChromeLabs/web-vitals-report), so developers can reference it as an example of the techniques outlined in this section.

<img src="https://user-images.githubusercontent.com/326742/101584324-3f9a0900-3992-11eb-8f2d-182f302fb67b.png" alt="Capturas de pantalla de Web Vitals &lt;span translate =" no=""> Informe "data-md-type =" image "&gt;

{% Aside %} Tip: The [`web-vitals`](https://github.com/GoogleChrome/web-vitals) JavaScript library provides an ID for each metric instance reported making it possible to build distributions in most analytics tools. See the [`Metric`](https://github.com/GoogleChrome/web-vitals#metric) interface documentation for more details. {% endAside %}

## Envíe sus datos en el momento adecuado

Some performance metrics can be calculated once the page has finished loading, while others (like CLS) consider the entire lifespan of the page—and are only final once the page has started unloading.

This can be problematic, however, since both the `beforeunload` and `unload` events are not reliable (especially on mobile) and their use is [not recommended](https://developers.google.com/web/updates/2018/07/page-lifecycle-api#legacy-lifecycle-apis-to-avoid) (since they can prevent a page from being eligible for the [Back-Forward Cache](https://developers.google.com/web/updates/2018/07/page-lifecycle-api#page-navigation-cache)).

For metrics that track the entire lifespan of a page, it's best to send whatever the metric's current value is during the `visibilitychange` event, whenever the page's visibility state changes to `hidden`. This is because—once the page's visibility state changes to `hidden`—there's no guarantee that any script on that page will be able to run again. This is especially true on mobile operating systems where the browser app itself can be closed without any page callbacks being fired.

Note that mobile operating systems do generally fire the `visibilitychange` event when switching tabs, switching apps, or closing the browser app itself. They also fire the `visibilitychange` event when closing a tab or navigating to a new page. This makes the `visibilitychange` event far more reliable than the `unload` or `beforeunload` events.

{% Aside 'gotchas' %} Due to [some browser bugs](https://github.com/w3c/page-visibility/issues/59#issue-554880545), there are a few cases where the `visibilitychange` event does not fire. If you're building your own analytics library, it's important to be aware of these bugs. Note that the [web-vitals](https://github.com/GoogleChrome/web-vitals) JavaScript library does account for all of these bugs. {% endAside %}

## Supervisar el rendimiento a lo largo del tiempo

Once you've updated your analytics implementation to both track and report on the Core Web Vitals metrics, the next step is to track how changes to your site affect performance over time.

### Version your changes

A naive (and ultimately unreliable) approach to tracking changes is to deploy changes to production and then assume that all metrics received after the deployment date correspond to the new site and all metrics received before the deployment date correspond to the old site. However, any number of factors (including caching at the HTTP, service worker, or CDN layer) can prevent this from working.

Un enfoque mucho mejor es crear una versión única para cada cambio implementado y luego rastrear esa versión en su herramienta de análisis. La mayoría de las herramientas de análisis admiten la configuración de una versión. Si el suyo no lo hace, puede crear una dimensión personalizada y establecer esa dimensión en su versión implementada.

### Ejecutar experimentos

Puede llevar el control de versiones un paso más allá mediante el seguimiento de varias versiones (o experimentos) al mismo tiempo.

If your analytics tool lets you define experiment groups, use that feature. Otherwise, you can use custom dimensions to ensure each of your metric values can be associated with a particular experiment group in your reports.

With experimentation in place in your analytics, you can roll out an experimental change to a subset of your users and compare the performance of that change to the performance of users in the control group. Once you have confidence that a change does indeed improve performance, you can roll it out to all users.

{% Aside %} Experiment groups should always be set on the server. Avoid using any experimentation or A/B testing tool that runs on the client. These tools will typically block rendering until a user's experiment group is determined, which can be detrimental to your LCP times. {% endAside %}

## Ensure measurement doesn't affect performance

When measuring performance on real users, it's absolutely critical that any performance measurement code you're running does not negatively impact the performance of your page. If it does, then any conclusions you attempt to draw on how your performance affects your business will be unreliable, as you'll never know if the presence of the analytics code itself is having the largest negative impact.

Always follow these principles when deploying RUM analytics code on your production site:

### Defer your analytics

El código de análisis siempre debe cargarse de forma asincrónica y sin bloqueo y, por lo general, debe cargarse en último lugar. Si carga su código de análisis de forma bloqueada, puede afectar negativamente a LCP.

All of the APIs used to measure the Core Web Vitals metrics were specifically designed to support asynchronous and deferred script loading (via the [`buffered`](https://www.chromestatus.com/feature/5118272741572608) flag), so there's no need to rush to get your scripts loaded early.

In the event that you're measuring a metric that cannot be computed later in the page load timeline, you should inline *only* the code that needs to run early into the `<head>` of your document (so it's not a [render-blocking request](/render-blocking-resources/)) and defer the rest. Do not load all your analytics early just because a single metric requires it.

### Do not create long tasks

Analytics code often runs in response to user input, but if your analytics code is conducting a lot of DOM measurements or using other processor-intensive APIs the analytics code itself can cause poor input responsiveness. In addition, if the JavaScript file containing your analytics code is large, executing that file can block the main thread and negatively affect FID.

### Use non-blocking APIs

APIs like <code>&lt;a href="https://developer.mozilla.org/docs/Web/API/Navigator/sendBeacon" data-md-type="link"&gt;sendBeacon()&lt;/a&gt;</code> and <code>&lt;a href="https://developer.mozilla.org/docs/Web/API/Window/requestIdleCallback" data-md-type="link"&gt;requestIdleCallback()&lt;/a&gt;</code> are specifically designed for running non-critical tasks in a way that doesn't block user-critical tasks.

Estas API son excelentes herramientas para usar en una biblioteca de análisis de RUM.

In general, all analytics beacons should be sent using the `sendBeacon()` API (if available), and all passive analytics measurement code should be run during idle periods.

{% Aside %} For guidance on how to maximize the use of idle time, while still ensuring code can be run urgently when needed (like when a user is unloading the page), refer to the [idle-until-urgent](https://philipwalton.com/articles/idle-until-urgent/) pattern. {% endAside %}

### No rastree más de lo que necesita

The browser exposes a lot of performance data, but just because the data is available does not necessarily mean you should record it and send it to your analytics servers.

Por ejemplo, la [API de tiempo de recursos](https://w3c.github.io/resource-timing/) proporciona datos de tiempo detallados para cada recurso cargado en su página. Sin embargo, es poco probable que todos esos datos sean necesarios o útiles para mejorar el rendimiento de la carga de recursos.

In short, don't just track data because it's there, ensure the data will be used before consuming resources tracking it.