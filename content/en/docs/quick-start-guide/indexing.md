---
title: "Indexing"
linkTitle: "Indexing"
weight: 40
date: 2021-07-09
tags:
  - index
description: >
  Indexing the ingested data.
---

Before you can visualise your data with dashboards you have to index the data;
First I opted for creating a specific volume to hold my data, just because I wanted to keep my shards away from the default volumes;

Go to the Tools ➔ Volumes menu

{{< image "quick-start-guide/index/001_volumes.png" >}}Volumes{{< /image >}}

Once the volumes dialogue opens click the blue plus sign at the top left of the window to add a new one

{{< image "quick-start-guide/index/002_volume_list.png" >}}Volume list{{< /image >}}

Select the node where you want the volume to be and the path you want to create, (because we are following the quick-start guide we just have one node and limited size, but we do want to set it as active so we can write documents to it and we want it to be public, because we might want other indexes to use it; your needs might be different.

{{< image "quick-start-guide/index/003_volume_edit.png" >}}Volume edit{{< /image >}}

Click ok and we’re good to go.

Then we can create an index by selecting index item in the explorer tree. You do this in the same way you create any of the items. Just select/create a folder that you want to create the new index in and right click, select New Index.

{{< image "quick-start-guide/index/004_index_new.png" >}}New index{{< /image >}}

Choose a name for your new index

{{< image "quick-start-guide/index/005_index_name.png" >}}Index name{{< /image >}}

In the settings tab we need to specify the volume where we will store our shards

Now you need to add fields to this index.

Firstly there are two mandatory fields that need to be added: `StreamId` and `EventId`

Both should be of type `Id`, stored and indexed with the `Keyword` analyser

{{< image "quick-start-guide/index/006_index_field.png" >}}Index field{{< /image >}}

If you were following the quick-start instruction on ingesting the `mock_stroom_data.csv`, we’ll use those fields here.

Open the fields tab then create the following fields:

Name         | Type   | Store  | Index  | Positions  | Analyser       | Case Sensitive
----         | ----   | -----  | -----  | ---------  | --------       | --------------
StreamId     | Id     | Yes    | Yes    | No         | Keyword        | false
EventId      | Id     | Yes    | Yes    | No         | Keyword        | false
Id           | Text   | Yes    | Yes    | No         | Keyword        | false
Guid         | Text   | Yes    | Yes    | No         | Alpha numeric  | false
FromIp       | Text   | Yes    | Yes    | Yes        | Keyword        | false
ToIp         | Text   | Yes    | Yes    | Yes        | Keyword        | false
Application  | Text   | Yes    | Yes    | Yes        | Alpha numeric  | false

We are creating fields in our index to match the fields we have ingested to provide a place for the data to go that Stroom can reference.

{{< image "quick-start-guide/index/007_index_field_list.png" >}}Index field list{{< /image >}}

When you've done that, save the new index.

Now create a new XSLT.  We are going to convert xml data into something indexable by Stroom.

{{< image "quick-start-guide/index/008_xslt_new.png" >}}New XSLT{{< /image >}}

To make things manageable we create our new XSLT with the same name as the index in the same folder. After you've set the name just save it and close it, we’ll add some code in there later.

{{< image "quick-start-guide/index/009_xslt_name.png" >}}XSLT name{{< /image >}}

{{< image "quick-start-guide/index/010_xslt_settings.png" >}}XSLT settings{{< /image >}}

Now we get to send data to the index

Create a new pipeline called Indexing (we are going to make this a template for future indexing requirements).

{{< image "quick-start-guide/index/011_pipeline_new.png" >}}New pipeline{{< /image >}}

Edit the structure of the pipeline

Add the following element types with the specified names

Type                |Name
----                |----
XMLParser           |parser
SplitFilter         |splitFilter
IdEnrichmentFilter  |idEnrichmentFilter
XSLTFilter          |xsltFilter
IndexingFilter      |indexingFilter

So it looks like this (excluding the ReadRecordCountFilter and WriteRecordCountFilter elements)

{{< image "quick-start-guide/index/012_indexing_pipeline.png" >}}Indexing pipeline{{< /image >}}

Once the elements have been added you need to set the following property on the elements:

Element                 |Property   |Value
-------                 |--------   |-----
splitFilter             |splitCount |100

To do this we select the element then double click the property value in the property panel which is below it.

{{< image "quick-start-guide/index/013_pipeline_element_property.png" >}}Pipeline element property{{< /image >}}

The dialogue pops up where you can set the values

{{< image "quick-start-guide/index/014_pipeline_edit_element_property.png" >}}Pipeline edit element property{{< /image >}}

Save the pipeline, using the top left icon {{< stroom-icon "save.svg" "Save" >}}, then close the pipeline tab.

Now create a new pipeline

{{< image "quick-start-guide/index/015_pipeline_name.png" >}}Pipeline name{{< /image >}}

Which we will base on our new “Indexing” template pipeline

On our structure tab

{{< image "quick-start-guide/index/016_pipeline_structure_new.png" >}}Pipeline structure{{< /image >}}

Click in the “Inherit From” window

{{< image "quick-start-guide/index/017_pipeline_inheritance.png" >}}Pipeline inheritance{{< /image >}}

Select our Indexing pipeline template that we just created

{{< image "quick-start-guide/index/018_pipeline_inheritance_selection.png" >}}Pipeline inheritance selection{{< /image >}}

Now we need to set the XSLT property on the `xsltFilter` to point at the XSLT we created earlier and set the index on the `indexFilter` to point to the index we created.
This will appear as below (excluding the ReadRecordCountFilter and WriteRecordCountFilter elements)

{{< image "quick-start-guide/index/019_pipeline_properties_xslt.png" >}}Pipeline XSLT properties{{< /image >}}

Once that's done you can save your new pipeline

Next we need to create an XSLT that the `IndexingFilter` understands.

Open the feed we created in the quick-start guide if you find some processed data in your feed - i.e. browse the data

{{< image "quick-start-guide/index/020_data_browsing.png" >}}Data browsing{{< /image >}}

Click the stepping button {{< stroom-icon "stepping.svg" "Stepping" >}}

Select your new pipeline

{{< image "quick-start-guide/index/022_stepping_choose_pipeline.png" >}}Stepping choose pipeline{{< /image >}}

Paste the following into your `xsltFilter`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<xsl:stylesheet
  xmlns="records:2" xmlns:stroom="stroom"
  xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  version="2.0">
  <xsl:template match="/Events">
    <records xsi:schemaLocation="records:2 file://records-v2.0.xsd"
      version="2.0">
      <xsl:apply-templates />
    </records>
  </xsl:template>
  <xsl:template match="Event">
    <record>
      <data name="StreamId">
        <xsl:attribute name="value" select="@StreamId" />
      </data>
      <data name="EventId">
        <xsl:attribute name="value" select="@EventId" />
      </data>
      <xsl:apply-templates select="*" />
    </record>
  </xsl:template>
  <!-- Index the Id -->
  <xsl:template match="Id">
        <data name="Id">
          <xsl:attribute name="value" select="text()" />
        </data>
  </xsl:template>
  <!-- Index the Guid -->
  <xsl:template match="Guid">
    <data name="Guid">
      <xsl:attribute name="value" select="text()" />
    </data>
  </xsl:template>
  <!-- Index the FromIp -->
  <xsl:template match="FromIp">
    <data name="FromIp">
      <xsl:attribute name="value" select="text()" />
    </data>
  </xsl:template>
  <!-- Index the ToIp -->
  <xsl:template match="ToIp">
    <data name="ToIp">
      <xsl:attribute name="value" select="text()" />
    </data>
  </xsl:template>
  <!-- Index the Application -->
  <xsl:template match="Application">
    <data name="Application">
      <xsl:attribute name="value" select="text()" />
    </data>
  </xsl:template>
</xsl:stylesheet>
```

Which should look like this

{{< image "quick-start-guide/index/023_stepping_edit_xslt.png" >}}Stepping edit XSLT{{< /image >}}

What we are trying to do is turn the data into Stroom `record` format. This is basically name value pairs that we pass to the index. Step through the data using the top right arrows to ensure the XSLT produces correct output.

We're nearly there for indexing the data - you just need to tell the pipeline to pick up all processed data and index it.

Go back to your pipeline and go to the processors tab.

{{< image "quick-start-guide/index/024_processors_list.png" >}}Processors{{< /image >}}

Add a filter using {{< stroom-icon "add.svg" "Add" >}} and tell it to process all `Events` data when the filter dialogue opens so it looks like this

{{< image "quick-start-guide/index/025_stream_filter_edit.png" >}}Edit stream filter{{< /image >}}

Enable the processor and the filter by clicking the enabled tick boxes

{{< image "quick-start-guide/index/026_processors_enable.png" >}}Enable processors{{< /image >}}

Stroom should then index the data, assuming everything is correct

If there are errors you'll see error streams produced in the data browsing page, i.e. where you would normally see your processed and raw data.  If no errors have occurred, there will be no rows in the data page.

If it all goes to plan you'll see index shards appear if you open the index you created and click the shards tab.

{{< image "quick-start-guide/index/027_index_shard_list.png" >}}Index shards{{< /image >}}

The document count doesn't update immediately so don't worry if the count is 0. The count is updated on shard flush and happens in the background.

Now that we have finished indexing we can display data on a [dashboard](../dashboard/dashboard.md).