---
layout: post
title: "Shared Lightning Web Component State"
date: 2022-01-03 19:43:24 -0500
---

# How to Share State Across Multiple LWCs

Lightning Web Components are pretty amazing, but getting multiple components to work together on a single page can be tricky. Sometimes, multiple components need access to the same data. Traditionally, Salesforce developers have duplicated data across multiple components - often involving multiple round trips to the server (one for each component). In this post, we will share some techniques to help share state across multiple Lightning Web Components on a single page. These techniques will help you to share data which might have a large footprint, or be computationally expensive to fetch repeatedly.

## Component Overview

The approach outlined below will work for multiple unique components rendered on a lightning page, but for the sake of this demonstration, we will consider a very simple component which will be rendered multiple times on the same page. This component is called `myComponent` and has three buttons:

- Set to 'Blue'
- Set to 'Red'
- Refresh

The component also will display the `Current State`, which can be altered by pressing the buttons. The `Current State` only stores a color in this simple example, but you can imagine these components sharing more complicated data such as the list of current fields which are accessible on a given sObject for the running user or all groups that the running user is a member of.

![Buttons Image](/images/sharedState/buttons.png)

This component is rather simple, but if we want multiple instances of this component (or others) to share access to the `Current State`, then we need to introduce something beyond the standard `myComponent.js` file which controls the component.

## Files in the Component Bundle

To allow for this shared state, we will introduce a new file into our Lighting Web Component bundle called `sharedState.js`.

```
└── force-app
    └── main
        └── default
            └── lwc
                └── myComponent
                    ├── myComponent.html
                    ├── myComponent.js
                    ├── myComponent.js-meta.xml
                    └── sharedState.js
```

This file will use JavaScript object literals to create a singleton
which will store the shared state and grant access across multiple
components.

```js
// sharedState.js
let _data;

const SharedState = {
  setData: (newVal) => {
    _data = newVal;
  },
  getData: () => {
    return _data;
  },
};

Object.freeze(SharedState);

export { SharedState };
```

This file will import the `SharedState` object from `sharedState.js` and use it to update the shared state upon button press.

```js
// myComponent.js
import { LightningElement, track } from "lwc";
import { SharedState } from "./sharedState";

export default class MyComponent extends LightningElement {
  @track stateData;

  refreshStateData() {
    return (this.stateData = SharedState.getData());
  }

  updateState(newValue) {
    SharedState.setData(newValue);
    this.refreshStateData();
  }

  updateStateToBlue() {
    this.updateState("Blue");
  }
  updateStateToRed() {
    this.updateState("Red");
  }
}
```

Notice how one component will set the state to a certain color, and when the others get the refreshed state, they will also see the same color specified by the different component. Excellent - we are able to share the same state across multiple components!

![](/images/sharedState/manualRefresh.gif)

However, we do have a limitation with our current setup. The other components on the screen must manually refresh the state when it is updated somewhere on the page. This might be fine if you only need to check the shared state upon a button click or a specific event, but you might want to automatically refresh the neighboring components whenever the state is updated.

### Lightning Message Service For The Win

Lighting Message service is a powerful part of the Salesforce platform. It can be used to propagate messages across the Lightning page to communicate with components that are not composed in a parent/child relationship. We will use LMS to publish a refreshState event whenever the shared state is updated. To get started, we must first create a message channel.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningMessageChannel xmlns="http://soap.sforce.com/2006/04/metadata">
    <masterLabel>refreshState</masterLabel>
    <isExposed>true</isExposed>
    <description>This is used to refresh the shared state.</description>
</LightningMessageChannel>
```

With this message channel defined, we now need to import it and use it in the `myComponent.js` file.

```js
// myComponent.js
import { LightningElement, track, wire } from "lwc";
import { SharedState } from "./sharedState";
import {
  APPLICATION_SCOPE,
  MessageContext,
  publish,
  subscribe,
  unsubscribe,
} from "lightning/messageService";
import refreshState from "@salesforce/messageChannel/refreshState__c";

export default class MyComponent extends LightningElement {
  @wire(MessageContext)
  messageContext;

  @track stateData;

  refreshStateData() {
    return (this.stateData = SharedState.getData());
  }

  updateState(newValue) {
    SharedState.setData(newValue);
    publish(this.messageContext, refreshState);
  }

  updateStateToBlue() {
    this.updateState("Blue");
  }
  updateStateToRed() {
    this.updateState("Red");
  }

  subscribeToMessageChannel() {
    if (!this.subscription) {
      this.subscription = subscribe(
        this.messageContext,
        refreshState,
        (message) => this.handleMessage(message),
        { scope: APPLICATION_SCOPE }
      );
    }
  }

  unsubscribeToMessageChannel() {
    unsubscribe(this.subscription);
    this.subscription = null;
  }

  handleMessage(message) {
    this.refreshStateData();
  }

  connectedCallback() {
    this.subscribeToMessageChannel();
  }

  disconnectedCallback() {
    this.unsubscribeToMessageChannel();
  }
}
```

![](/images/sharedState/autoRefresh.gif)

Voilà! We now have multiple components sharing the same state, and
updating their rendered HTML automatically upon updates to that
shared state. You can use these techniques to improve the
performance of your application when multiple components need to
share state which is large or computationally expensive to retrieve
repeatedly.
