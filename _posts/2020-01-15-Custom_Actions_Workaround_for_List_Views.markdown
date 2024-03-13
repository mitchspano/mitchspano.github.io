---
layout: post
title: "Custom Actions Workaround for List Views"
date: 2020-01-15 19:43:24 -0500
---

Salesforce's List views are a powerful tool to help users organize and navigate to records. Unfortunately, the only actions they support are the standard Edit Delete, and Change Owner.

![](/images/customListViewAction/standard-Actions.png)

While it is impossible to add actions to this list, we can be clever and expose custom actions using Formula Fields and Lightning Components.

#### Step 1: Create Lightning Component

Create a component and make it implement the lightning:isUrlAddressable interface. In this project, the component we are using is the Aura Component called myComponent.

```xml
<aura:component implements="flexipage:availableForAllPageTypes,force:appHostable,lightning:isUrlAddressable" access="global">;
    <aura:handler name="init" value="{!this}" action="{!c.initialize}">;
    <div class="slds-box slds-theme_default">;
        Some Stuff Here
    </div>;
</aura:component>
```

```js
({
  initialize: function (component, event, helper) {
    var recordId = component.get("v.pageReference").state.c__id;
    console.log("Here is record Id" + JSON.stringify(recordId));
  },
});
```

#### Step 2: Create a Hyperlink Formula Field

This formula will format the URL for the component mentioned above and pass the record Id as a parameter to the component itself. In this project, the formula field is on Account and is called 'Action'.

```
HYPERLINK("/lightning/cmp/c__myComponent?c__id=" + Id, "Click Me!","_self")
```

#### Step 3: Add Formula Field to List View.

Add this formula field to a list view. Typically you would want to put this in the right-most column in the list view so it is next to the actions drop down.

![](/images/customListViewAction/inline-action.gif)

#### Component Use

Now you can do whatever you need to do with your component and the record Id passed to it. In this scenario, we are just logging it to the console. Perhaps you might want to perform an update to the record then navigate to it, maybe you want to pass that Id into another Aura or Lightning Web Component. You can get creative with this approach!

Make sure to share this with your Salesforce Administrator or Developer, and check out the repository located [here](https://github.com/mitchspano/dynamicDataTable).
