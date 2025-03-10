# {{ tracker-full-name }} revision history for February 2023

* [Gantt chart for issue queues and filters](#gannt-queue-filter)
* [Setting up a color palette in a Gantt chart for projects](#gantt-colors)
* [Displaying external blockers in a list of issues on a project's Gantt chart](#ext-blockers-gannt)
* [Workflow templates](#work-templates)
* [Automatically adding issues to a board](#auto-add-to-board)
* [Using a planning poker on a board](#poker)
* [Retaining a list of projects](#projects-preset)
* [Displaying files attached to an issue](#attaches)
* [Dashboard access rights](#access-dashboards)

## Gantt chart for issue queues and filters {#gannt-queue-filter}

You can plan your work on the calendar as a Gantt chart for issue [queues](../user/queue.md) and [filters](../user/create-filter.md). To switch to the chart, select a queue or set up an issue filter and open the **Gantt chart** tab.

## Setting up a color palette in a Gantt chart for projects {#gantt-colors}

You can now set up the issue color manually in a [project's Gantt chart](../manager/gantt-project.md). To enable this, click **Chart settings** and select **Color: manual**. To choose the issue color, hover over its name in the table row on the left and click ![](../../_assets/horizontal-ellipsis.svg) → ![](../../_assets/tracker/svg/gannt-palette.svg) **Choose color**.

## Displaying external blockers in a list of issues on a project's Gantt chart {#ext-blockers-gannt}

If an issue has blockers outside the current project, you will see the ![](../../_assets/tracker/svg/blocker.svg) icon with their number to the left of the issue bar.

You can now enable displaying external blockers in the issue list. To do so, go to ![](../../_assets/tracker/svg/gantt-settings-button.svg)&nbsp;**Chart settings** and select **Show external blockers**. They will be shown in gray under the dependent issue in the issue list. If you selected to display issues as a hierarchy, the blockers will be shown under the issue branch.

## Workflow templates {#work-templates}

To set up your workflow, you can now use templates that contain ready-made sets of tools. Currently, the **Project management**, **Development**, **Support**, and **Basic** templates are available.

You can create a workflow from a template under [**My page**]({{ link-tracker }}pages/my) using the **Workflow templates** widget.

## Automatically adding issues to a board {#auto-add-to-board}

You can now set up [new boards](../manager/agile-new.md) so that they automatically display new issues from a filter.
To do this, click ![](../../_assets/tracker/svg/settings.svg) **Settings** in the top right corner of your board and select **Automatic issues import and removal**. Next, set the filter parameters under **Adding issues to the board**.

## Using a planning poker on a board {#poker}

<q>Scrum</q> boards in the [new interface](../manager/agile-new.md) now support [planning poker]({{ link-wiki-poker }}), a tool for estimating issue complexity in collaboration with team members. Estimates are made using a deck of cards with a different number of [Story Points](../manager/agile.md#dlen_sp).

To set the planning poker parameters, click ![](../../_assets/tracker/svg/settings.svg) **Settings** in the top-right corner of the board and select **Poker**.

## Retaining a list of projects {#projects-preset}

Now, when setting up a filter for a [list of projects](../manager/my-projects.md), both the filter parameters and the selected preset (**My projects**, **Created by me**, etc.) are retained. The preset is restored when you open the **All projects** page again.

## Displaying files attached to an issue {#attaches}

[Files attached](../user/attach-file.md) from comments are now displayed in the **Attachments** tab. If files are added directly to an issue, they are only displayed in its summary.

## Dashboard access rights {#access-dashboards}

The {{ tracker-name }} new interface now also allows you to change the owner of a dashboard and set up user permissions to it.
