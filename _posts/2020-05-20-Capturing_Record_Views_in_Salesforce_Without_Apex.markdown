---
layout: post
title: "Capturing Record Views in Salesforce Without Apex*"
date: 2020-05-10 19:43:24 -0500
---

Oftentimes, sales and service managers want greater visibility into their team's workflow. With Salesforce's Field History Tracking, you can know when someone changed a status of a record they own to “Working” or “In Progress”, but management has no idea how long it took for that representative to set the status after they began looking at the record. It is important to know that this problem is not unique to standard objects like Cases and Tasks - it could be applied on any sObject, standard or custom, so we want to build a robust reusable solution which will capture when the owner of a record initially views it.

![](/images/viewRecord/viewedDate.gif)

While it is impossible to add actions to this list, we can be clever and expose custom actions using Formula Fields and Lightning Components.

#### Lightning Web Component

The majority of the work in this project is completed by a Lightning Web Component called **RecordViewedStamp**.

The HTML for this component is empty, so it will not be visible to the end users. When an administrator drags this component onto a Lightning Record Page, it uses a dynamic picklist to display the list of updateable date/time fields on the particular object they are viewing. This selected field is where the date/time will be stamped when the owner of a record initially views it.

This Lightning Web Component has a few responsibilities:

- The LWC must read the passed in target date/time field as specified by the System Administrator who added the component to a record page.
- The LWC must dynamically fetch the OwnerId and the value for the target date/time field for the particular record the user is viewing. To perform this we will use the getRecord service with the optionalFields parameter - this allows for dynamic runtime definition of the fields requested for the object.
- If the owner is the current user and the date/time has not yet been set it must update that time to now. To perform this we will use theupdateRecord service.
- Finally, if an update has been made, the component must force the page to refresh in order to prevent the "this record has already been modified" message from popping up when the user saves their record edits. We will use the hacky `eval("$A.get('e.force:refreshView').fire();");` to force this to occur but for a more robust way to refresh Lightning Web Components and have them play nicely with other Aura Components on your page, check out my LWC Refresh Demonstration Project.

This component uses <ins>no Apex</ins> to actually perform the actions outlined above.

```js
import { LightningElement, wire, api, track } from "lwc";
import { getRecord, updateRecord } from "lightning/uiRecordApi";
import Id from "@salesforce/user/Id";

export default class RecordViewedStamp extends LightningElement {
  @api recordId;
  @api objectApiName;
  @api targetDateTimeField;
  @track _myFields;
  _userId = Id;
  _wiredResponse;

  connectedCallback() {
    if (this.targetDateTimeField && this.objectApiName) {
      this._myFields = [];
      this._myFields.push(this.objectApiName + "." + this.targetDateTimeField);
      this._myFields.push(this.objectApiName + ".OwnerId");
    }
  }

  @wire(getRecord, { recordId: "$recordId", optionalFields: "$_myFields" })
  wiredRecord(response) {
    this._wiredResponse = response;
    if (response.error) {
      let message = "Unknown Error";
      if (Array.isArray(response.error.body)) {
        message = response.error.body.map((e) => e.message).join(", ");
      } else if (typeof response.error.body.message === "string") {
        message = response.error.body.message;
      }
      console.log(
        "Error while trying to get the current owner and viewed date/time : " +
          message
      );
    } else if (response.data) {
      let owner;
      let viewedDateTime;
      if (
        response.data &&
        response.data.fields &&
        response.data.fields.OwnerId &&
        response.data.fields.OwnerId.value
      ) {
        owner = response.data.fields.OwnerId.value;
      }
      if (
        response.data &&
        response.data.fields &&
        response.data.fields[this.targetDateTimeField] &&
        response.data.fields[this.targetDateTimeField].value
      ) {
        viewedDateTime = response.data.fields[this.targetDateTimeField].value;
      }
      if (!viewedDateTime && this._userId === owner) {
        this.setViewedDateTime();
      }
    }
  }

  setViewedDateTime() {
    let fields = {};
    fields["Id"] = this.recordId;
    fields[this.targetDateTimeField] = new Date().toISOString();
    let input = { fields };
    updateRecord(input)
      .then(() => {
        eval("$A.get('e.force:refreshView').fire();");
      })
      .catch((error) => {
        console.log("Error while trying to set the Viewed Date/Time");
      });
  }
}
```

#### Without Apex\*

There is no Apex involved in the update of the record, but there is some Apex involved to allow for the dynamic target date/time field selection when within Lightning Page Builder. This class builds a list of the fields which are updateable date/time fields for the particular object that the component is being added to the page for.

```java
public class RecordViewedStampOptions extends VisualEditor.DynamicPicklist {
    VisualEditor.DesignTimePageContext context;

    public RecordViewedStampOptions(
        VisualEditor.DesignTimePageContext context
    ) {
        this.context = context;
    }

    public override VisualEditor.DataRow getDefaultValue() {
        VisualEditor.DataRow returnValue = new VisualEditor.DataRow(
            'none',
            'NONE'
        );
        return returnValue;
    }

    public override VisualEditor.DynamicPicklistRows getValues() {
        VisualEditor.DynamicPicklistRows returnValue = new VisualEditor.DynamicPicklistRows();
        sObjectType mysObjectType = Schema.getGlobalDescribe()
            .get(this.context.entityName);
        Map<String, Schema.sObjectField> fields = mysObjectType.getDescribe()
            .fields.getMap();
        for (Schema.sObjectField field : fields.values()) {
            if (
                field.getDescribe().isUpdateable() &&
                String.valueOf(field.getDescribe().getType()).equals('DATETIME')
            ) {
                returnValue.addRow(
                    new VisualEditor.DataRow(
                        field.getDescribe().getName(),
                        field.getDescribe().getName()
                    )
                );
            }
        }
        return returnValue;
    }
}
```

#### Opportunities for Improvement

There are a few opportunities to improve this component:

- As mentioned before, there are better ways to refresh Lightning Web Components.
- This approach does not work if the sObject does not have an Owner. So if the object is on the detail side of a Master-Detail relationship, it will not work.
- Right now we are just logging the error messages to the console. More robust error handling would be nice to add.
- It would be wise to also implement a trigger action before update when the OwnerId is changed to set the date/time field back to null.

Thank you for reading this post. Be sure to share this with Salesforce Developers, Administrators, and Architects. Also check out the project [repository](https://github.com/mitchspano/RecordViewedDemo)
