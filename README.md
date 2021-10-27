# High available Node Client for OpenStack Switf Object Storage

![GitHub release (latest by date)](https://img.shields.io/github/v/release/carboneio/ovh-object-storage-ha?style=for-the-badge)
[![Documentation](https://img.shields.io/badge/documentation-yes-brightgreen.svg?style=for-the-badge)](#api-usage)


> High availability, Performances, and Simplicity are the main focus of this tiny Node SDK to request the OpenStack Object Storage API. It was initially made to request the OVHCloud Object storage, but it can be used for any OpenStack Object Storage.

## Features
* 🦄 **Simple to use** - Only 4 methods: `Upload`, `Delete`, `List` and `Download` files
* 🌎 **High availability** - Initiate the SDK with a list of object storages credentials, and the SDK will switch storage if something goes wrong (Server/DNS not responding, timeout, error 500, too many redirection, authentication error, and more...).
* ✨ **Reconnect automatically** - If a request fails due to an authentication token expiration, the SDK fetches a new authentication token and retry the initial request with it.
* 🚀 **Performances** - Less than 500 lines of code with only 2 dependencies `simple-get` and `debug`.
* ✅ **100% tested**

## Install

### 1. Prior installing

you need a minimum of one object storage container, or you can synchronize Object Storages containers in order to access same objects if a fallback occur:
- Sync 2 containers: `1 <=> 2`. They would both need to share the same secret synchronization key.
- You can also set up a chain of synced containers if you want more than two. You would point `1 -> 2`, then `2 -> 3`, and finally `3 -> 1` for three containers. They would all need to share the same secret synchronization key.
Learn more [on the OpenStack documentation](https://docs.openstack.org/swift/latest/overview_container_sync.html) or [on the OVHCloud documentation](https://docs.ovh.com/us/en/storage/pcs/sync-container/).

<details>
  <summary>Quick tutorial to synchronise 1 container into another with OVHCloud Object Storage (1 -> 2 one way sync)</summary>

  1. Install the `swift-pythonclient`, an easy way to access Storages is with the Swift command line client, run on your terminal:
  ```
  $ pip install python-swiftclient
  ```
  2. Download the OpenStack RC file on the OVH account to change environment variables. Tab `Public Cloud` > `Users & Roles` > Pick the user and “Download OpenStack’s RC file”
  3. Open a terminal, load the contents of the file into the current environment:
  ```bash
  $ source openrc.sh
  ```
  4. In order for the containers to identify themselves, a key must be created and then configured on each container:
  ```bash
  $ sharedKey=$(openssl rand -base64 32)
  ```
  5. See which region you are connected to:
  ```bash
  env | grep OS_REGION
  ```
  6. Retrieve the Account ID `AUTH_xxxxxxx` of the destination container in order to configure the source container:
  ```bash
  destContainer=$(swift --debug stat containerBHS 2>&1 | grep 'curl -i.*storage' | awk '{ print $4 }') && echo $destContainer
  ```
  7. Change to the source region:
  ```bash
  OS_REGION_NAME=RegionSource
  ```
  8. Upload the key and the destination sync url to the source container:
  ```bash
  $ swiftclient post -t ‘//OVH_PUBLIC_CLOUD/RegionDestination/AUTH_xxxxxxxxx/containerNameDestination’ -k "$sharedKey" containerNameSource
  ```
  9. You can check that this has been configured by using the following command:
  ```bash
  $ swift stat containerName
  ```
  10. You can check if the synchronization worked by listing the files in each of the containers:
  ```bash
  $ OS_REGION_NAME=RegionSource && swift list containerName
  $ OS_REGION_NAME=RegionDestination && swift list containerName
  ```
</details>

### 2. Install the package with your package manager:

```bash
$ npm install --save ovh-object-storage-ha
// od
$ yarn add ovh-object-storage-ha
```
## API Usage

### Connection

Initialise the SDK with one or multiple storage, if something goes wrong, the next region will take over automatically. If any storage is available, an error message is returned `Error: Object Storages are not available`.

```js
const storageSDK = require('ovh-object-storage-ha');

let storage = storageSDK([{
  authUrl    : 'https://auth.cloud.ovh.net/v3',
  username   : 'username-1',
  password   : 'password-1',
  tenantName : 'tenantName-1',
  region     : 'region-1'
},
{
  authUrl    : 'https://auth.cloud.ovh.net/v3',
  username   : 'username-2',
  password   : 'password-2',
  tenantName : 'tenantName-2',
  region     : 'region-2'
}]);

storage.connection((err) => {
  if (err) {
    // Invalid credentials
  }
  // Success, connected!
})
```
### Upload a file

```js
const path = require(path);

/** SOLUTION 1: The file content can be passed by giving the file absolute path **/
storage.uploadFile('container', 'filename.jpg', path.join(__dirname, './assets/file.txt'), (err) => {
  if (err) {
    // handle error
  }
  // success
});

/** SOLUTION 2: A buffer can be passed for the file content **/
storage.uploadFile('container', 'filename.jpg', Buffer.from("File content"), (err) => {
  if (err) {
    // handle error
  }
  // success
});

/** SOLUTION 3: the function accepts a optionnal fourth argument `option` including query parameters and headers. List of query parameters and headers: https://docs.openstack.org/api-ref/object-store/?expanded=create-or-replace-object-detail#create-or-replace-object **/
storage.uploadFile('container', 'filename.jpg', Buffer.from("File content"), { queries: { temp_url_expires: '1440619048' }, headers: { 'X-Object-Meta-LocationOrigin': 'Paris/France' }}, (err) => {
  if (err) {
    // handle error
  }
  // success
});
```

### Download a file

```js
storage.downloadFile('templates', 'filename.jpg', (err, body, headers) => {
  if (err) {
    // handle error
  }
  // success, the `body` argument is the content of the file as a Buffer
});
```

### Delete a file

```js
storage.deleteFile('templates', 'filename.jpg', (err) => {
  if (err) {
    // handle error
  }
  // success
});
```

### List objects from a container

```js
/**
 * SOLUTION 1
 **/
storage.listFiles('templates', function (err, body) {
  if (err) {
    // handle error
  }
  // success
});

/**
 * SOLUTION 2
 * Possible to pass queries and overwrite request headers, list of options: https://docs.openstack.org/api-ref/object-store/? expanded=show-container-details-and-list-objects-detail#show-container-details-and-list-objects
 **/
storage.listFiles('templates', { queries: { prefix: 'prefixName' }, headers: { Accept: 'application/xml' } }, function (err, body) {
  if (err) {
    // handle error
  }
  // success
});
```

### Log

The package uses debug to print logs into the terminal. To activate logs, you must pass the `DEBUG=*` environment variable.
You can use the `setLogFunction` to override the default log function. Create a function with two arguments: `message` as a string, `level` as a string and the value can be: `info`/`warning`/`error`. Example to use:
```js
storage.setLogFunction((message, level) => {
  console.log(`${level} : ${message}`);
})
```

## Run tests

Install

```bash
$ npm install
```

To run all the tests:

```bash
$ npm run test
```

## 🤝 Contributing

Contributions, issues and feature requests are welcome!

Feel free to check [issues page](https://github.com/carboneio/ovh-object-storage-ha/issues).

## Show your support

Give a ⭐️ if this project helped you!

## 👤 Author

- [**@steevepay**](https://github.com/steevepay)
