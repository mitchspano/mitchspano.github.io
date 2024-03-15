---
layout: post
title: "Build your own Website from Scratch: Part 2 of 3"
date: 2021-09-25 19:43:24 -0500
---

# Data and Services

In the [last blog post](https://www.mitchspano.com/blog/build_your_website_from_scratch_part_1), we covered some of the motivation behind the website [mitchspano.com](https://www.mitchspano.com), the initial setup for the node.js LWC app, and discussed some of the building blocks of the front end user interface. Today, we will switch gears and learn how the backend of the application is built to serve the content of this site. The interactions with the backend are very simple, but fitting all the pieces together can be a challenge, so let's dive in.

## Create the Database

We need to use a database to store the content of each of the blog posts for this site. I decided to go with [PostrgreSQL](https://www.postgresql.org/download/) as the persistent data storage to serve this website. To interact with the database, we will use the command line interface.

```
createdb myapp
psql -a myapp
```

If completed correctly, the terminal will prompt you for input for the `myapp` database. We can interact with the database from the terminal by entering sql statements directly to the prompt.

```

psql (13.4)
Type "help" for help.

myapp=# SELECT table_name FROM information_schema.tables WHERE table_schema='public';

table_name
------------
(0 rows)

myapp=#
```

To exit the psql terminal, press `Ctrl+d`.

## Define the Entities

Now that we have a provisioned database, let's go over the design. The database for this site is remarkably simple; it only has one table.

| blog_posts    |                   |                                               |
| ------------- | ----------------- | --------------------------------------------- |
| **id**        | serial            | Automatically generated Id for each blog post |
| **post_date** | date              | Date when the post was first created          |
| **title**     | character varying | Title of the blog post                        |
| **post_name** | character varying | The URL addressable name of the blog post     |
| **body**      | character varying | The raw HTML body of the blog post            |

To define this table, we need to execute a SQL script. Instead of writing our SQL directly from the terminal, we will write our commands to a `.sql` file, then pipe that file into our command. First, create a file called `createDatabase.sql` with the following contents:

```sql

CREATE TABLE IF NOT EXISTS blog_posts
(
    id        serial            NOT NULL,
    post_date date              NOT NULL,
    title     character varying NOT NULL,
    post_name character varying NOT NULL,
    body      character varying NOT NULL
);
```

Interacting with the database can be accomplished easily by using the command line interface. To execute a predefined SQL script, use the `-f` flag and pass in the path to the `.sql` file.

```
psql -a myapp -f createDatabase.sql
```

For this database, the rows contain raw HTML for the markup of each blog post. To make interacting with the database much easier, I have added some utility scripts which prevent me from needing to write raw SQL commands every time. For each blog post, I create a local `newpost.html` then I define the contents of that file as a variable to be referenced in a SQL script for inserts and updates by using the `\set` notation.

```html
<h1>Hello World!</h1>
<p>Thank you for visiting my blog.</p>
```

```sql
\set content `cat newpost.html`
INSERT INTO blog_posts (
    post_name,
    post_date,
    title,
    body
)
VALUES(
    'hello_world',
    '2021-09-26',
    'Hello World',
    :'content'
);
```

```sql
\set content `cat newpost.html`
UPDATE blog_posts
SET body = :'content'
WHERE post_name = 'hello_world';
```

Now we can define, insert, and update blog posts as rows in the database, but we need to allow the web application to transact with the database.

## Create a REST-ful API

The web application needs to perform two different interactions with the database. First, when a user navigates to [mitchspano.com/blog](https://www.mitchspano.com/blog), the system must query for all records in the `blog_posts` table ordered by their `post_date`.

![blog query](/images/websiteFromScratch/blogQuery.png)

Second, when a user navigates to a specific blog post, the system must query for all of the details for the blog post. Each blog post record has a URL addressable `post_name` that will be used to identify the post and create the SQL query.

![blog post query](/images/websiteFromScratch/blogPostQuery.png)

### Server

To facilitate these interactions, we must create two new files within the server: `database.js` to establish the initial connection pool and define the queries which can be executed on the Postgres database, and `router.js` to route traffic from the site to API endpoints which can handle the interactions.

```js
//database.js
const { Pool } = require('pg');

const pool = new Pool({
    connectionString:
        process.env.DATABASE_URL || 'postgres://localhost/myapp',
    ssl: process.env.DATABASE_URL ? { rejectUnauthorized: false } : false
});

const getPosts = async () =>
    pool.query(
        'SELECT id, post_date, title, post_name, body FROM blog_posts ORDER BY post_date DESC'
    );

const getPost = async (postName) =>
    pool.query(
        `SELECT id, post_date, title, post_name,  body FROM blog_posts WHERE post_name = $1`,
        [postName]
    );

module.exports = {
    getPosts,
    getPost
};"
```

```js
// router.js
const { Router } = require("express");
const router = new Router();
const db = require("./database");
const bodyParser = require("body-parser");

router.use(bodyParser.json());
router.use(bodyParser.urlencoded({ extended: true }));

router.get("/posts", (request, response) => {
  doQuery(response, db.getPosts);
});

router.get("/posts/:id", (request, response) => {
  doQuery(response, db.getPost, request.params.id);
});

async function doQuery(response, queryFunction, ...params) {
  try {
    const results = await queryFunction(...params);
    if (results) {
      response.status(200).json(results.rows);
    } else {
      response.status(200).json([]);
    }
  } catch (error) {
    console.error(error);
    response.status(500).json({ status: "error", message: `Error:${error}` });
  }
}

module.exports = router;
```

### Client

Now that these functions are defined, we must import these functions from our server to the web client so that our lighting web components will be able to call the functions. We accomplish this by first creating a new file in `src/client/modules/data` called `dataService.js`.

```js
const apiUrl = process.env.API_ENDPOINT || "http://localhost:5000"; // eslint-disable-line
const POSTS_URL = `${apiUrl}/api/v1/posts`;

export const getPosts = () =>
  fetch(POSTS_URL)
    .then((response) => {
      if (!response.ok) {
        throw new Error("No response from server");
      }
      return response.json();
    })
    .then((result) => {
      return result;
    })
    .catch((error) => {
      console.log(`Error retrieving posts: ${error}`);
    });

export const getPostById = (postsId) =>
  fetch(`${POSTS_URL}/${postsId}`)
    .then((response) => {
      if (!response.ok) {
        throw new Error("No response from server");
      }
      return response.json();
    })
    .then((result) => {
      return result;
    })
    .catch((error) => {
      console.log(`Error retrieving post: ${error}`);
    });
```

The `.js` controller file in the lightning web component bundle can then import the `dataService`. We also can use `routeParams` from `@lwce/router` to grab URL parameters and use them as input to our database queries. Below is the source code for `post.js` which will grab the `id` parameter from the URL, pass that into the `getPostById` function, and parse the results from the database. Finally, the post details can be passed to the lightning web component's HTML, and rendered to the user.

```js
import { LightningElement, wire } from "lwc";
import { getPostById } from "../../data/dataService";
import { routeParams } from "@lwce/router"; // eslint-disable-line
export default class Post extends LightningElement {
  @wire(routeParams) params;

  thePost = {};

  connectedCallback() {
    getPostById(this.params.id)
      .then((result) => {
        if (result.length > 0) {
          let post = result[0];
          let postWithDetail = {
            id: post.id,
            body: post.body,
            post_name: post.post_name,
            formatted_date: this.formatDate(new Date(post.post_date)),
            title: post.title,
            link: "/blog/" + post.post_name,
          };
          this.thePost = postWithDetail;
        }
      })
      .catch((error) => {
        console.log(`Error retrieving posts: ${error}`);
      });
  }

  formatDate(theDate) {
    let options = {
      weekday: "long",
      year: "numeric",
      month: "long",
      day: "numeric",
    };
    return theDate.toLocaleDateString("en-US", options);
  }
}
```

```html
<template>
  <div class="slds-p-around_large">
    <div class="slds-box">
      <div class="slds-text-heading_medium">
        <lwce-link href="{thePost.link}">{thePost.title}</lwce-link>
      </div>
      <p class="slds-p-vertical_medium slds-border_bottom">
        <b> {thePost.formatted_date} </b>
      </p>
      <br />
      <lightning-formatted-rich-text
        value="{thePost.body}"
      ></lightning-formatted-rich-text>
    </div>
  </div>
</template>
```

## We're Almost Complete...

Awesome! We have accomplished a lot of things at this point; we have:

- Created a Postgres database
- Defined the entities (one table in this case)
- Inserted and modified some rows
- Created a REST interface for interacting with the database
- Written a component which uses the defined REST interface to fetch rows from the database

Things are looking pretty good, but we still need to take some final steps to make our web app production ready and available at the domain name of our choice. Stay tuned for the final post in this blog series where we will discuss how to get your web app up and running using Heroku.
