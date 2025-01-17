---
title: "Icon Gallery"
linkTitle: "Icon Gallery"
weight: 60
date: 2021-07-12
tags: 
description: >
  A gallery of all the icons in use in stroom for reference.
---

This page can be used as a reference for finding icons and their filenames to use them in the documentation.

See [here]({{< ref "using-images#stroom-icons" >}}) for how to add icons to pages.


## General icons

General icons used in Stroom.

**Example**: {{< stroom-icon "edit.svg">}} `{{</* stroom-icon "edit.svg" */>}}`

{{< stroom-icons-gallery "images/stroom-ui/" >}}


## Pipeline element icons

Icons used for the different pipeline elements.

**Base path**: `pipeline`  
**Example**: {{< stroom-icon "pipeline/stream.svg">}} `{{</* stroom-icon "pipeline/stream.svg" */>}}`

{{< stroom-icons-gallery "images/stroom-ui/pipeline/" >}}


## Document type icons

Icons used for the different _document_ entity types, i.e. those seen in the explorer tree.

**Base path**: `document`  
**Example**: {{< stroom-icon "document/Index.svg">}} `{{</* stroom-icon "document/Index.svg" */>}}`

{{< stroom-icons-gallery "images/stroom-ui/document/" >}}


## Dashboard icons

Icons used on the Dashboard

**Base path**: `dashboard`.

{{< stroom-icons-gallery "images/stroom-ui/dashboard/" >}}


## Assorted UI elements

Assorted images used in the user interface.

**Base path**: `assorted`  
**Example**: {{< stroom-icon "assorted/popup.png" "Menu selection">}} `{{</* stroom-icon "assorted/popup.png" "Menu selection" */>}}`

{{< stroom-icons-gallery "images/stroom-ui/assorted/" >}}


## Updating this gallery

This gallery of icons can be updated by running the following script which will copy all the icons from the stroom repository.
You should ensure your local stroom repository is up to date and on the correct branch before running this.

{{< command-line >}}
./update_stroom_icons.sh <stroom repo root>
{{</ command-line >}}

e.g. 

{{< command-line >}}
./update_stroom_icons.sh ../v7stroom/
{{</ command-line >}}
