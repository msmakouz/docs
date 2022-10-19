# ODM Bridge
Use module `spiral/hybrid-db` in order to create link to ODM models. Such approach can be used to store heavy or unstructured data in a separate database.

> Currently only Document::ONE relation is supported.

## Installation
You can install ODM bridge using `spiral/hybrid-db` package:

`composer require spiral/hybrid-db`
`spiral register spiral/hybrid-db`

## Example
```php
class Photo extends Record
{
    const SCHEMA = [
        'id'          => 'primary',
        'filesize'    => 'int',
        'filename'    => 'string',
        'metadata'    => [Document::ONE => Metadata::class]
    ];
}
```

Where:
```php
class Metadata extends Document
{
    const SCHEMA = [
        '_id'      => 'MongoId',
        'photo_id' => 'int',
        'keywords' => ['string'],
    ];
}
```

> Note that you have to declare `photo_id` field in your Document manually due this models belongs to different domains.

You can use your photo model `metadata` field as for usual ORM relations:

```php
$photo = $this->orm->make(Photo::class);
$photo->filesize = 100;
$photo->filename = 'filename';
$photo->metadata = $this->odm->make(Metadata::class, [
    'keywords' => ['metadata', 'keyword']
]);
$photo->save();
```

## Inheritance
ODM relations fully support model inheritance:

```php
class IPTCMetadata extends Metadata
{
    const SCHEMA = [
        'iptc' => 'mixed'
    ];
}
```

```php
$photo = new Photo(); //Only in IoC scope
$photo->filesize = 100;
$photo->filename = 'filename';
$photo->metadata = new IPTCMetadata([
    'keywords' => ['metadata', 'keyword'],
    'iptc'     => [
        'some' => 'value'
    ]
]);
$photo->save();
```

## Loading
ODM relations support lazy-loading as well as eager loading, though you are not able to filter your records by data located in MongoDB database:

```php
$photo = $this->orm->source(Photo::class)->findByPK(1);
dump($photo->metadata);
```

Eager loading:

```php
$photos = $this->orm->source(Photo::class)->find()->load('metadata');

foreach($photos as $photo) {
    dump($photo->metadata);
}
```
