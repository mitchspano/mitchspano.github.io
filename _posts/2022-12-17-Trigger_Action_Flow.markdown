---
layout: post
title: "Trigger Action Flow"
date: 2022-12-17 19:43:24 -0500
---

# Trigger Action Flow

If you've been following the development of the [Apex Trigger Actions Framework](https://github.com/mitchspano/apex-trigger-actions-framework), you may have heard about our previous efforts to provide interoperability between Apex and Flow based actions. While these efforts were well-intentioned, the ergonomics and performance profile of the resulting flows left much to be desired. In short, they were slow and clunky, and didn't offer much in the way of practical benefits.

I am very happy to announce that there has recently been a major breakthrough in the flows portion of the Trigger Actions Framework. Through the use of the `Invocable` namespace, we are now able to offload trigger processing from Apex to Flow in a performant and bulkified manner. This means that Trigger Action flows are now much faster and more efficient, with an ergonomics and performance profile that is very comparable to Record Triggered flows. With this recent development, **the framework now offers complete and performant interoperability with granular control over the order of execution between your Flow and Apex automations**.

This level of control over your automation has never been possible on Salesforce. The vision of an “automation studio view” for all of your record triggered automation has come to fruition.

![Automation Studio](/images/triggerActionFlow/automationStudio.png)

---

## Comparison with Record Triggered Flows

[Record Triggered Flows](https://trailhead.salesforce.com/content/learn/modules/record-triggered-flows/build-a-record-triggered-flow) are a very powerful automation tool for Salesforce administrators. The ability to define automation with drag and drop components instead of code is one of the main value propositions of the Salesforce platform. Let's compare Trigger Action flows with Record Triggered flows.

### Order of Execution

The primary issue with Record Triggered Flows is that they **always execute before your Apex triggers**, therefore your choice of automation tool has order of execution consequences. The main advantage of Trigger Action flows is that they allow for granular control of the relative order of execution of both Apex and Flow based automations.

### Ergonomics

To illustrate the similarities between Trigger Action flows and Record Triggered flows, we have created the same automation in each of the technologies. This automation detects if an account records's name has changed, and if so, the account's description will be set to a default value.

Here is how we would define that automation using Record Triggered flows and the same automation using Trigger Action flows:

| Record Triggered                                                  | Trigger Action Flow                                                   |
| ----------------------------------------------------------------- | --------------------------------------------------------------------- |
| ![RecordTriggered](/images/triggerActionFlow/recordTriggered.png) | ![TriggerActionFlow](/images/triggerActionFlow/triggerActionFlow.png) |

The ergonomics are almost identical. Trigger Action flows should feel very familiar to any admin or developer working on the platform.

### Performance

We created the above mentioned flows and set up the following anonymous Apex script to help us benchmark the performance of Trigger Action flows compared to Record Triggered flows:

```java
System.SavePoint sp = Database.setSavepoint();

List<Account> accounts = new List<Account>();
for (Integer i = 0; i < 200; i++) {
   accounts.add(new Account(Name = ’Foo ’ + i));
}
insert accounts;
for (Integer i = 0; i < 200; i++) {
   accounts[i].Name = ’Bar ’ + i;
}

Long before = System.currentTimeMillis();
update Accounts;
Long after = System.currentTimeMillis();

System.debug('MS To perform 200 record updates: ' + (after - before));
Database.rollback(sp)
```

We then executed the script a few times in three scenarios in a scratch org:

- No Automation
- Record Triggered Flow
- Trigger Action Flow

Here are the performance benchmarks captured in milliseconds:

| No Automation | Record Triggered Flow | Trigger Action Flow |
| ------------- | --------------------- | ------------------- |
| 1581          | 1404                  | 1694                |
| 1359          | 1471                  | 1455                |
| 1223          | 1407                  | 2162                |
| 1919          | 2028                  | 1626                |
| **Average**   | **Average**           | **Average**         |
| 1472          | 1580                  | 1740                |

You can see that there is a very slight performance overhead with Trigger Action flows, but the performance profile is still very attractive and sufficient for most automation needs.

### Advantages of Record Triggered Flows

There are some things that Trigger Action Flows cannot do. Record Triggered flows can have entry criteria defined, which allows you to specify conditions that must be met before the flow is triggered. Trigger Action flows do not have this capability, so they will always be triggered and flow interviews will always be created. Record Triggered flows also support time-based actions, which allows you to schedule actions to be performed at a specific date and time; Trigger Action flows do not support this feature. Finally, Record Triggered flows can support workflow outbound messages to send data to external systems while Trigger Action flows cannot.

### Advantages of Trigger Action Flows

Despite these differences, Trigger Action flows have some key advantages over Record Triggered flows. Trigger Action flows also have interoperability with Apex, which allows you to exercise complete control of the order of execution of all of your record triggered logic. Trigger Action flows can also take advantage of the [bypass mechanisms](https://github.com/mitchspano/apex-trigger-actions-framework#bypass-mechanisms), which allow you to control when and how the flow is executed. Trigger Action flows can also be used in all seven trigger contexts including after delete and after undelete. Additionally, Trigger Action flows support the `addError()` invocable apex action, which allows you to add formatted error messages to records directly from the flow.

## Examples

Here are some example flows within the Apex Trigger Actions Framework operating in all seven trigger contexts.

<iframe
    title="video"
    width="700"
    height="420"
    src="https://www.youtube.com/embed/6HHfqCaGRR4?autoplay=1&mute=1"
    allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture"
    allowfullscreen
></iframe>

---

## Go Forth and Build!

This latest breakthrough with the flows portion of the Trigger Actions framework has enabled a new level of interoperability between Apex and Flow. No longer do teams need to have a religious debate about which automation tool to use. Admins and developers can now make their automations with whatever tool makes the most sense for the job, then use the Apex Trigger Actions Framework to plug them together harmoniously.

We believe that this new development in the Trigger Action flow architecture will be a game-changer for Salesforce teams, and we can't wait to see the amazing automations you will create with it. Please check out the project's repository and share these new capabilities with your team!
