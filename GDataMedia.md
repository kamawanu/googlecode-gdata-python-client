# Introduction #

Some GData APIs, such as Picasa Web, allow binary and multipart POSTs and PUTs.  The Python client library now provides methods that can be used for these operations.


# Details #

There are a few media-related operations:  Create, Update, and Read.  Deletion is handled in the same way as any other GData feed.

## Creating a Media Entry ##

There are two ways to create a media entry -- with and without metadata.

To create a new media entry without metadata, first create a MediaSource class and populate it with the source of your media:

```
ms = gdata.MediaSource()
ms.setFile('testimage.jpg', 'image/jpeg')
```

Then, POST this MediaSource class to the feed:

```
media_entry = gd_client.Post(None, albumFeedURL, media_source = ms)
```

Note that the 'data' field of the Post method is None, which indicates that we are not sending metadata along with the request.  To send metadata along with the request, create an entry of the appropriate type, and include it in the Post method call:

```
new_media_entry = gdata.GDataEntry()
new_media_entry.title = atom.Title(text='testimage1.jpg')
new_media_entry.summary = atom.Summary(text='Test Image')
new_media_entry.category.append(atom.Category(scheme = 'http://schemas.google.com/g/2005#kind', term = 'http://schemas.google.com/photos/2007#photo'))
media_entry = gd_client.Post(new_media_entry, albumFeedURL, media_source = ms)
```

The entry returned is a GData Media entry containing links to the created media.

## Updating a Media Entry ##

There are two parts of a Media entry that can be updated: the media itself, and its metadata.  These operations are done in the same way as shown above, with the operation changed to Put.

To update only the media itself, first create a MediaSource containing the new media and use the Put method to update the entry.  Note that the "edit-media" link should be used as the URI:

```
ms = gdata.MediaSource()
ms.setFile('testimage.jpg', 'image/jpeg')
updated_entry = gd_client.Put(None, media_entry.GetEditMediaLink().href, media_source = ms)
```

To update both the metadata and the media at once, include the entry in the Put method call.  Note that the "edit" link should be used as the URI:

```
ms = gdata.MediaSource()
ms.setFile('testimage.jpg', 'image/jpeg')
media_entry.summary = atom.Summary(text='Updated Test Image')
media_entry2 = gd_client.Put(media_entry, media_entry.GetEditLink().href, media_source = ms)
```

## Reading Media ##

The media data itself is not included in the entries returned by Google data feeds.  Instead, a reference to the media is included in the entry.  You can retrieve this reference using the GetMediaURL method:

```
media_url = media_entry.GetMediaURL()
```

To get a MediaSource containing a handle to the media along with the length and content type, use the GetMedia method on a GData Service:

```
imageSource = gd_client.GetMedia(media_entry2.GetMediaURL())
```

## Deleting Media ##

Deleting a Media entry is accomplished in the same way as any other GData entry.  Either the "edit" or "edit-media" links can be used, and both the entry and the media will be deleted.

```
gd_client.Delete(media_entry.GetEditLink().href)
```