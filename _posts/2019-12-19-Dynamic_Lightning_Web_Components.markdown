---
layout: post
title: "Dynamic Lightning Web Components"
date: 2019-12-19 19:43:24 -0500
---

One of the biggest challenges in making the change to Lightning Web Components is the lack of inline expression. For example, a developer might want to do something like this:

```html
<button onclick="handleButtonClick('Hello World');"></button>
```

This will not work with lightning web components. This is particularly frustrating when you have a collection of objects with buttons and want to do something particular to the object you pressed the button for.

However, we can be clever and use public attributes to allow for an event with a dynamic name and payload to be fired. Below is an example of a component called eventDetailButton. This component renders a lightning button and fires a custom payload with a dynamic detail whenever the button is clicked.

**EventDetailButton.js**

```js
import { LightningElement, api } from "lwc";

export default class EventDetailButton extends LightningElement {
  @api detail;
  @api eventName;
  @api name;
  @api value;
  @api label;
  @api variant;
  @api iconName;
  @api iconPosition;
  @api type;

  buttonClick() {
    const event = new CustomEvent(this.eventName, {
      detail: this.detail,
    });
    this.dispatchEvent(event);
  }
}
```

**EventDetailButton.html**

```html
<template>
  <lightning-button
    name="{name}"
    value="{value}"
    label="{label}"
    variant="{variant}"
    icon-name="{iconName}"
    icon-position="{iconPosition}"
    type="{type}"
    onclick="{buttonClick}"
  >
  </lightning-button>
</template>
```

Now, developers can use the component by dynamically setting an event name and custom detail.

```html
<template for:each="{myCollection}" for:item="anyObject">
  <c-event-detail-button
    variant="brand"
    event-name="custombuttonclick"
    label="Example"
    detail="{anyObject.property}"
    oncustombuttonclick="{handleClick}"
  >
  </c-event-detail-button>
</template>
```

```js

handleClick(event) {
    let detail = event.detail;
    console.log(JSON.stringify(detail));
}
```

![](/images/dynamicLightningWebComponents/unlock.gif)

Note that the event-name is the name of the custom event handler - in this example, that is _custombuttonclick_.

This custom event handling allows the user to more effectively work with dynamic collections at run time.

![](/images/dynamicLightningWebComponents/Event-Detail-Demo.gif)
