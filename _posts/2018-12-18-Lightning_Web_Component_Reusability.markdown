---
layout: post
title: "Lightning Web Component Reusability"
date: 2018-12-18 19:43:24 -0500
---

For even the simplest of user interface components, the cost in Salesforce labor and development may become significant. Requirements must be gathered, analyzed, and prioritized. User experience teams have to perform a plethora of tests. An array of stakeholders are required to sign off throughout the whole process. It is in the best interest of developers, architects, and product owners to reuse components whenever possible.

Imagine a scenario where a developer has created a component which is to be used on an Account page. This component just displays some Account fields and attributes while using the lightning-record-form component and is called **AccountHighlights**.

![Account Highlights](/images/lightningWebComponentReusability/account-Highlights.gif)

**AccountHighlights.js**

```javascript
import { LightningElement, api } from "lwc";
export default class AccountHighlights extends LightningElement {
  @api recordId;
}
```

**AccountHighlights.html**

```html
<template>
  <div class="slds-theme_default slds-box">
    <lightning-record-form
      record-id="{recordId}"
      columns="5"
      object-api-name="Account"
      layout-type="Compact"
      mode="view"
    >
    </lightning-record-form>
  </div>
</template>
```

**AccountHighlights.js-meta.xml**

```html
<?xml version="1.0" encoding="UTF-8" ?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
  <apiVersion>47.0</apiVersion>
  <isExposed>true</isExposed>
  <targets>
    <target>lightning__RecordPage</target>
  </targets>
  <targetConfigs>
    <targetConfig targets="lightning__RecordPage">
      <objects>
        <object>Account</object>
      </objects>
    </targetConfig>
  </targetConfigs>
</LightningComponentBundle>
```

Users love this component and after it is launched, they request to see that same component on related contact records. We can use the Lightning Web Component Framework’s @api decorator to modify the recordId which is passed to the accountHighlights component.

Most of the time, when you use the @api recordId notation, you are simply using it to fetch the recordId of the currently viewed record. However, the @api decorator also exposes those properties as public, meaning that other components can explicitly set that value.

Here is an example of a component called contactAccountHighlights. This component uses the getRecord API in order to fetch the AccountId for the given contact, then pass that AccountId into the **ContactAccountHighlights** component.

![contact-Account-Highlights](/images/lightningWebComponentReusability/contact-Account-Highlights.gif)

**ContactAccountHighlights.js**

```js
import { LightningElement, wire, api } from "lwc";
import { getRecord } from "lightning/uiRecordApi";

export default class ContactAccountHighlights extends LightningElement {
  @api recordId;
  //wire getRecord with this record's Id to property called myContact
  @wire(getRecord, { recordId: "$recordId", fields: ["Contact.AccountId"] })
  myContact;

  //extract the AccountId field value from the myContact object
  get accountId() {
    if (
      this.myContact &&
      this.myContact.data &&
      this.myContact.data.fields &&
      this.myContact.data.fields.AccountId &&
      this.myContact.data.fields.AccountId.value
    ) {
      return this.myContact.data.fields.AccountId.value;
    }
    return undefined;
  }
}
```

```html
<template>
  <template if:true="{accountId}">
    <!--Pass the AccountId from the contact record into the recordId for the accountHighlights component-->
    <c-account-highlights record-id="{accountId}"> </c-account-highlights>
  </template>
</template>
```

This component is pretty useful, but we can take it one step further. What happens if the users also want to see this component on related Opportunities? In order to make it reusable and configurable, we will create a new component called genericParentAccountHighlights and edit the component’s configuration file to allow administrators to configure the component within the context of lightning page builder.

**GenericParentAccountHighlights.js**

```js
import { LightningElement, wire, api } from "lwc";
import { getRecord } from "lightning/uiRecordApi";

export default class GenericParentAccountHighlights extends LightningElement {
  @api recordId;
  @api objectApiName;
  @api accountFieldName;
  //wire getRecord with this record's Id to property called myRecord
  @wire(getRecord, { recordId: "$recordId", fields: "$fields" }) myRecord;

  //extract AccountId from generic sobject record
  get accountId() {
    if (
      this.accountFieldName &&
      this.myRecord &&
      this.myRecord.data &&
      this.myRecord.data.fields &&
      this.myRecord.data.fields[this.accountFieldName] &&
      this.myRecord.data.fields[this.accountFieldName].value
    ) {
      return this.myRecord.data.fields[this.accountFieldName].value;
    }
    return undefined;
  }

  // define the combination of sobject name + account relationship field name
  get accountFieldNameForGetRecord() {
    return this.objectApiName + "." + this.accountFieldName;
  }
  // flag to show error if configured improperly
  get isMissingFieldName() {
    return !this.accountFieldName;
  }

  // create dynamic field array
  get fields() {
    let returnValue = [];
    returnValue.push(this.accountFieldNameForGetRecord);
    return returnValue;
  }
}
```

**GenericParentAccountHighlights.html**

```html
<template>
  <!-- Friendly error message for administrators -->
  <template if:true="{isMissingFieldName}">
    <div class="slds-theme_default slds-box">
      This component has not been configured properly. Please contact your
      system administrator.
    </div>
  </template>
  <template if:true="{accountId}">
    <c-account-highlights record-id="{accountId}"> </c-account-highlights>
  </template>
</template>
```

**GenericParentAccountHighlights.js-meta.xml**

```xml

<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>47.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__RecordPage</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__RecordPage">
          <!--expose the accountFieldName variable to lightning page builder-->
            <property name="accountFieldName"
                type="String" default="AccountId"
                label="Enter the API Name of the field which stores the parent Account’s record ID."/>
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

![](/images/lightningWebComponentReusability/Generic-Highlights-App-Builder.gif)

Now we can use this component on any object that has a relationship to the Account Object including standard and custom objects.

Hopefully this example helps to illustrate the powerful reusability of Lightning Web Components. Be sure to check out the project’s repository located
[here](https://github.com/mitchspano/ParentChildComponentDemo")
