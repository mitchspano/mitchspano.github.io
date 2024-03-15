---
layout: post
title: "Build your own Website from Scratch: Part 3 of 3"
date: 2021-10-31 19:43:24 -0500
---

# Deploying to Production

So far in this blog series, we have created our own node.js LWC application which includes a server that can interact with a PostgreSQL database to fetch the stored the blog posts. Running your LWC application and PostgreSQL server locally is challenging, but getting it running online at the domain of your choice adds another layer of complexity. Today, we will be discussing how to turn your project into a production application using Heroku.

## Heroku Setup

The first step is to create a new Heroku account and application. Your application name must be unique. After you have a registered account and a unique application name, navigate to your application. Within your application, navigate to the "Resources" and add "Heroku PostgreSQL". For this application, the storage needs are minimal, as we are just storing the markup of some blog posts, so the "Hobby Dev - Free" plan is more than sufficient.

Once the Heroku PostgreSQL database is configured, navigate to the "Settings" tab and click the "Reveal Config Vars" button. this will render the URL for the database - the configuration variable key will be `DATABASE_URL`.

![config vars image](/images/websiteFromScratch/configVars.png)

This URL is important, because it will tell our application which database to interact with when launched to production.

## Test Locally

Testing your LWC application locally when preparing to deploy to Heroku is a little different than testing a normal open-source LWC application. First, download and install the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli).

### Procfile

Once the CLI is installed, we will need to create a file called `Procfile`. This file defines the commands that are executed by the app when it starts executing on Heroku. The body of the `Procfile` should contain the following line:

```raw
web: npm run serve
```

### Environment Variables

The first environment variable that we need to manage is the `DATABASE_URL`. We can access environment variables in any node.js project by using `process.env.[variable name]`. In the `src/server/database.js` file, we reference this variable.

```js
const pool = new Pool({
  connectionString:
    process.env.DATABASE_URL || "postgres://localhost/mitchspano",
  ssl: process.env.DATABASE_URL ? { rejectUnauthorized: false } : false,
});
```

This approach works well, but it is suitable only for _server-side code_. In the `src/client/modules/data/dataService.js` file, we have a client-side module that must reference an `API_ENDPOINT` environment variable. The production value for this variable will be defined in a Heroku config variable, but we expect this endpoint to be different when testing locally.

```
const apiUrl = process.env.API_ENDPOINT || 'http://localhost:5000'; // eslint-disable-line"
```

To inject environment variables into our client-side code, we will use **Webpack**. Create a file in the root directory of your project called `webpack.config.js` with the following body.

```js
const webpack = require("webpack");
module.exports = {
  output: {
    publicPath: "/",
  },
  plugins: [new webpack.EnvironmentPlugin(["API_ENDPOINT"])],
};
```

Now that we have the Procfile defined, the CLI installed, and our
environment variables defined we can run Heroku locally and test our
application out.

![](/images/websiteFromScratch/runLocally.png)

## Domain Management

By default, Heroku apps will be available at `[appName].herokuapp.com`. The complete guide for custom domain management from Heroku is available [here](https://devcenter.heroku.com/articles/custom-domains), but we will go over some of the basics.

Purchase your domain from any of your favorite domain name providers. Some commonly used domain merchants include [GoDaddy](http://godaddy.com) and [NameCheap](http://NameCheap.com). Once you have obtained your URL, navigate to your application in Heroku, open the "Settings" tab, and scroll to the "Domains" section. Add your domain here.

![Domain settings image](/images/websiteFromScratch/domains.png)

Once the domain has been added, add the DNS settings (which are automatically generated on Heroku) to your domain.

### SSL

We might be able to use a free database, but if we want our application to support SSL, we will need to upgrade from the free dyno to the hobby dyno. Upgrading from free to hobby dyno costs approximately $7 USD per month - this is the only cost to the whole project.

![Dyno types image](/images/websiteFromScratch/dynoTypes.png)

## Deploy from Source

Now that we have a Heroku application with environment variable support and our own domain, we can _finally_ deploy from source to production. Navigate to your application, select your deployment method. In this example, we will be using GitHub.

![GitHub connection image](/images/websiteFromScratch/githubConnection.png)

Once your GitHub repository is linked, you can choose to deploy manually, or set up automatic deploys. With automatic deployment, Heroku will listen to changes on a specified branch of your repository and kick off the deployment process whenever a change is merged into the specified branch.

## That's all folks!

Congratulations! Now you have all of the knowledge necessary to build your own application from scratch using node.js as your server, PostgreSQL as your database, Lightning Web Components for your user interface, and Heroku for your infrastructure. I hope you enjoyed this three-part blog post. Learning new technologies can be challenging, but hopefully, this guide will help you in your journey to learn about these development tools so you can build amazing products and services.
