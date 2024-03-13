---
layout: post
title: "Dynamic Row Actions With Lightning Datatable"
date: 2020-02-11 19:43:24 -0500
---

Lightning Datatable is one of the more powerful base Lightning Web
Components. This component allows developer to display multiple rows
of objects with dynamically formatted columns for a variety of data
types. The supported data types are: action, boolean, button,
button-icon, currency, date, date-local, email, location, number,
percent, phone, text, and url. The data type that we are most
interested in is action.

![](/images/dynamicRowActions/standard-row-actions.png)

Lightning Datatable's action column allows a developer to specify a
list of actions which are available for the user to select.

This is great for most use cases, but the actions are defined as a
constant, which means that <u>every row must share the same actions</u>.

After scouring Salesforce's existing documentation and forum posts
from all corners of the internet, I could not locate any information
about how to define dynamic actions at the row level. I would like
to offer a big shout out to Jeff Cook (@jet89cook) for helping me to
discover a solution to this problem.

### Define a Datatable

This should look familiar as it's almost identical to the examples
in Salesforce's documentation.

```html
<template>
  <div class="slds-box slds-theme_default">
    <lightning-datatable
      key-field="id"
      data="{data}"
      columns="{columns}"
      onrowaction="{handleRowAction}"
      hide-checkbox-column
    >
    </lightning-datatable>
  </div>
</template>
```

### Define your Columns

This is where the magic happens. We must define the `typeAttributes` property of the action column to achieve the desired functionality.

```js
columns = [
  {
    label: "Name",
    fieldName: "name",
  },
  {
    label: "Profession",
    fieldName: "profession",
  },
  {
    type: "action",
    typeAttributes: {
      rowActions: { fieldName: "rowActions" },
    },
  },
];
```

#### Define your Data

We need to make sure that every element of our data has an array
called “rowActions” which stores the dynamic actions.

```js
const USE_LIGHTSABER = "Use Lightsaber";
const MIND_TRICK = "Mind Trick";

export default function getData() {
  return [
    {
      name: "Luke Skywalker",
      profession: "Jedi Knight",
      rowActions: [
        {
          label: USE_LIGHTSABER,
          name: USE_LIGHTSABER,
        },
        {
          label: MIND_TRICK,
          name: MIND_TRICK,
        },
      ],
    },
    //more data
  ];
}
```

#### Put it all Together

Once everything else is all wired together, we can see that every row has dynamic actions as we have defined.

![](/images/dynamicRowActions/dynamic-Row-Actions.gif)

#### Limitations

Unfortunately there is one minor limitation to this implementation.
If a row has no actions, the drop-down arrow will still show and
clicking it will display an empty list. It would be nice if this was
not the case, but I have not discovered any way to hide the action
arrow.

![](/images/dynamicRowActions/empty-action.png)

Please feel free share this with any Salesforce Developers you know
who might face this issue. For more details and information, please
check out the project repository located
[here](https://github.com/mitchspano/dynamicDataTable).
Thank you!
