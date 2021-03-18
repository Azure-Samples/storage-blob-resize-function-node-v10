---
page_type: sample
languages:
- javascript
- nodejs
products:
- azure-storage
- azure
description: "This sample implements a function triggered by Azure Blob Storage to resize an image in Node.js."
---

# Azure Storage Blob Trigger Image Resize Function in Node.js using the v10 SDK

> IMPORTANT! This sample uses the version 10 of the Storage Blob SDK. Please refer to the sample [storage-blob-resize-function](https://github.com/Azure/azure-sdk-for-js/tree/master/sdk/storage/storage-blob/samples/storage-blob-resize-function) that uses the latest version of the Storage Blob SDK instead.

This sample implements a function triggered by Azure Blob Storage to resize an image in Node.js. Once the image is resized, the thumbnail image is uploaded back to blob storage.

The key aspects of this sample are in the function bindings and implementation.

This sample is used by the topic [Tutorial: Automate resizing uploaded images using Event Grid](https://docs.microsoft.com/en-us/azure/event-grid/resize-images-on-storage-blob-upload-event?tabs=nodejsv10#deploy-the-function-code/)

## Function bindings

In order to interface with image data, you need to configure the function to process binary data.

The following code sets the `datatype` parameter to `binary` in the `function.json` file.

```javascript
{
  "disabled": false,
  "bindings": [
    {
      "type": "eventGridTrigger",
      "name": "eventGridEvent",
      "direction": "in"
    },
    {
      "type": "blob",
      "name": "inputBlob",
      "path": "{data.url}",
      "connection": "AZURE_STORAGE_CONNECTION_STRING",
      "direction": "in",
      "datatype": "binary"
    }
  ]
}
```

## Function implementation

The sample uses [Jimp](https://github.com/oliver-moran/jimp) to resize an incoming buffer to a thumbnail. The buffer is then converted to a stream and uploaded to Azure Storage.

```javascript
const stream = require('stream');
const Jimp = require('jimp');

const {
  Aborter,
  BlobURL,
  BlockBlobURL,
  ContainerURL,
  ServiceURL,
  SharedKeyCredential,
  StorageURL,
  uploadStreamToBlockBlob
} = require("@azure/storage-blob");

const ONE_MEGABYTE = 1024 * 1024;
const ONE_MINUTE = 60 * 1000;
const uploadOptions = { bufferSize: 4 * ONE_MEGABYTE, maxBuffers: 20 };

const containerName = process.env.BLOB_CONTAINER_NAME;
const accountName = process.env.AZURE_STORAGE_ACCOUNT_NAME;
const accessKey = process.env.AZURE_STORAGE_ACCOUNT_ACCESS_KEY;

const sharedKeyCredential = new SharedKeyCredential(
  accountName,
  accessKey);
const pipeline = StorageURL.newPipeline(sharedKeyCredential);
const serviceURL = new ServiceURL(
  `https://${accountName}.blob.core.windows.net`,
  pipeline
);

module.exports = (context, eventGridEvent, inputBlob) => {  

  const aborter = Aborter.timeout(30 * ONE_MINUTE);
  const widthInPixels = 100;
  const contentType = context.bindingData.data.contentType;
  const blobUrl = context.bindingData.data.url;
  const blobName = blobUrl.slice(blobUrl.lastIndexOf("/")+1);

  const image = await Jimp.read(inputBlob);
  const thumbnail = image.resize(widthInPixels, Jimp.AUTO);
  const thumbnailBuffer = await thumbnail.getBufferAsync(Jimp.AUTO);
  const readStream = stream.PassThrough();
  readStream.end(thumbnailBuffer);

  const containerURL = ContainerURL.fromServiceURL(serviceURL, containerName);
  const blockBlobURL = BlockBlobURL.fromContainerURL(containerURL, blobName);
  try {

    await uploadStreamToBlockBlob(aborter, readStream,
      blockBlobURL, uploadOptions.bufferSize, uploadOptions.maxBuffers,
      { blobHTTPHeaders: { blobContentType: "image/*" } });

  } catch (err) {

    context.log(err.message);

  } finally {

    context.done();

  }
};
```

You can use the [Azure Storage Explorer](https://azure.microsoft.com/features/storage-explorer/) to view blob containers and verify the resize is successful.
