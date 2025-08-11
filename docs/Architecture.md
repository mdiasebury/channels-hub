# Architecture of Eburyonline <!-- omit in toc -->

1. [Overview](#overview)
2. [Server](#server)
   1. [MVC](#mvc)
   2. [API consistency](#api-consistency)
   3. [Error handling](#error-handling)
   4. [Endpoints](#endpoints)
3. [Browser](#browser)
4. [Backends (internal systems)](#backends-internal-systems)
5. [Infrastructure](#infrastructure)
6. [CloudFront and S3](#cloudfront-and-s3)
7. [Web Application Firewall (WAF)](#web-application-firewall-waf)
8. [3rd party](#3rd-party)
9. [Monitoring and metrics](#monitoring-and-metrics)

## Overview

![Architecture diagram](./images/architecture.png)

## Server

The server runs [Django 4.2](https://www.djangoproject.com/) using an MVC design pattern. EBO has its own Postgres DB and Redis for caching data and storing user sessions.

The legacy code uses Django templates to generate HTML code, while the modern stack uses Django only as a REST API and the HTML is part of the SPA client written using [Vue.js](https://vuejs.org/guide/introduction.html).

The API is stateful for legacy and the modern stack, which means there is a user session stored on the server side. The development team treats this session only as a user-level cache for the important stuff. The session can be lost at any time and everything must be recoverable from external backend services or a database. Thus no load balancing based on cookies or any session locking is needed when serving parallel requests from one user. If a state is required (e.g. for persisting wizard forms), the state must be either held on the client side (using SPA) or persisted in a temporary table in the DB.

### MVC

Every API endpoint should follow the MVC pattern.

A view should extract the data from the request, then ask the controller to get or update the given data, and then the view should produce an HTTP response based on the requested format. It could be JSON, XML, PDF, CSV, HTML, etc.

Meanwhile, the controller should be responsible for validating the inputs coming from the requests and then retrieving the data from multiple backends or updating data in those backends.

Backends (e.g. `backends/bos/` or `backends/salesforce`) are just simple, context free, service clients in order to call endpoints of the backend. There should be no logic in these clients, they should not have any dependencies so everyone can call them from any place without creating circular imports. If any logic is required, the controller is the right place to do so.

### API consistency

The API should not format data like dates, addresses, or amounts, or translate texts. Data should always be returned raw and let the frontend format them as they want. This offers flexibility for our designers to adjust texts in the UI without modifying/deploying the server side. The only acceptable format for dates is the ISO 8601 standard. I18n and l10n have more detailed documentation available: [Dates and Time Zones](./DATES_AND_TIME_ZONES.md) and [Translations](./TRANSLATIONS.md).

The UI must not be aware of what service is called under the hood and maintain a consistent API schema and error handling. Examples of inconsistencies from the past that must be avoided:

- The service A communicates errors via HTTP status codes (400, 404) and detailed error description in the JSON object, meanwhile the service B returns 200 all the time and the error status code is in the JSON body.
- The service A returns all boolean values using `True`/`False` values, the service B uses numerical `0`/`1`
- The service A returns dates in ISO format (`2022-01-01`), the service B made up its own (`01012022`)
- The service A returns numbers in plain format (`1234.56`), the service B preformattes them (`"1,234.56"`)

The discrepancies must be eliminated on the service layer (or in service client if agreed) but never in UI.

### Error handling

To keep the code easy to understand, it should only contain a happy path. If an error is encountered, the code should always raise an exception and let the global error handlers to handle the exception. That means no silly error propagation throughout the code using error codes, `True/False` or `None` to indicate a call ended with an error. This adds lots of unnecessary logic into the code and makes the code cluttered with things that are not important. Raising the exception is the preferred way. The code should only catch these exceptions if there is a way to recover from it.

### Endpoints

To define the endpoints contract we use [OpenAPI](OPENAPI.md).

Every API endpoint should use an URL that points to a resource (URL stands for Uniform **Resource** Locator). Thus all URLs must use only nouns, not the verbs (e.g. `POST /api/trade/update/` should be `PUT /api/trades/`; or `POST /api/payment/{id}/authorise/` should be `POST /api/payments/{id}/authorisations/`).

In order to identify issues with the server and raise alerts, the endpoints must return correct [status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status). Using only `200` with the status code buried in the JSON body is strictly prohibited. Some HTTP status codes (e.g. `400`, `422`) may return additional error code in the body that helps the UI to better describe the cause of the error and display a proper message.

Every `5XX` error is automatically reported to Sentry and an alert is risen to the ONL team, so the endpoint should never raise those errors intentionally. `4XX` status codes must be used to indicate errors that are expected and user can recover from them.

To improve performance, endpoints should cache the output if it's possible using `Cache-Control` headers. The endpoint can cache the response globally for all users, or per user, or per language (using additional `Vary: Cookie` or `Vary: Accept-Language` header).

## Browser

Legacy code is generated on the server side using Django's templating engine and then the UX is enhanced using jQuery 3 and Backbone with Marionette views. The legacy code is deprecated and only bugs should be fixed there.

Then there is a Vue.js SPA code which also has a legacy part and the modern part (they're located in `frontend_legacy` (Vue 2) and `frontend` (Vue 3) folders). All new features in EBO must be developed in the modern SPA, if there is a need for new features in the legacy part, it should get migrated first to the modern stack.

The modern stack uses a [components architecture](https://vuejs.org/guide/essentials/component-basics.html) with strict separation of concerns and presentation logic. Every component must be testable and tested in unit tests and/or in integration tests. Each component style must follow BEM methodology, while the application styles follow the [ITCSS](https://www.xfive.co/blog/itcss-scalable-maintainable-css-architecture/) architecture on top of that. We built our own components library that adheres to these rules, you can find the [modern version](https://github.com/Ebury/chameleon) and the [legacy version](https://github.com/Ebury/ebury-chameleon) in GitHub.

The SPA uses a [state management](https://pinia.vuejs.org/) pattern to store the state of components on the client side (just like [Django's session](https://docs.djangoproject.com/en/4.2/topics/http/sessions/) does the same on the server side). The app, in order to be SPA, also includes [routing](https://router.vuejs.org/) (just like [Django's URLs](https://docs.djangoproject.com/en/4.2/topics/http/urls/)).

CSS evolved over time from using [LESS](https://lesscss.org/) to [SASS](https://sass-lang.com/) to [PostCSS](https://postcss.org/) with [CSS variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties), you can find all of them still being used in the project. The modern stack must only use PostCSS with CSS variables in order to scale our branding solution. In addition, we use [Tailwind CSS](https://tailwindcss.com/) for better DXP and very needed consistency of our CSS styles.

Everything is bundled together using [webpack 5](https://webpack.js.org/concepts/) (wrapped by [Vue CLI 5](https://cli.vuejs.org/)) using modern principles like [three shaking](https://v4.webpack.js.org/guides/tree-shaking/) and [dynamic imports](https://v2.vuejs.org/v2/guide/components-dynamic-async.html?redirect=true#Async-Components) to minimise the size of the final bundle. The legacy Vue code uses [webpack 4](https://v4.webpack.js.org/concepts/) (wrapped by [Vue CLI 4](https://cli.vuejs.org/)); and the legacy core, written before the webpack era, uses [Gulp 4](https://gulpjs.com/).

The weakest point of the legacy code is lack of testing. The spaghetti code is not testable at all and the development requires lots of manual tests to be performed before going live. Some scenarios are covered by the [E2E tests](https://github.com/Ebury/cell-eburyautomation) but mainly critical happy paths in the legacy.

We set a high priority for testing everything properly in the modern stack. The components architecture allows us to test every component individually in unit tests using [Jest](https://jestjs.io/) and then test all components together in integration tests ([Cypress](https://www.cypress.io/)) covering all edge cases. We test the code to have a full confidence when shipping the code and making changes in the code.

For more detailed documentation about our modern UI stack check out the [Frontend guide](./FRONTEND.md).

## Backends (internal systems)

EBO doesn't own any domain data, but rather retrieves data from backends behind the EBO (e.g. Ebury's `natonly`, `backoffice` clusters or from Salesforce servers). Here are the list of all direct backends and domains they cover for EBO views:

- [BOS](https://github.com/Ebury/bos): payments, deals, trades, funds, clients, contacts, beneficiaries, fees, statements
- [FXS](https://github.com/Ebury/fxsuite): rates, currencies
- [Salesforce](https://github.com/Ebury/sfdx): trade finance, suppliers
- [Verify](https://github.com/Ebury/verify): 2FA (login and payments)
- [Ebury API](https://github.com/Ebury/ebury-api-webapp): account details
- [Beneficiary Query Service](https://github.com/Ebury/ebury-services/tree/main/services/beneficiary-query-service): beneficiaries
- [Payment Query Service](https://github.com/Ebury/payment-query-service): payments delivery
- [Document Service](https://github.com/Ebury/documents-service/): document uploads
- [token.io Service](https://github.com/Ebury/ebury-token-io-service): payment initiation
- [BEXS Gateway](https://github.com/Ebury/bexs-gateway): payments, trades (Brazil)
- [The BAVS](https://github.com/Ebury/the-bavs): bank accounts validations

The list above only covers direct dependencies of the EBO, there may be more indirect services that are involved when processing requests from EBO, e.g.

- [Account Details Service](https://github.com/Ebury/account-details-service) (accessed via Ebury API)
- [Smart Date Service](https://github.com/Ebury/smart-date) (accessed via BOS)
- [Fee Tier Service](https://github.com/Ebury/fee-tier-service) (accessed via BOS)

## Infrastructure

EBO runs as a ReplicaSet in [K8s](https://kubernetes.io/), inside of the `backoffice` cluster. The deployment uses 2 pods, the number of them is auto scaled (e.g. 3 to 9 pods in prod, 2 to 6 in staging). A full configuration of EBO can be found in [ebury-manifests](https://github.com/Ebury/ebury-manifests/tree/master/k8s-environments) repo, in the `online` folders.

On top of that, there's a full set of AWS components spawned inside `publicfrontal` cluster that opens EBO to public and protects EBO from public, e.g Route 53, ALB, S3 buckets, WAF and Cloudfront CDN. The database and redis are also deployed inside this cluster and EBO connects to them from the `backoffice` cluster. A full configuration of EBO in ECS can be found in [publicfrontal cluster](https://github.com/Ebury/terraform-publicfrontal/blob/develop/src/app_ebo.tf) repo.

## CloudFront and S3

When deployed to production in AWS, EBO has a caching layer in front of its servers - [CloudFront](https://aws.amazon.com/cloudfront/), which serves as a CDN to distribute static content to our clients across Europe, Canada and Australia. The content, generated by webpack/gulp, is built during the release and then stored in S3 which is used by the CF as a content origin. The static content in S3 contains JS, CSS, image files built by the legacy core and Vue.js apps. On top of that, CloudFront caches every response from EBO's REST API if the [Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) header is present in the HTTP response.

The CloudFront is managed automatically via Terraform and the configuration for EBO can be found in the [publicfrontal cluster](https://github.com/Ebury/terraform-publicfrontal/blob/develop/src/locals.tf#L277) repo.

## Web Application Firewall (WAF)

Similarly to CloudFront, [WAF](https://aws.amazon.com/waf/) is yet another layer in front of CloudFront and EBO. It protects the entire stack from automated attacks, restricting the access based on the rate limiter and blocks malicious requests using predefined set of rules. If a request doesn't pass the checks, a 403 response is returned to the client and the request is not forwarded to the CF.

The full configuration of the firewall can be found in [publicfrontal cluster](https://github.com/Ebury/terraform-publicfrontal/blob/develop/src/locals.tf#L295) repo.

All blocked or allowed requests are in the logs for WAF in [Kibana](https://kibana6.ebury.rocks/_plugin/kibana/app/kibana#/discover?_g=(refreshInterval:(pause:!f,value:60000),time:(from:now-24h,mode:quick,to:now))&_a=(columns:!(action,httpRequest.clientIp,httpRequest.httpMethod,httpRequest.uri),index:'3923d2c0-cac1-11eb-943d-8f4c68701928',interval:auto,query:(language:lucene,query:''),sort:!(timestamp,desc),viewMode:view)).

## 3rd party

EBO relies on a few 3rd party services outside of Ebury:

- [Sentry](https://sentry.io/organizations/eburytech/issues/?project=1510303): To log and alert issues from client and server side.
- [Google Analytics](https://analytics.google.com/analytics/web/#/report-home/a15157589w80206421p86763122): To collect events for our UX Team
- [Segment](https://app.segment.com/ebo/overview): To collect events for our Data Team
- [Intercom](https://www.intercom.com/): To provide support for the clients
- [Inspectlet](https://www.inspectlet.com/): To record sessions needed for debugging client issues
- [Scanii](https://scanii.com/): To scan uploaded files

All 3rd party services are configured the way that EBO must continue working if they are down. We can remove any of these services from EBO just by flipping [switches/flags](https://waffle.readthedocs.io/en/stable/types/index.html) in our DB.

## Monitoring and metrics

Check out the [Monitoring](./MONITORING.md) and [Metrics](./METRICS.md) guides.
