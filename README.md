[![NPM](https://nodei.co/npm/filer.png?downloads=true&stars=true)](https://nodei.co/npm/filer/)

###Filer

Filer is a POSIX-like file system interface for browser-based JavaScript.

### Contributing

Want to join the fun? We'd love to have you! See [CONTRIBUTING](CONTRIBUTING.md).

###Downloading

Pre-built versions of the library are available in the repo:

* [filer.js](https://raw.github.com/js-platform/filer/develop/dist/filer.js)
* [filer.min.js](https://raw.github.com/js-platform/filer/develop/dist/filer.min.js)

### Getting Started

Filer is as close to the node.js [fs module](http://nodejs.org/api/fs.html) as possible,
with the following differences:

* No synchronous versions of methods (e.g., `mkdir()` but not `mkdirSync()`).
* No permissions (e.g., no `chown()`, `chmod()`, etc.).
* No support (yet) for `fs.watchFile()`, `fs.unwatchFile()`, `fs.watch()`.
* No support for stream-based operations (e.g., `fs.ReadStream`, `fs.WriteStream`).

Filer has other features lacking in node.js (e.g., swappable backend
storage providers, support for encryption and compression, extended attributes, etc).

Like node.js, the API is asynchronous and most methods expect the caller to provide
a callback function (note: like node.js, Filer will supply one if it's missing).
Errors are passed to callbacks through the first parameter.  As with node.js,
there is no guarantee that file system operations will be executed in the order
they are invoked. Ensure proper ordering by chaining operations in callbacks.

### Example

To create a new file system or open an existing one, create a new `FileSystem`
instance.  By default, a new [IndexedDB](https://developer.mozilla.org/en/docs/IndexedDB)
database is created for each file system. The file system can also use other
backend storage providers, for example [WebSQL](http://en.wikipedia.org/wiki/Web_SQL_Database)
or even RAM (i.e., for temporary storage). See the section on [Storage Providers](#providers).

```javascript
var fs = new Filer.FileSystem();
fs.open('/myfile', 'w+', function(err, fd) {
  if (err) throw err;
  fs.close(fd, function(err) {
    if (err) throw err;
    fs.stat('/myfile', function(err, stats) {
      if (err) throw err;
      console.log('stats: ' + JSON.stringify(stats));
    });
  });
});
```

### API Reference

Like node.js, callbacks for methods that accept them are optional but suggested (i.e., if
you omit the callback, errors will be thrown as exceptions). The first callback parameter is
reserved for passing errors. It will be `null` if no errors occurred and should always be checked.

#### Filer.FileSystem(options, callback) constructor

File system constructor, invoked to open an existing file system or create a new one.
Accepts two arguments: an `options` object, and an optional `callback`. The `options`
object can specify a number of optional arguments, including:

* `name`: the name of the file system, defaults to `'"local'`
* `flags`: one or more flags to use when creating/opening the file system. Use `'FORMAT'` to force Filer to format (i.e., erase) the file system
* `provider`: an explicit storage provider to use for the file system's database context provider. See the section on [Storage Providers](#providers).

The `callback` function indicates when the file system is ready for use. Depending on the storage provider used, this might
be right away, or could take some time. The callback should expect an `error` argument, which will be null if everything worked.
Also users should check the file system's `readyState` and `error` properties to make sure it is usable.

```javascript
var fs;

function fsReady(err) {
  if(err) throw err;
  // Safe to use fs now...
}

fs = new Filer.FileSystem({
  name: "my-filesystem",
  flags: 'FORMAT',
  provider: new Filer.FileSystem.providers.Memory()
}, fsReady);
```

NOTE: if the optional callback argument is not passed to the `FileSystem` constructor,
operations done on the resulting file system will be queued and run in sequence when
it becomes ready.

####Filer.FileSystem.providers - Storage Providers<a name="providers"></a>

Filer can be configured to use a number of different storage providers. The provider object encapsulates all aspects
of data access, making it possible to swap in different backend storage options.  There are currently 4 different
providers to choose from:

* `FileSystem.providers.IndexedDB()` - uses IndexedDB
* `FileSystem.providers.WebSQL()` - uses WebSQL
* `FileSystem.providers.Fallback()` - attempts to use IndexedDB if possible, falling-back to WebSQL if necessary
* `FileSystem.providers.Memory()` - uses memory (not suitable for data that needs to survive the current session)

You can choose your provider when creating a `FileSystem`:

```javascript
var FileSystem = Filer.FileSystem;
var providers = FileSystem.providers;

// Example 1: Use the default provider (currently IndexedDB)
var fs1 = new FileSystem();

// Example 2: Explicitly use IndexedDB
var fs2 = new FileSystem({ provider: new providers.IndexedDB() });

// Example 3: Use one of IndexedDB or WebSQL, whichever is supported
var fs3 = new FileSystem({ provider: new providers.Fallback() });
```

Every provider has an `isSupported()` method, which returns `true` if the browser supports this provider:

```javascript
if( Filer.FileSystem.providers.WebSQL.isSupported() ) {
  // WebSQL provider will work in current environment...
}
```

You can also write your own provider if you need a different backend. See the code in `src/providers` for details.

####Filer.FileSystem.adapters - Adapters for Storage Providers

Filer based file systems can acquire new functionality by using adapters. These wrapper objects extend the abilities
of storage providers without altering them in anway. An adapter can be used with any provider, and multiple
adapters can be used together in order to compose complex functionality on top of a provider.

There are currently 2 adapters available:

* `FileSystem.adapters.Compression(provider)` - a compression adapter that uses [Zlib](https://github.com/imaya/zlib.js)
* `FileSystem.adapters.Encryption(passphrase, provider)` - an encryption adapter that uses [AES encryption](http://code.google.com/p/crypto-js/#AES)

```javascript
var FileSystem = Filer.FileSystem;
var providers = FileSystem.providers;
var adapters = FileSystem.adapters;

// Create a WebSQL-based, Encrypted, Compressed File System by
// composing a provider and adatpers.
var webSQLProvider = new providers.WebSQL();
var encryptionAdatper = new adapters.Encryption('super-secret-passphrase', webSQLProvider);
var compressionAdatper = new adatpers.Compression(encryptionAdapter);
var fs = new FileSystem({ provider: compressionAdapter });
```

You can also write your own adapter if you need to add new capabilities to the providers. Adapters share the same
interface as providers.  See the code in `src/providers` and `src/adapters` for many examples.

####Filer.Path

The node.js [path module](http://nodejs.org/api/path.html) is available via the `Filer.Path` object. It is
identical to the node.js version with the following differences:
* No support for `exits()` or `existsSync()`. Use `fs.stat()` instead.
* No notion of a current working directory in `resolve` (the root dir is used instead)

```javascript
var path = Filer.Path;
var dir = path.dirname('/foo/bar/baz/asdf/quux');
// dir is now '/foo/bar/baz/asdf'

var base = path.basename('/foo/bar/baz/asdf/quux.html');
// base is now 'quux.html'

var ext = path.extname('index.html');
// ext is now '.html'

var newpath = path.join('/foo', 'bar', 'baz/asdf', 'quux', '..');
// new path is now '/foo/bar/baz/asdf'
```

For more info see the docs in the [path module](http://nodejs.org/api/path.html) for a particular method:
* `path.normalize(p)`
* `path.join([path1], [path2], [...])`
* `path.resolve([from ...], to)`
* `path.relative(from, to)`
* `path.dirname(p)`
* `path.basename(p, [ext])`
* `path.extname(p)`
* `path.sep`
* `path.delimiter`

###FileSystem Instance Methods

Once a `FileSystem` is created, it has the following methods. NOTE: code examples below assume
a `FileSystem` instance named `fs` has been created like so:

```javascript
var fs = new Filer.FileSystem();
```

* [fs.rename(oldPath, newPath, callback)](#rename)
* [fs.ftruncate(fd, len, callback)](#ftruncate)
* [fs.truncate(path, len, callback)](#truncate)
* [fs.stat(path, callback)](#stat)
* [fs.fstat(fd, callback)](#fstat)
* [fs.lstat(path, callback)](#lstat)
* [fs.link(srcpath, dstpath, callback)](#link)
* [fs.symlink(srcpath, dstpath, [type], callback)](#symlink)
* [fs.readlink(path, callback)](#readlink)
* [fs.realpath(path, [cache], callback)](#realpath)
* [fs.unlink(path, callback)](#unlink)
* [fs.rmdir(path, callback)](#rmdir)
* [fs.mkdir(path, [mode], callback)](#mkdir)
* [fs.readdir(path, callback)](#readdir)
* [fs.close(fd, callback)](#close)
* [fs.open(path, flags, [mode], callback)](#open)
* [fs.utimes(path, atime, mtime, callback)](#utimes)
* [fs.futimes(fd, atime, mtime, callback)](#fsutimes)
* [fs.fsync(fd, callback)](#fsync)
* [fs.write(fd, buffer, offset, length, position, callback)](#write)
* [fs.read(fd, buffer, offset, length, position, callback)](#read)
* [fs.readFile(filename, [options], callback)](#readFile)
* [fs.writeFile(filename, data, [options], callback)](#writeFile)
* [fs.appendFile(filename, data, [options], callback)](#appendFile)
* [fs.setxattr(path, name, value, [flag], callback)](#setxattr)
* [fs.fsetxattr(fd, name, value, [flag], callback)](#fsetxattr)
* [fs.getxattr(path, name, callback)](#getxattr)
* [fs.fgetxattr(fd, name, callback)](#fgetxattr)
* [fs.removexattr(path, name, callback)](#removexattr)
* [fs.fremovexattr(fd, name, callback)](#fremovexattr)

#### fs.rename(oldPath, newPath, callback)<a name="rename"></a>

Renames the file at `oldPath` to `newPath`. Asynchronous [rename(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/rename.html).
Callback gets no additional arguments.

Example:

```javascript
// Rename myfile.txt to myfile.bak
fs.rename("/myfile.txt", "/myfile.bak", function(err) {
  if(err) throw err;
  // myfile.txt is now myfile.bak
});
```

#### fs.ftruncate(fd, len, callback)<a name="ftruncate"></a>

Change the size of the file represented by the open file descriptor `fd` to be length
`len` bytes. Asynchronous [ftruncate(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/ftruncate.html).
If the file is larger than `len`, the extra bytes will be discarded; if smaller, its size will
be increased, and the extended area will appear as if it were zero-filled. See also [fs.truncate()](#truncate).

Example:

```javascript
// Create a file, shrink it, expand it.
var buffer = new Uint8Array([1, 2, 3, 4, 5, 6, 7, 8]);

fs.open('/myfile', 'w', function(err, fd) {
  if(err) throw error;
  fs.write(fd, buffer, 0, buffer.length, 0, function(err, result) {
    if(err) throw error;
      fs.ftruncate(fd, 3, function(err) {
        if(err) throw error;
        // /myfile is now 3 bytes in length, rest of data discarded

        fs.ftruncate(fd, 50, function(err) {
          if(err) throw error;
          // /myfile is now 50 bytes in length, with zero padding at end

          fs.close(fd);
        });
      });
    });
  });
});
```

#### fs.truncate(path, len, callback)<a name="truncate"></a>

Change the size of the file at `path` to be length `len` bytes. Asynchronous [truncate(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/truncate.html). If the file is larger than `len`, the extra bytes will be discarded; if smaller, its size will
be increased, and the extended area will appear as if it were zero-filled. See also [fs.ftruncate()](#ftruncate).

Example:

```javascript
// Create a file, shrink it, expand it.
var buffer = new Uint8Array([1, 2, 3, 4, 5, 6, 7, 8]);

fs.open('/myfile', 'w', function(err, fd) {
  if(err) throw error;
  fs.write(fd, buffer, 0, buffer.length, 0, function(err, result) {
    if(err) throw error;
    fs.close(fd, function(err) {
      if(err) throw error;

      fs.truncate('/myfile', 3, function(err) {
        if(err) throw error;
        // /myfile is now 3 bytes in length, rest of data discarded

        fs.truncate('/myfile', 50, function(err) {
          if(err) throw error;
          // /myfile is now 50 bytes in length, with zero padding at end

        });
      });
    });
  });
});
```

#### fs.stat(path, callback)<a name="stat"></a>

Obtain file status about the file at `path`. Asynchronous [stat(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/stat.html).
Callback gets `(error, stats)`, where `stats` is an object with the following properties:

```
{
  node: <string>   // internal node id (unique)
  dev: <string>    // file system name
  size: <number>   // file size in bytes
  nlinks: <number> // number of links
  atime: <number>  // last access time
  mtime: <number>  // last modified time
  ctime: <number>  // creation time
  type: <string>   // file type (FILE, DIRECTORY, SYMLINK)
}
```

If the file at `path` is a symbolik link, the file to which it links will be used instead.
To get the status of a symbolic link file, use [fs.lstat()](#lstat) instead.

Examples:

```javascript
// Check if a directory exists
function dirExists(path, callback) {
  fs.stat(path, function(err, stats) {
    if(err) return callback(err);
    var exists = stats.type === "DIRECTORY";
    callback(null, exists);
  });
};

// Get the size of a file in KB
function fileSize(path, callback) {
  fs.stat(path, function(err, stats) {
    if(err) return callback(err);
    var kb = stats.size / 1000;
    callback(null, kb);
  });
}
```

#### fs.fstat(fd, callback)<a name="fstat"></a>

Obtain information about the open file known by the file descriptor `fd`.
Asynchronous [fstat(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/fstat.html).
Callback gets `(error, stats)`. `fstat()` is identical to `stat()`, except that the file to be stat-ed is
specified by the open file descriptor `fd` instead of a path.  See also [fs.stat](#stat)

Example:

```javascript
fs.open("/file.txt", "r", function(err, fd) {
  if(err) throw err;
  fs.fstat(fd, function(err, stats) {
    if(err) throw err;
    // do something with stats object
    // ...
    fs.close(fd);
  });
});
```

#### fs.lstat(path, callback)<a name="lstat"></a>

Obtain information about the file at `path` (i.e., the symbolic link file itself) vs.
the destination file to which it links. Asynchronous [lstat(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/lstat.html).
Callback gets `(error, stats)`. See also [fs.stat](#stat).

Example:

```javascript
// Create a symbolic link, /data/logs/current to /data/logs/august
// and get info about the symbolic link file, and linked file.
fs.link("/data/logs/august", "/data/logs/current", function(err) {
  if(err) throw err;

  // Get status of linked file, /data/logs/august
  fs.stat("/data/logs/current", function(err, stats) {
    if(err) throw err;
    // Size of /data/logs/august
    var size = stats.size;
  });

  // Get status of symbolic link file itself
  fs.lstat("/data/logs/current", function(err, stats) {
    if(err) throw err;
    // Size of /data/logs/current
    var size = stats.size;
  });
});
````

#### fs.link(srcPath, dstPath, callback)<a name="link"></a>

Create a (hard) link to the file at `srcPath` named `dstPath`. Asynchronous [link(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/link.html). Callback gets no additional arguments. Links are directory entries that point to the same file node.

Example:

```javascript
fs.link('/logs/august.log', '/logs/current', function(err) {
  if(err) throw err;
  fs.readFile('/logs/current', 'utf8', function(err, data) {
    // data is the contents of /logs/august.log
    var currentLog = data;
  });
});
```

#### fs.symlink(srcPath, dstPath, [type], callback)<a name="symlink"></a>

Create a symbolic link to the file at `dstPath` containing the path `srcPath`. Asynchronous [symlink(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/symlink.html). Callback gets no additional arguments.
Symbolic links are files that point to other paths.

NOTE: Filer allows for, but ignores the optional `type` parameter used in node.js.

Example:

```javascript
fs.symlink('/logs/august.log', '/logs/current', function(err) {
  if(err) throw err;
  fs.readFile('/logs/current', 'utf8', function(err, data) {
    // data is the contents of /logs/august.log
    var currentLog = data;
  });
});
```

#### fs.readlink(path, callback)<a name="readlink"></a>

Reads the contents of a symbolic link. Asynchronous [readlink(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/readlink.html). Callback gets `(error, linkContents)`, where `linkContents` is a string containing the symbolic link's link path.

Example:

```javascript
fs.symlink('/logs/august.log', '/logs/current', function(error) {
  if(error) throw error;

  fs.readlink('/logs/current', function(error, linkContents) {
    // linkContents is now '/logs/august.log'
  });
});
```

#### fs.realpath(path, [cache], callback)<a name="realpath"></a>

NOTE: Not yet implemented, see https://github.com/js-platform/filer/issues/85

#### fs.unlink(path, callback)<a name="unlink"></a>

Removes the directory entry located at `path`. Asynchronous [unlink(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/unlink.html).
Callback gets no additional arguments. If `path` names a symbolic link, the symbolic link will be removed
(i.e., not the linked file). Otherwise, the filed named by `path` will be removed (i.e., deleted).

Example:

```javascript
// Delete regular file /backup.old
fs.unlink('/backup.old', function(err) {
  if(err) throw err;
  // /backup.old is now removed
});
```

#### fs.rmdir(path, callback)<a name="rmdir"></a>

Removes the directory at `path`. Asynchronous [rmdir(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/rmdir.html).
Callback gets no additional arguments. The operation will fail if the directory at `path` is not empty.

Example:

```javascript
/**
 * Given the following dir structure, remove docs/
 *  /docs
 *    a.txt
 */

// Start by deleting the files in docs/, then remove docs/
fs.unlink('/docs/a.txt', function(err) {
  if(err) throw err;
  fs.rmdir('/docs', function(err) {
    if(err) throw err;
  });
});
```

#### fs.mkdir(path, [mode], callback)<a name="mkdir"></a>

Makes a directory with name supplied in `path` argument. Asynchronous [mkdir(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/mkdir.html). Callback gets no additional arguments.

NOTE: Filer allows for, but ignores the optional `mode` argument used in node.js.

Example:

```javascript
// Create /home and then /home/carl directories
fs.mkdir('/home', function(err) {
  if(err) throw err;

  fs.mkdir('/home/carl', function(err) {
    if(err) throw err;
    // directory /home/carl now exists
  });
});
```

#### fs.readdir(path, callback)<a name="readdir"></a>

Reads the contents of a directory. Asynchronous [readdir(3)](http://pubs.opengroup.org/onlinepubs/009695399/functions/readdir.html).
Callback gets `(error, files)`, where `files` is an array containing the names of each directory entry (i.e., file, directory, link) in the directory, excluding `.` and `..`.

Example:

```javascript
/**
 * Given the following dir structure:
 *  /docs
 *    a.txt
 *    b.txt
 *    c/
 */
fs.readdir('/docs', function(err, files) {
  if(err) throw err;
  // files now contains ['a.txt', 'b.txt', 'c']
});
```

#### fs.close(fd, callback)<a name="close"></a>

Closes a file descriptor. Asynchronous [close(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/close.html).
Callback gets no additional arguments.

Example:

```javascript
fs.open('/myfile', 'w', function(err, fd) {
  if(err) throw error;

  // Do something with open file descriptor `fd`

  // Close file descriptor when done
  fs.close(fd);
});
```

#### fs.open(path, flags, [mode], callback)<a name="open"></a>

Opens a file. Asynchronous [open(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/open.html).
Callback gets `(error, fd)`, where `fd` is the file descriptor. The `flags` argument can be:

* `'r'`: Open file for reading. An exception occurs if the file does not exist.
* `'r+'`: Open file for reading and writing. An exception occurs if the file does not exist.
* `'w'`: Open file for writing. The file is created (if it does not exist) or truncated (if it exists).
* `'w+'`: Open file for reading and writing. The file is created (if it does not exist) or truncated (if it exists).
* `'a'`: Open file for appending. The file is created if it does not exist.
* `'a+'`: Open file for reading and appending. The file is created if it does not exist.

NOTE: Filer allows for, but ignores the optional `mode` argument used in node.js.

Example:

```javascript
fs.open('/myfile', 'w', function(err, fd) {
  if(err) throw error;

  // Do something with open file descriptor `fd`

  // Close file descriptor when done
  fs.close(fd);
});
```

#### fs.utimes(path, atime, mtime, callback)<a name="utimes"></a>

Changes the file timestamps for the file given at path `path`. Asynchronous [utimes(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/utimes.html). Callback gets no additional arguments. Both `atime` (access time) and `mtime` (modified time) arguments should be a JavaScript Date.

Example:

```javascript
var now = Date.now();
fs.utimes('/myfile.txt', now, now, function(err) {
  if(err) throw err;
  // Access Time and Modified Time for /myfile.txt are now updated
});
```

#### fs.futimes(fd, atime, mtime, callback)<a name="futimes"></a>

Changes the file timestamps for the open file represented by the file descriptor `fd`. Asynchronous [utimes(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/utimes.html). Callback gets no additional arguments. Both `atime` (access time) and `mtime` (modified time) arguments should be a JavaScript Date.

Example:

```javascript
fs.open('/myfile.txt', function(err, fd) {
  if(err) throw err;

  var now = Date.now();
  fs.futimes(fd, now, now, function(err) {
    if(err) throw err;

    // Access Time and Modified Time for /myfile.txt are now updated

    fs.close(fd);
  });
});
```

#### fs.fsync(fd, callback)<a name="fsync"></a>

NOTE: Not yet implemented, see https://github.com/js-platform/filer/issues/87

#### fs.write(fd, buffer, offset, length, position, callback)<a name="write"></a>

Writes bytes from `buffer` to the file specified by `fd`. Asynchronous [write(2), pwrite(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/write.html). The `offset` and `length` arguments describe the part of the buffer to be written. The `position` refers to the offset from the beginning of the file where this data should be written. If `position` is `null`, the data will be written at the current position. The callback gets `(error, nbytes)`, where `nbytes` is the number of bytes written.

NOTE: Filer currently writes the entire buffer in a single operation. However, future versions may do it in chunks.

Example:

```javascript
// Create a file with the following bytes.
var buffer = new Uint8Array([1, 2, 3, 4, 5, 6, 7, 8]);

fs.open('/myfile', 'w', function(err, fd) {
  if(err) throw error;

  var expected = buffer.length, written = 0;
  function writeBytes(offset, position, length) {
    length = length || buffer.length - written;

    fs.write(fd, buffer, offset, length, position, function(err, nbytes) {
      if(err) throw error;

      // nbytes is now the number of bytes written, between 0 and buffer.length.
      // See if we still have more bytes to write.
      written += nbytes;

      if(written < expected)
        writeBytes(written, null);
      else
        fs.close(fd);
    });
  }

  writeBytes(0, 0);
});
```

#### fs.read(fd, buffer, offset, length, position, callback)<a name="read"></a>

Read bytes from the file specified by `fd` into `buffer`. Asynchronous [read(2), pread(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/read.html). The `offset` and `length` arguments describe the part of the buffer to be used. The `position` refers to the offset from the beginning of the file where this data should be read. If `position` is `null`, the data will be written at the current position. The callback gets `(error, nbytes)`, where `nbytes` is the number of bytes read.

NOTE: Filer currently reads into the buffer in a single operation. However, future versions may do it in chunks.

Example:

```javascript
fs.open('/myfile', 'r', function(err, fd) {
  if(err) throw error;

  // Determine size of file
  fs.fstat(fd, function(err, stats) {
    if(err) throw error;

    // Create a buffer large enough to hold the file's contents
    var nbytes = expected = stats.size;
    var buffer = new Uint8Array(nbytes);
    var read = 0;

    function readBytes(offset, position, length) {
      length = length || buffer.length - read;

      fs.read(fd, buffer, offset, length, position, function(err, nbytes) {
        if(err) throw error;

        // nbytes is now the number of bytes read, between 0 and buffer.length.
        // See if we still have more bytes to read.
        read += nbytes;

        if(read < expected)
          readBytes(read, null);
        else
          fs.close(fd);
      });
    }

    readBytes(0, 0);
  });
});
```

#### fs.readFile(filename, [options], callback)<a name="readFile"></a>

Reads the entire contents of a file. The `options` argument is optional, and can take the form `"utf8"` (i.e., an encoding) or be an object literal: `{ encoding: "utf8", flag: "r" }`. If no encoding is specified, the raw binary buffer is returned via the callback. The callback gets `(error, data)`, where data is the contents of the file.

Examples:

```javascript
// Read UTF8 text file
fs.readFile('/myfile.txt', 'utf8', function (err, data) {
  if (err) throw err;
  // data is now the contents of /myfile.txt (i.e., a String)
});

// Read binary file
fs.readFile('/myfile.txt', function (err, data) {
  if (err) throw err;
  // data is now the contents of /myfile.txt (i.e., an Uint8Array of bytes)
});
```

#### fs.writeFile(filename, data, [options], callback)<a name="writeFile"></a>

Writes data to a file. `data` can be a string or a buffer, in which case any encoding option is ignored. The `options` argument is optional, and can take the form `"utf8"` (i.e., an encoding) or be an object literal: `{ encoding: "utf8", flag: "w" }`. If no encoding is specified, and `data` is a string, the encoding defaults to `'utf8'`.  The callback gets `(error)`.

Examples:

```javascript
// Write UTF8 text file
fs.writeFile('/myfile.txt', "...data...", function (err) {
  if (err) throw err;
});

// Write binary file
var buffer = new Uint8Array([1, 2, 3, 4, 5, 6, 7, 8]);
fs.writeFile('/myfile', buffer, function (err) {
  if (err) throw err;
});
```

#### fs.appendFile(filename, data, [options], callback)<a name="appendFile"></a>

Writes data to the end of a file. `data` can be a string or a buffer, in which case any encoding option is ignored. The `options` argument is optional, and can take the form `"utf8"` (i.e., an encoding) or be an object literal: `{ encoding: "utf8", flag: "w" }`. If no encoding is specified, and `data` is a string, the encoding defaults to `'utf8'`.  The callback gets `(error)`.

Examples:

```javascript
// Append UTF8 text file
fs.writeFile('/myfile.txt', "More...", function (err) {
	if (err) throw err;
});
fs.appendFile('/myfile.txt', "Data...", function (err) {
  if (err) throw err;
});
// '/myfile.txt' would now read out 'More...Data...'

// Append binary file
var more = new Uint8Array([1, 2, 3, 4]);
var data = new Uint8Array([5, 6, 7, 8]);
fs.writeFile('/myfile', more, function (err) {
	if (err) throw err;
});
fs.appendFile('/myfile', buffer, function (err) {
  if (err) throw err;
});
// '/myfile' would now contain [1, 2, 3, 4, 5, 6, 7, 8]
```

#### fs.setxattr(path, name, value, [flag], callback)<a name="setxattr"></a>

Sets an extended attribute of a file or directory named `path`. Asynchronous [setxattr(2)](http://man7.org/linux/man-pages/man2/setxattr.2.html).
The optional `flag` parameter can be set to the following:
* `XATTR_CREATE`: ensures that the extended attribute with the given name will be new and not previously set. If an attribute with the given name already exists, it will return an `EExists` error to the callback.
* `XATTR_REPLACE`: ensures that an extended attribute with the given name already exists. If an attribute with the given name does not exist, it will return an `ENoAttr` error to the callback.

Callback gets no additional arguments.

Example:

```javascript
fs.writeFile('/myfile', 'data', function(err) {
  if(err) throw err;

  // Set a simple extended attribute on /myfile
  fs.setxattr('/myfile', 'extra', 'some-information', function(err) {
    if(err) throw err;

    // /myfile now has an added attribute of extra='some-information'
  });

  // Set a complex object attribute on /myfile
  fs.setxattr('/myfile', 'extra-complex', { key1: 'value1', key2: 103 }, function(err) {
    if(err) throw err;

    // /myfile now has an added attribute of extra={ key1: 'value1', key2: 103 }
  });
});
```

#### fs.fsetxattr(fd, name, value, [flag], callback)<a name="fsetxattr"></a>

Sets an extended attribute of the file represented by the open file descriptor `fd`. Asynchronous [setxattr(2)](http://man7.org/linux/man-pages/man2/setxattr.2.html).  See `fs.setxattr` for more details. Callback gets no additional arguments.

Example:

```javascript
fs.open('/myfile', 'w', function(err, fd) {
  if(err) throw err;

  // Set a simple extended attribute on fd for /myfile
  fs.fsetxattr(fd, 'extra', 'some-information', function(err) {
    if(err) throw err;

    // /myfile now has an added attribute of extra='some-information'
  });

  // Set a complex object attribute on fd for /myfile
  fs.fsetxattr(fd, 'extra-complex', { key1: 'value1', key2: 103 }, function(err) {
    if(err) throw err;

    // /myfile now has an added attribute of extra={ key1: 'value1', key2: 103 }
  });

  fs.close(fd);
});
```

#### fs.getxattr(path, name, callback)<a name="getxattr"></a>

Gets an extended attribute value for a file or directory. Asynchronous [getxattr(2)](http://man7.org/linux/man-pages/man2/getxattr.2.html).
Callback gets `(error, value)`, where `value` is the value for the extended attribute named `name`.

Example:

```javascript
// Get the value of the extended attribute on /myfile named `extra`
fs.getxattr('/myfile', 'extra', function(err, value) {
  if(err) throw err;

  // `value` is now the value of the extended attribute named `extra` for /myfile
});
```

#### fs.fgetxattr(fd, name, callback)<a name="fgetxattr"></a>

Gets an extended attribute value for the file represented by the open file descriptor `fd`.
Asynchronous [getxattr(2)](http://man7.org/linux/man-pages/man2/getxattr.2.html).
See `fs.getxattr` for more details. Callback gets `(error, value)`, where `value` is the value for the extended attribute named `name`.

Example:

```javascript
// Get the value of the extended attribute on /myfile named `extra`
fs.open('/myfile', 'r', function(err, fd) {
  if(err) throw err;

  fs.fgetxattr(fd, 'extra', function(err, value) {
    if(err) throw err;

    // `value` is now the value of the extended attribute named `extra` for /myfile
  });

  fs.close(fd);
});
```

#### fs.removexattr(path, name, callback)<a name="removexattr"></a>

Removes the extended attribute identified by `name` for the file given at `path`. Asynchronous [removexattr(2)](http://man7.org/linux/man-pages/man2/removexattr.2.html). Callback gets no additional arguments.

Example:

```javascript
// Remove an extended attribute on /myfile
fs.removexattr('/myfile', 'extra', function(err) {
  if(err) throw err;

  // The `extra` extended attribute on /myfile is now gone
});
```

#### fs.fremovexattr(fd, name, callback)<a name="fremovexattr"></a>

Removes the extended attribute identified by `name` for the file represented by the open file descriptor `fd`.
Asynchronous [removexattr(2)](http://man7.org/linux/man-pages/man2/removexattr.2.html). See `fs.removexattr` for more details.
Callback gets no additional arguments.

Example:

```javascript
// Remove an extended attribute on /myfile
fs.open('/myfile', 'r', function(err, fd) {
  if(err) throw err;

  fs.fremovexattr(fd, 'extra', function(err) {
    if(err) throw err;

    // The `extra` extended attribute on /myfile is now gone
  });

  fs.close(fd);
});
```
