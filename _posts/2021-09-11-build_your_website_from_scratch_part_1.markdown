---
layout: post
title: "Build your own Website from Scratch: Part 1 of 3"
date: 2021-09-11 19:43:24 -0500
---

## Motivation

As a Salesforce engineer, I have written plenty of Apex throughout my career, but I am a complete novice when it comes to other technologies. I rarely get the opportunity to make anything that is not on the core Salesforce platform, so I thought it would be fun to learn some new engineering skills and build my own personal website and blog from scratch.

I used four technologies to build this site:

- Lightning Web Components
- Node.js
- PostgreSQL
- Heroku

There is a lot to cover regarding this topic, so this blog post will be split out into three parts:

- Initial App Setup
- Database and API
- Deploying to Production
- Today, we will discuss some of the initial web application setup, the overview of the components, and single page application routing.

## Getting Started

### Install Node.js

Get started by installing node.js on your local machine by following the directions located here.

### Create LWC App

Once node is installed, we are going to need to create our initial LWC application. Execute the following command:

```shell
npx create-lwc-app my-blog
```

Complete the steps described in the command prompt. This will generate the initial scaffolding for your Lightning Web Component application.

### Install Lightning Base Components

Install the lightning-base-components package into your project with the following command:

```shell
npm install lightning-base-components
```

After successfully installing the npm package add the following code to the modules in the `lwc.config.json` file.

```js
{ "npm": "lightning-base-components" }
```

This will allow you to use some of the lightning web components that are familiar to us Salesforce developers such as `lightning-card` and `lightning-icon`.

### Add SLDS

Salesforce Lightning Design System, or [SLDS](https://www.lightningdesignsystem.com/), is a CSS framework that can allow you to use the style of Salesforce in any web application of your choice. To install and use SLDS, install this package from the npm registry:

```shell
npm install @salesforce-ux/design-system
```

Once the npm package is installed, we need to add the below markup to the `src/client/index.html` file:

```html
<link
  rel="stylesheet"
  href="/resources/assets/styles/salesforce-lightning-design-system.min.css"
/>
```

Finally, we need to explicitly override styles with Synthetic Shadow by adding import `'@lwc/synthetic-shadow';` to `src/client/index.js`.

### Component Layout

There are seven components which currently define the structure of this whole website.

#### App

App is the container for the application which all other components are rendered in.

#### Header

Header is the top header bar which contains the website title and the avatar icon.

#### About

About holds the content displayed on the default page of the website.
![](/images/websiteFromScratch/about.png)

#### Certifications

The Certifications component renders a list of all of my active certifications.
![](/images/websiteFromScratch/certifications.png)

#### Blog

This component renders a collection of clickable tiles for all blog posts.
![](/images/websiteFromScratch/blog.png)

#### Blog Post

Blog Post stores the markup for an individual blog post.
![](/images/websiteFromScratch/blogpost.png)

### Single Page Application Routing

Having all of these components is nice, but making them work within a single page application took a little bit of work. We can use [@lwce/router](https://github.com/LWC-Essentials/route) to make sure that URL changes and direct routes to specific parts of the application work. To use this in your project, install the following package off the npm registry:

```shell
npm install --save @lwce/router
```

Then, add `{"npm": "@lwce/router"}` to your lwc.config.json file.

With the router installed in the project, we need to make some modifications to the app component. We will modify this component to use some custom LWCs from the `@lwce/router` package:

- `<lwce-router>` this tag will define the total "route-able" space of the application
- `<lwce-route path="/some_path">` this tag will define a route who's content will be rendered when the URL is active.
- `<lwce-link path="/some_path">` this tag will define a link to route to the corresponding route.

With these components, we can use SLDS to define a markup within the app component that renders just like the `<lightning-tabset>` component, but supports all of our single page application routing.

![](/images/websiteFromScratch/certification-route.png)

Now we have a working Lightning Web Component application which supports base lightning components, SLDS, and single page application routing. All of this is great, but a few things are missing before we can actually use this application.

## Stay Tuned!

In the next part of this blog post - we will discuss setting up the database which will store all blog posts, the API for interacting with those posts, and how to wire the data returned by the API to the user interface. In the final post, we will share how to launch your application to production using Heroku.
