---
title: How to create a dataset using a GeoJson package
description: Learn how to create a dataset using a GeoJson package.
author: stevemunk
ms.author: v-munksteve
ms.date: 11/01/2021
ms.topic: how-to
ms.service: azure-maps
services: azure-maps
---

# Create a dataset using a GeoJson package (Preview)

Azure Maps Creator enables users to import their indoor map data in GeoJSON format with [Facility Ontology 2.0][Facility Ontology], which can then be used to create a [dataset][dataset-concept].

> [!NOTE]
> This article explains how to create a dataset from a GeoJSON package. For information on additional steps required to complete an indoor map, see [Next steps](#next-steps).

## Prerequisites

- Basic understanding of [Creator for indoor maps](creator-indoor-maps.md).
- Basic understanding of [Facility Ontology 2.0][Facility Ontology].
- [Azure Maps account][Azure Maps account].
- [Azure Maps Creator resource][Creator resource].
- [Subscription key][Subscription key].
- Zip package containing all required GeoJSON files. If you don't have GeoJSON
  files, you can download the [Contoso building sample][Contoso building sample].

>[!IMPORTANT]
>
> - This article uses the `us.atlas.microsoft.com` geographical URL. If your Creator service wasn't created in the United States, you must use a different geographical URL.  For more information, see [Access to Creator Services](how-to-manage-creator.md#access-to-creator-services).
> - In the URL examples in this article you will need to replace `{Azure-Maps-Primary-Subscription-key}` with your primary subscription key.

## Create dataset using the GeoJSON package

For more information on the GeoJSON package, see the [Geojson zip package requirements](#geojson-zip-package-requirements) section.

### Upload the GeoJSON package

Use the [Data Upload API](/rest/api/maps/data-v2/upload) to upload the Drawing package to Azure Maps Creator account.

The Data Upload API is a long running transaction that implements the pattern defined in [Creator Long-Running Operation API V2](creator-long-running-operation-v2.md).

To upload the GeoJSON package:

1. Execute the following HTTP POST request that uses the [Data Upload API](/rest/api/maps/data-v2/upload):

    ```http
    https://us.atlas.microsoft.com/mapData?api-version=2.0&dataFormat=zip&subscription-key={Your-Azure-Maps-Primary-Subscription-key}
    ```

    1. Set `Content-Type` in the **Header** to `application/zip`.

1. Copy the value of the `Operation-Location` key in the response header. The `Operation-Location` key is also known as the `status URL` and is required to check the status of the upload, which is explained in the next section.

### Check the GeoJSON package upload status

To check the status of the GeoJSON package and retrieve its unique identifier (`udid`):

1. Execute the following HTTP GET request that uses the status URL you copied as the last step in the previous section of this article. The request should look like the following URL:

```http
https://us.atlas.microsoft.com/mapData/operations/{operationId}?api-version=2.0&subscription-key={Your-Azure-Maps-Primary-Subscription-key}
```

1. Copy the value of the `Resource-Location` key in the response header, which is the `resource location URL`. The `resource location URL` contains the unique identifier (`udid`) of the GeoJSON package resource.

### Create a dataset

A dataset is a collection of map features, such as buildings, levels, and rooms. To create a dataset from your GeoJSON, use the new [Dataset Create API][Dataset Create 2022-09-01-preview]. The Dataset Create API takes the `udid` you got in the previous section and returns the `datasetId` of the new dataset.

> [!IMPORTANT]
> This is different from the [previous version][Dataset Create] in that it doesn't require a `conversionId` from a converted Drawing package.

To create a dataset:

1. Enter the following URL to the dataset service. The request should look like the following URL (replace {udid} with the `udid` obtained in [Check the GeoJSON package upload status](#check-the-geojson-package-upload-status) section):

<!--1. Enter the following URL to the [Dataset service][Dataset Create 2022-09-01-preview]. The request should look like the following URL (replace {udid} with the `udid` obtained in [Check the GeoJSON package upload status](#check-the-geojson-package-upload-status) section):-->

  ```http
  https://us.atlas.microsoft.com/datasets?api-version=2022-09-01-preview&udid={udid}&subscription-key={Your-Azure-Maps-Primary-Subscription-key}
  ```

1. Copy the value of the `Operation-Location` key in the response header. The `Operation-Location` key is also known as the `status URL` and is required to check the status of the dataset creation process and to get the `datasetId`, which is required to create a tileset.

### Check the dataset creation status

To check the status of the dataset creation process and retrieve the `datasetId`:

1. Enter the status URL you copied in [Create a dataset](#create-a-dataset). The request should look like the following URL:

    ```http
    https://us.atlas.microsoft.com/datasets/operations/{operationId}?api-version=2022-09-01-preview&subscription-key={Your-Azure-Maps-Primary-Subscription-key}
    ```

1. In the Header of the HTTP response, copy the value of the unique identifier contained in the `Resource-Location` key.

    > `https://us.atlas.microsoft.com/datasets/**c9c15957-646c-13f2-611a-1ea7adc75174**?api-version=2022-09-01-preview`

See [Next steps](#next-steps) for links to articles to help you complete your indoor map.

## Add data to an existing dataset

Data can be added to an existing dataset by providing the `datasetId` parameter to the [dataset create API][Dataset Create 2022-09-01-preview] along with the unique identifier of the data you wish to add. The unique identifier can be either a `udid` or `conversionId`. This creates a new dataset consisting of the data (facilities) from both the existing dataset and the new data being imported. Once the new dataset has been created successfully, the old dataset can be deleted.

One thing to consider when adding to an existing dataset is how the feature IDs are created. If a dataset is created from a converted drawing package, the feature IDs are generated automatically. When a dataset is created from a GeoJSON package, feature IDs must be provided in the GeoJSON file. When appending to an existing dataset, the original dataset drives the way feature IDs are created. If the original dataset was created using a `udid`, it uses the IDs from the GeoJSON, and will continue to do so with all GeoJSON packages appended to that dataset in the future.  If the dataset was created using a `conversionId`, IDs will be internally generated, and will continue to be internally generated with all GeoJSON packages appended to that dataset in the future.

### Add to dataset created from a GeoJSON source

If your original dataset was created from a GoeJSON source and you wish to add another facility created from a drawing package, you can append it to your existing dataset by referencing its `conversionId`, as demonstrated by this HTTP POST request:

```http
https://us.atlas.microsoft.com/datasets?api-version=2022-09-01-preview&conversionId={conversionId}&outputOntology=facility-2.0&datasetId={datasetId}
```

| Identifier   | Description                                                       |
|--------------|-------------------------------------------------------------------|
| conversionId | The ID returned when converting your drawing package. For more information, see [Convert a Drawing package][conversion]. |
| datasetId    | The dataset ID returned when creating the original dataset from a GeoJSON package). |

## Geojson zip package requirements

The GeoJSON zip package consists of one or more [RFC 7946][RFC 7946] compliant GeoJSON files, one for each feature class, all in the root directory (subdirectories aren't supported), compressed with standard Zip compression and named using the `.ZIP` extension.

Each feature class file must match its definition in the [Facility ontology 2.0][Facility ontology] and each feature must have a globally unique identifier.

Feature IDs can only contain alpha-numeric (a-z, A-Z, 0-9), hyphen (-), dot (.) and underscore (_) characters.

> [!TIP]
> If you want to be certain you have a globally unique identifier (GUID), consider creating it by running a GUID generating tool such as the Guidgen.exe command line program (Available with [Visual Studio][Visual Studio]). Guidgen.exe never produces the same number twice, no matter how many times it is run or how many different machines it runs on.

### Facility ontology 2.0 validations in the Dataset

[Facility ontology][Facility ontology] defines how Azure Maps Creator internally stores facility data, divided into feature classes, in a Creator dataset. When importing a GeoJSON package, anytime a feature is added or modified, a series of validations run. This includes referential integrity checks as well as geometry and attribute validations. These validations are described in more detail below.

- The maximum number of features that can be imported into a dataset at a time is 150,000.
- The facility area can be between 4 and 4,000 Sq Km.
- The top level element is [facility][facility], which defines each building in the file *facility.geojson*.
- Each facility has one or more levels, which are defined in the file *levels.goejson*.
  - Each level must be inside the facility.
- Each [level][level] contain [units][unit], [structures][structure], [verticalPenetrations][verticalPenetration] and [openings][opening]. All of the items defined in the level must be fully contained within the Level geometry.
  - `unit` can consist of an array of items such as hallways, offices and courtyards, which are defined by [area][areaElement], [line][lineElement] or [point][pointElement] elements. Units are defined in the file *unit.goejson*.
    - All `unit` elements must be fully contained within their level and intersect with their children.
  - `structure` defines physical, non-overlapping areas that can't be navigated through, such as a wall. Structures are defined in the file *structure.goejson*.
  - `verticalPenetration` represents a method of navigating vertically between levels, such as stairs and elevators and are defined in the file *verticalPenetration.geojson*.
    - verticalPenetrations can't intersect with other verticalPenetrations on the same level.
  - `openings` define traversable boundaries between two units, or a `unit` and `verticalPenetration` and are defined in the file *opening.geojson*.
    - Openings can't intersect with other openings on the same level.
    - Every `opening` must be associated with at least one `verticalPenetration` or `unit`.

## Next steps

> [!div class="nextstepaction"]
> [Create a tileset](tutorial-creator-indoor-maps.md#create-a-tileset)

> [!div class="nextstepaction"]
> [Query datasets with WFS API](tutorial-creator-wfs.md)

> [!div class="nextstepaction"]
> [Create a feature stateset](tutorial-creator-feature-stateset.md)

[Contoso building sample]: https://github.com/Azure-Samples/am-creator-indoor-data-examples
[unit]: creator-facility-ontology.md?pivots=facility-ontology-v2#unit
[structure]: creator-facility-ontology.md?pivots=facility-ontology-v2#structure
[level]: creator-facility-ontology.md?pivots=facility-ontology-v2#level
[facility]: creator-facility-ontology.md?pivots=facility-ontology-v2#facility
[verticalPenetration]: creator-facility-ontology.md?pivots=facility-ontology-v2#verticalpenetration
[opening]: creator-facility-ontology.md?pivots=facility-ontology-v2#opening
[areaElement]: creator-facility-ontology.md?pivots=facility-ontology-v2#areaelement
[lineElement]: creator-facility-ontology.md?pivots=facility-ontology-v2#lineelement
[pointElement]: creator-facility-ontology.md?pivots=facility-ontology-v2#pointelement

[conversion]: tutorial-creator-indoor-maps.md#convert-a-drawing-package
[Azure Maps account]: quick-demo-map-app.md#create-an-azure-maps-account
[Creator resource]: how-to-manage-creator.md
[Subscription key]: quick-demo-map-app.md#get-the-primary-key-for-your-account
[Facility Ontology]: creator-facility-ontology.md?pivots=facility-ontology-v2
[RFC 7946]: https://www.rfc-editor.org/rfc/rfc7946.html
[dataset-concept]: creator-indoor-maps.md#datasets
[Dataset Create 2022-09-01-preview]: /rest/api/maps/v20220901preview/dataset/create
[Visual Studio]: https://visualstudio.microsoft.com/downloads/
