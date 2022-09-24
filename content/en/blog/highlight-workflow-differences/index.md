---
title: "Showing Changes Between Two Workflow Versions"
description: ""
lead: "We discuss how to display the differences between two workflow versions in Texera."
date: 2022-09-22T15:16:36-07:00
lastmod: 2022-09-22T15:16:36-07:00
draft: false
weight: 50
# images: ["highlight-workflow-differences.jpg"]
contributors: ["Jingqi Yao"]
---
Texera has a version control system that automatically saves workflow versions so that users can easily view or restore early versions. The current implementation only shows each historical workflow. Users may find it hard to see how it differs from another workflow version, especially when there is only a tiny change. We want to improve the usability of the version control system by visually displaying the difference between two workflow versions.

### Task Overview
The task can be divided into two phases: calculating and showing the differences between two workflow versions.

In this task, we are showing all differences in an aggregated style (i.e., only displaying the historical version of workflow and highlighting the changes with respect to the current version). Although showing both versions side-by-side with highlights might better display the changes, the current Texera system does not support displaying two workflows simultaneously. Showing workflow differences in two windows is left for future enhancement.

To keep the user interface neat, we are only showing the differences of operators, while similar logic can be applied to other elements in the workflow, e.g., links and comment boxes. For simplicity of the user interface, operators' position changes are not considered for now.

### An Example
The following is a current workflow:

<img src="current_workflow.png"  width="700">

When the user selects Version #3 on the list on the right (highlighted in red), we retrieve the version from the history and highlight the differences. In this example, the Regular Expression operator was deleted in the current version, the Filter operator was added, and the "Worker count" property of the Python UDF is changed from 1.

<img src="historical_workflow.png"  width="700">

### Implementation
Once the user selects a particular version, we calculate the difference between that version and the current version and highlight their differences.

#### Calculating the Differences
The `getWorkflowsDifference` function takes two workflow objects and returns all the IDs of operators that are different in the two versions.

For each operator, the change can be categorized into three types: 
- **Added**: the operator is added by the user;
- **Modified**: the operator's properties are modified;
- **Deleted**: the operator is deleted.

To facilitate later comparisons, we create a map for each version, with an operatorID as the key, and the operator's properties as the value.  Then we run two passes on the map.

1. In the first pass, we iterate through the operators of the historical version. Our goal is to find the added and modified operators. To find the added operators, we simply check if the operatorID exists in the map of the current version. If the operatorID exists in both versions, we then perform a deep comparison of the two operators' properties associated with the operatorID using the `isEqual()` method from `lodash`, a JavaScript utility library. If they are different, we add the operatorID to the list of modified operators.

2. In the second pass, we iterate through the operators of the current version to generate a list of operators that are deleted.

So far, we have obtained all the IDs of operators that have been added, modified, or deleted between the two versions.

#### Highlighting the Differences
We only display the historical version on the user interface, and all differences should be visible on one canvas.

For deleted and modified operators, we simply fill the `rect.boundary` attribute of the operator to give the operators a highlighted background. We use green to indicate deletion and orange to indicate modification. Furthermore, we highlight the outline of modified properties in the property window on the right-hand side.

However, we cannot do the same for the added operators because they do not exist in the historical version we are displaying. Instead, we put brackets on the connected operators to indicate that there are additional operators between them in the current version.
<p align="center">
    <img src="highlight_meaning.png"  width="500">
</p>

### Summary 
In this blog, we share how we visually display the differences between the workflows in the Texera version control system. This enhancement would improve the usability of the current version control system and help users to view and restore the right version they want.

#### Acknowledgement
Thanks to Sadeem Alsudais, Professor Li, and the Texera team, for their help in the task and this blog.
