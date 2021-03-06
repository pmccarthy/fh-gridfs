fh-gridfs
=======

This is a wrapper module for grid-fs to provide version control for files.

This module makes use of streams to ensure whole files do not need to be loaded into memory to save into a mongo database. Files can be grouped together to create a versioned file system.

The "MongoFileHandler" module can be initialised with the following parameters

    --> config (Optional, if not specified, defaults will be used).
        --> THUMBNAIL_PREPEND --> The name of the prefix for thumbnail images. (default: "thumbnail_")
        --> DEFAULT_THUMBNAIL_WIDTH --> The default thumbnail width if not specified when saving an image.
        --> DEFAULT_THUMBNAIL_HEIGHT --> The default thumbnail height if not specified when saving an image.
        --> MAX_MONGO_QUEUE_LENGTH --> The maximum array length for caching streams to the mongo database before pausing the stream. This variable can be used to tune the amount of memory streaming to the database consumes. (default: 2000).
        --> ROOT_COLLECTION --> The name of the collections storing the files. (default: "fileStorage").
        --> LOG_LEVEL --> The level of logging for the fileHandler. (See logger parameter).
    --> logger --> Any object capable of logging. (Optional, if not specified, a default logger will be used with logging levels : LOGLEVEL_ERROR = 0, LOGLEVEL_WARNING = 1, LOGLEVEL_INFO = 2, LOGLEVEL_DEBUG = 3)

Methods

saveFile(db, fileName, fileReadStream, options, callback(err, fileDetails))

    -- db -> Database to stream the file to.
    -- fileName --> The name of the file (including file extension)
    -- fileReadStream --> Any readable stream (e.g. file stream, request stream etc)
    -- options -->
            --> "thumbnail": {"width": thumbnailWidth, "height":thumbnailHeight} . If the file is an image and you want to create a thumbnail for the image, supply an option called. (Optional: if left out, no thumbnail file will be created).
            --> "groupId", the identifier for the group that the file belongs to. (Optional: If "groupId" is not specified, the file will be created with a groupId equal to the fileId.)
    -- callback --> The callback executed after the file has been streamed to the database. The response will contain an "error" object if there was an error with saving. If there was no error, the "fileDetails" parameter will contain
            --> "fileName" --> The name of the file including file extension.
            --> "version" --> The version of the file grouped by groupId.
            --> "hash" --> The md5 hash of the file contents.
            --> "groupId" --> The id of the group the file belongs to.
            --> "contentType" --> The mime type of the file (e.g. application/pdf).
            --> "thumbnailHash" --> Tf a thumbnail was created, the hash of the thumbnail file created as part of saving the file will be included.
            --> "thumbnailFileName" --> The name of the thumbnail file created.

deleteFile(db, searchParameters, callback(err))

    -- db --> Database to delete the file from
    -- searchParameters --> The parameters to search the file by. An object with the following fields
            --> groupId --> The id of the group the file belongs to.
            --> version --> The version of the file in the groupId to delete. (Optional: if left out, the latest file will be deleted).
    -- callback --> The callback executed after the file has been deleted. The response will contain an "error" object if there was an error deleting the file.

deleteAllFileVersions(db, groupId, callback(err))

    -- db -> Database to delete the file from
    -- groupId --> The id of the group to remove.
    -- callback --> The callback executed after the file has been deleted. The response will contain an "error" object if there was an error deleting the file.

migrateFiles(dbFrom, dbTo, arrayOfGroupsToMigrate, callback(err, migrationDetails))

    -- dbFrom --> The database containing the files to migrate.
    -- dbTo --> The database to migrate the files to.
    -- arrayOfGroupsToMigrate --> This is an array of objects containing the following parameters
            --> groupId --> The id of the group to migrate (Note, it is not possible to migrate more than one file from the same groupId).
            --> version --> The version of the file in the group to migrate. (Optional: if left out, the latest file will be migrated).
    -- callback --> The callback will contain an error message if there was an error migrating the files. If successful, the migrationDetails parameter will contain an object indexed by the groupId. Each element in the object will contain the following elements:
            --> "fileName" --> The name of the file including file extension.
            --> "version" --> The version of the file grouped by groupId. (Note: this is the version of the file in the database migrated to. May be different from version in database migrated from.)
            --> "hash" --> The md5 hash of the file contents.
            --> "groupId" --> The id of the group the file belongs to.
            --> "contentType" --> The mime type of the file (e.g. application/pdf).
            --> "thumbnailHash" --> If a thumbnail exists for this file, the hash of the thumbnail file migrated as part of migrating the file will be included.
            --> "thumbnailFileName" --> The name of the thumbnail file migrated.

getFileHistory(db, groupId, callback(err, fileHistory))

    -- db --> Database containing the fileGroup
    -- groupId --> The id of the file group.
    -- callback --> callback after getFileHistory is executed. If an error occurred, the first parameter will contain the error. If successful, the second parameter will contain an array of objects in decending order based on the version (most recent first), containing the following fields
            --> "fileName" --> The name of the file including file extension.
            --> "version" --> The version of the file grouped by groupId.
            --> "hash" --> The md5 hash of the file contents.
            --> "groupId" --> The id of the group the file belongs to.
            --> "contentType" --> The mime type of the file (e.g. application/pdf).


streamFile(db, searchOptions, fileOptions, callback(err, fileStream)

    -- db --> The database to stream the file from.
    -- searchOptions --> The options for how to search for a file. This object can contain the following fields
            --> groupId --> The id of the group the file belongs to. (Optional, if searching by hash value, this field does not need to be included).
            --> version --> The version of the file in the group. (Optional, if left out, the latest file in the group will be downloaded).
            --> hash --> The hash value of the file to stream. (Optional, if searching by groupId, this option does not need to be included.)
    -- fileOptions --> An object for the options for a single file. Contains the following fields
            --> thumbnail --> If searching by groupId and version, the thumbnail parameter will get the thumbnail of the file if it is an image and a thumbnail exists for the file.
    -- callback --> The callback contains an error object if there was an error getting the stream. If the function was successful, the fileStream object will contain a readable stream that can be piped to any writable stream.

exportFilesStream(db, exportEntries, zipStream, cb)

    -- db --> The database to stream the file from.
    -- exportEntries --> An array of objects containing the following fields
        -- groupId --> The id of the group to export.
        -- version : The version of the file contained in the group. (Optional: if left out, the latest file in the group is added.).
    -- zipStream --> An "archiver" zip object. Archiver is a node module for zipping files. Create an archiver object, add any files needed, pass it to the exportFilesStream and then finalize it. All of the readStreams will be added to the zipStream.
    -- callback --> The callback returns an error if there is an error adding the file to the zipStream.

## Testing

Before running the grunt fh:test command, make sure you have a local instance (port 27017) of mongdb running.

Open a mongo shell and execute the following
```bash
use admin
db.addUser('admin','admin'); # for version 2.4.6
db.createUser({ user: "admin",pwd: "admin",roles: [ { role: "clusterAdmin", db: "admin" }, { role: "readWriteAnyDatabase", db: "admin" },"readWrite"] }, { w: "majority" , wtimeout: 25000 }); # for versions 2.6.4 or greater
exit
```

## License

fh-gridfs is licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/).
