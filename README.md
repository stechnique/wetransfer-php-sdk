# WeTransfer PHP SDK

[![Build Status](https://travis-ci.org/arkaitzgarro/wetransfer-php-sdk.svg?branch=master)](https://travis-ci.org/arkaitzgarro/wetransfer-php-sdk)
[![Latest Stable Version](https://poser.pugx.org/wetransfer/php-sdk/v/stable)](https://packagist.org/packages/wetransfer/php-sdk)
[![License](https://poser.pugx.org/wetransfer/php-sdk/license)](https://packagist.org/packages/wetransfer/php-sdk)
[![Coverage Status](https://coveralls.io/repos/github/arkaitzgarro/wetransfer-php-sdk/badge.svg?branch=master)](https://coveralls.io/github/arkaitzgarro/wetransfer-php-sdk?branch=master)

A PHP SDK for WeTransfer's Public API

## Installation

System Requirements
- PHP 5.6.4 or greater
- Composer

The WeTransfer PHP SDK can be installed through Composer.

```bash
$ php composer require arkaitzgarro/wetransfer-php-sdk
```

## Usage

In order to be able to use the SDK and access our public APIs, you must provide an API key, which is available in our [Developers Portal](https://developers.wetransfer.com/).

You can find a complete working example [here](https://github.com/arkaitzgarro/wetransfer-php-sdk/blob/master/example/CreateTransfer.php).

Firstly, the client needs to be configured with your API Key obtained from the WeTransfer's Developer.

```php
$wtClient = WeTransfer\Client::setApiKey(getenv['WT_API_KEY']);
```

### Transfer

Transfers can be created with or without items. Once the transfer has been created, items can be added at any time:

```php
$transfer = WeTransfer\Transfer::create('My Transfer', 'And optional description');
```

### Add items to a transfer

Once a transfer has been created you can then add items (files or links) to it. If you are adding files to the transfer, the files are not uploaded at this point, but in the next step.

```php
WeTransfer\Transfer::addLinks($transfer, [
  [
    'url' => 'https://en.wikipedia.org/wiki/Japan',
    'meta' => [
      'title' => 'Japan'
    ]
  ]
]);

WeTransfer\Transfer::addFiles($transfer, [
  [
    'filename' => 'Japan-01.jpg',
    'filesize' => 13370099
  ]
]);
```

The `$transfer` object will be updated with each item that was added to the transfer. For files, this objects will be used to upload the correspondent file to the transfer, as explained in the next section.

### Upload a file

Once the file has been added to the transfer, next step is to upload the file or files. You must provide the content of the file to upload as a reference (use `fopen` function for it), we will NOT read the file for you. The content of the file will be splited and uploaded in chunks of 5MB to our S3 bucket.

```php
foreach($transfer->getFiles() as $file) {
  WeTransfer\File::upload($file, fopen(realpath('./path/to/your/files.jpg'), 'r'));
}
```

### Request an upload URL

The previous steps work well for an environment where accessing the files directly is possible, like a CLI tool. In a web environment, we don't want to upload the files to the server, and from there, upload them to S3, but upload them directly from the client. `WeTransfer\File::createUploadUrl` method will create the necessary upload URL for a given part.

```js
// This code lives on the browser
async function uploadFile(item, content) {
  const MAX_CHUNK_SIZE = 6 * 1024 * 1024;
  for (let partNumber = 0; partNumber < item.meta.multipart_parts; partNumber++) {
    const chunkStart = partNumber * MAX_CHUNK_SIZE;
    const chunkEnd = (partNumber + 1) * MAX_CHUNK_SIZE;

    const multipartItem = await fetch('https://yourserver.com/create-upload-url', {
      method: 'POST',
      body: JSON.stringify({
        file_id: item.id.
        multipart_upload_id: item.meta.multipart_upload_id,
        part_number: partNumber
      })
    });
    
    await fetch(multipartItem.upload_url, {
      method: 'PUT',
      body: content.slice(chunkStart, chunkEnd)
    });
  }
};
```

```php
// src/Controller/LuckyController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Annotation\Route;

class FilesController
{
    /**
     * @Route("/create-upload-url", name="app_create_upload_url", methods={"POST"})
     */
    public function createUploadUrl(Request $request)
    {
        $data = json_decode(
            $request->getContent(),
            true
        );

        $uploadUrl = WeTransfer\File::createUploadUrl(
            $data['file_id'],
            $data['multipart_upload_id'],
            $data['part_number']
        );

        return new JsonResponse(
            $uploadUrl,
            JsonResponse::HTTP_CREATED
        );
    }
}
```

## Development

Get Composer. Follow the instructions defined on the official [Composer page](https://getcomposer.org/doc/00-intro.md), or if you are using `homebrew`, just run:

```bash
$ brew install composer
```

Install project dependencies:

```bash
$ composer install
```

Run the test suite:

```bash
$ ./vendor/bin/phpunit
```

Please adhere to [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) coding standard. Run the following commands before pushing your code:

```bash
$ ./vendor/bin/phpcs --standard=PSR2 -n src tests

# This command will automatically fix existing issues
$ ./vendor/bin/phpcbf --standard=PSR2 -n src tests
```
