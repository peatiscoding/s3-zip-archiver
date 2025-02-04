# s3-zip-arch

S3-Zip with access to Archiver object

Originally forked from: [orangewise/s3-zip](https://github.com/orangewise/s3-zip)

[![npm version][npm-badge]][npm-url]
[![Build Status][travis-badge]][travis-url]
[![Coverage Status][coveralls-badge]][coveralls-url]
[![JavaScript Style Guide](https://img.shields.io/badge/code%20style-standard-brightgreen.svg)](http://standardjs.com/)

Download selected files from an Amazon S3 bucket as a zip file.

## Install

```
npm install s3-zip-arch
```


## AWS Configuration

Refer to the [AWS SDK][aws-sdk-url] for authenticating to AWS prior to using this plugin.



## Usage

### Zip specific files

```javascript

const fs = require('fs')
const join = require('path').join
const s3Zip = require('s3-zip-arch')

const region = 'bucket-region'
const bucket = 'name-of-s3-bucket'
const folder = 'name-of-bucket-folder/'
const file1 = 'Image A.png'
const file2 = 'Image B.png'
const file3 = 'Image C.png'
const file4 = 'Image D.png'

const output = fs.createWriteStream(join(__dirname, 'use-s3-zip-arch.zip'))

s3Zip
  .archive({ region: region, bucket: bucket}, folder, [file1, file2, file3, file4])
  .pipe(output)

```

You can also pass a custom S3 client. For example if you want to zip files from a S3 compatible storage:

```javascript
const { S3Client } = require('@aws-sdk/client-s3')

const S3Client = new aws.S3({
  signatureVersion: 'v4',
  s3ForcePathStyle: 'true',
  endpoint: 'http://localhost:9000',
})

s3Zip
  .archive({ s3: S3Client, bucket: bucket }, folder, [file1, file2])
  .pipe(output)
```

### Zip files with AWS Lambda

Example of s3-zip-arch in combination with [AWS Lambda](aws_lambda.md).


### Zip a whole bucket folder

```javascript
const fs = require('fs')
const join = require('path').join
const {
  S3Client
} = require("@aws-sdk/client-s3")
const s3Zip = require('s3-zip-arch')
const XmlStream = require('xml-stream')

const region = 'bucket-region'
const bucket = 'name-of-s3-bucket'
const folder = 'name-of-bucket-folder/'
const s3 = new S3Client({ region: region })
const params = {
  Bucket: bucket,
  Prefix: folder
}

const filesArray = []
const files = s3.listObjects(params).createReadStream()
const xml = new XmlStream(files)
xml.collect('Key')
xml.on('endElement: Key', function(item) {
  filesArray.push(item['$text'].substr(folder.length))
})

xml
  .on('end', function () {
    zip(filesArray)
  })

function zip(files) {
  console.log(files)
  const output = fs.createWriteStream(join(__dirname, 'use-s3-zip.zip'))
  s3Zip
   .archive({ region: region, bucket: bucket, preserveFolderStructure: true }, folder, files)
   .pipe(output)
}
```

### Tar format support

```javascript
s3Zip
  .setFormat('tar')
  .archive({ region: region, bucket: bucket }, folder, [file1, file2])
  .pipe(output)
```

### Zip a file with protected password

```javascript
s3Zip
  .setRegisterFormatOptions('zip-encrypted', require("archiver-zip-encrypted"))
  .setFormat('zip-encryptable')
  .setArchiverOptions({zlib: {level: 8}, encryptionMethod: 'aes256', password: '123'})
  .archive({ region: region, bucket: bucket }, folder, [file1, file2])
  .pipe(output)
```

### Archiver options

We use [archiver][archiver-url] to create archives. To pass your options to it, use `setArchiverOptions` method:

```javascript
s3Zip
  .setFormat('tar')
  .setArchiverOptions({ gzip: true })
  .archive({ region: region, bucket: bucket }, folder, [file1, file2])
```

### Organize your archive with custom paths and permissions

You can pass an array of objects with type [EntryData][entrydata-url] to organize your archive.

```javascript
const files = ['flower.jpg', 'road.jpg']
const archiveFiles = [
  { name: 'newFolder/flower.jpg' },

  /* _rw_______ */
  { name: 'road.jpg', mode: parseInt('0600', 8)  }
];
s3Zip.archive({ region: region, bucket: bucket }, folder, files, archiveFiles)
```

### Using with ExpressJS

`s3-zip-arch` works with any framework which leverages on NodeJS Streams including ExpressJS.

```javascript
const s3Zip = require('s3-zip-arch')

app.get('/download', (req, res) => {
  s3Zip
    .archive({ region: region, bucket: bucket }, '', 'abc.jpg')
    .pipe(res)
})
```
Above should stream out the file in the response of the request.

### Customize it with access to Archiver 

```javascript
const s3Zip = require('s3-zip-arch')

s3Zip
  .setOnArchciverEnd(async (archiver) => {
    // this method will call right before archiver finalize itself.
    // this method support both Promise and non-Promise
    const content = await computeManifestContent()
    archiver.append(content, { name: 'manifest.json' })
  })
  .archive({ region: region, bucket: bucket }, folder, [file1, file2])
```

### Debug mode

Enable debug mode to see the logs:

```javascript
s3Zip.archive({ region: region, bucket: bucket, debug: true }, folder, files)
```

## Testing

Tests are written in Node Tap, run them like this:

```
npm t
```

If you would like a more fancy coverage report:

```
npm run coverage
```

[aws-sdk-url]: https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/configuring-the-jssdk.html
[npm-badge]: https://badge.fury.io/js/s3-zip-arch.svg
[npm-url]: https://badge.fury.io/js/s3-zip-arch
[archiver-url]: https://www.npmjs.com/package/archiver
[entrydata-url]: https://archiverjs.com/docs/global.html#EntryData
