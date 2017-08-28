# Vault Plugin: Chef Node Auth Backend

The "chef-node" auth backend allows Nodes registered with a Chef server to
 authenticate using their private keys.

#### Enable Chef node authentication in vault

```
$ vault write sys/plugins/catalog/chefnode-auth-plugin \
  sha_256=<expected SHA256 Hex value of the plugin binary> \
  command=vault-plugin-auth-chefnode
$ vault auth-enable -path=chef-node -plugin=chefnode-auth-plugin plugin
```

## Configuration

The endpoint for the login is `auth/chef-node/config`.

The backend needs to be configured with the API endpoint of the Chef server it is
going to be authenticating against. It also needs a client to be setup in the Chef
server that it will connect as. The client needs to be able to read the client
objects of the nodes that it will be authenticating. The knife acl plugin can
be used to setup the correct permissions.

```
$ knife acl add client vault containers clients read
$ knife acl bulk add client vault clients ".*" read
```

This will give the vault client read permissions to newly created clients and add
the read perimssions to any existing clients. 

### Configuration Parameters

* `client_name` (string,required) - Name of the client to connect as
* `client_key` (string,required) - PEM encoded private key for the client
* `base_url` (string,required) - URL of Chef API endpoint

#### Via the CLI

```
$ vault write auth/chef-node/config client_name=vault client_key=@vault.pem \
  base_url=https://manage.chef.io/organizations/vaulttest
```

## Authentication

The endpoint for the login is `auth/chef-node/login`.

### Authentication Parameters

* `client_name` (string, required) - The name of the Chef client we are authenticating
* `signature` (string, required) - Base64 encoded Chef authentication signature. As generated by mixlib-authentication rubygem.
* `signature_version` (string, required) - The version of the Chef signature used. Currently should be set to 'algorithm=sha1;version=1.0;' 
* `timestamp` (string, required) - Timestamp used to generate signature in time.RFC3339 format

#### Via the API


```
$ curl -X POST "http://$VAULT_ADDR/v1/auth/chef-node/login" -d \
'{
    "client_name": "test_client",
    "signature_version": "algorithm=sha1;version=1.0;",
    "timestamp": "2016-10-26T04:47:09Z",
    "signature": "s9CqukciwdqA7f4H23Wz5xWbnMaFjurEPKpHnQoa8pGLrFVWPYiOAmO7DffTROxQjtEBqeufxfUwYE1oXcd7K7F/7GEtjSqTbWZBvw=="
}' 
```

The response will be in JSON. For Example:

```javascript
{
  "request_id": "7b686c95-fb90-d5d0-9a8b-9d4d321b2d06",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": null,
  "wrap_info": null,
  "warnings": null,
  "auth": {
    "client_token": "a147bdb1-e4fe-88db-68a8-e7aa992ce294",
    "accessor": "21d56cd9-dfc5-425d-667e-f200a88922d3",
    "policies": [
      "default"
    ],
    "metadata": null,
    "lease_duration": 2764800,
    "renewable": true
  }
}
```

## Policy mapping

Policies can be mapped to a Chef client.

The mapping of Chef client to policies is managed by using the `client/` path.

### Chef client

Policies mapped to a Chef client will apply to only that client.

```
$ vault write auth/chef-node/client/vault.example.com policies=cp
```

## API
### /auth/chef-node/config
#### POST
<dl class="api">
  <dt> Description </dt>
  <dd>
  Configure the Chef Node authentication backend.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>/auth/chef-node/config</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">base_url</span>
        <span class="param-flags">required</span>
        The API endpoint of the Chef server to authenticate against. Example:
        `https://manage.chef.io/organizations/vault`
      </li>
    </ul>
    <ul>
      <li>
        <span class="param">client_name</span>
        <span class="param-flags">required</span>
        Name of the Chef client vault should use to connect to the Chef server.
      </li>
    </ul>
    <ul>
      <li>
        <span class="param">client_key</span>
        <span class="param-flags">required</span>
        Private key of the Chef client vault should use to connect to the Chef server.
      </li>
    </ul>
  </dd>

  <dt>Returns</dt>
  <dd>204 response code.</dd>
</dl>

#### GET
<dl class="api">
  <dt> Description </dt>
  <dd>
  Retrieve the Chef Node configuration.
  </dd>

  <dt>Method</dt>
  <dd>GET</dd>

  <dt>URL</dt>
  <dd>/auth/chef-node/config</dd>

  <dt>Parameters</dt>
  <dd>None</dd>

  <dt>Returns</dt>
  <dd>

    
```javascript
    {
      "request_id": "c8f4ee10-2811-5456-98d6-ab71cf969f79",
      "lease_id": "",
      "lease_duration": 0,
      "renewable": false,
      "data": {
        "base_url": "https://manage.chef.io/organizations/vault",
        "client_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEo....py0YxXVJRIr5264hWN3y9\n-----END RSA PRIVATE KEY-----",
        "client_name": "vault"
      },
     "warnings": [
       "Read access to this endpoint should be controlled via ACLs as it will return the configuration information as-is, including any passwords."
     ]
   }
```

  </dd>
</dl>

### /auth/chef-node/login
#### POST
<dl class="api">
  <dt> Description </dt>
  <dd>
  Authenticate to Vault.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>/auth/chef-node/login</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">client_name</span>
        <span class="param-flags">required</span>
        Name of the Chef node that is authenticating.
      </li>
    </ul>
    <ul>
      <li>
        <span class="param">signature_version</span>
        <span class="param-flags">required</span>
        Version of the Chef signature to use.  Currently only "algorithm=sha1;version=1.0"
        is supported
      </li>
    </ul>
    <ul>
      <li>
        <span class="param">timestamp</span>
        <span class="param-flags">required</span>
        Timestamp used when the signature was generated.
      </li>
    </ul>
    <ul>
      <li>
        <span class="param">signature</span>
        <span class="param-flags">required</span>
        Chef authentication signature Base 64 encoded.
      </li>
    </ul>
  </dd>

  <dt>Returns</dt>
  <dd>

```javascript
    {
       "request_id": "7b686c95-fb90-d5d0-9a8b-9d4d321b2d06",
       "lease_id": "",
       "renewable": false,
       "lease_duration": 0,
       "data": null,
       "wrap_info": null,
       "warnings": null,
       "auth": {
         "client_token": "a147bdb1-e4fe-88db-68a8-e7aa992ce294",
         "accessor": "21d56cd9-dfc5-425d-667e-f200a88922d3",
         "policies": [
          "default"
         ],
         "metadata": null,
         "lease_duration": 2764800,
         "renewable": true
       }
     }
```

  </dd>
</dl>

### /auth/chef-node/clients
#### LIST
<dl class="api">
  <dt> Description </dt>
  <dd>
  Retrieve the list of configured Chef clients.
  </dd>

  <dt>Method</dt>
  <dd>LIST/GET</dd>

  <dt>URL</dt>
  <dd>/auth/chef-node/clients (LIST) or /auth/chef-node/clients?list=true (GET)</dd>

  <dt>Parameters</dt>
  <dd>None</dd>

  <dt>Returns</dt>
  <dd>

```javascript
    {
      "request_id": "a19f3fd5-d38b-9dd9-9032-ea55d4fcb496",
      "lease_id": "",
      "renewable": false,
      "lease_duration": 0,
      "data": {
        "keys": [
          "vault.example.com"
        ]
      },
      "wrap_info": null,
      "warnings": null,
      "auth": null
    }
```

  </dd>
</dl>

### /auth/chef-node/client/[client_name]
#### POST
<dl class="api">
  <dt> Description </dt>
  <dd>
  Set the policy mappings for a Chef client.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>/auth/chef-node/client/[client_name]</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">policies</span>
        <span class="param-flags">required</span>
        Comma separated list of policies to associate to the client.
      </li>
    <ul>
  </dd>

  <dt>Returns</dt>
  <dd>204 response code.</dd>
</dl>

#### GET
<dl class="api">
  <dt> Description </dt>
  <dd>
  Get the policy mappings for a Chef client.
  </dd>

  <dt>Method</dt>
  <dd>GET</dd>

  <dt>URL</dt>
  <dd>/auth/chef-node/client/[client_name]</dd>

  <dt>Parameters</dt>
  <dd>None</dd>

  <dt>Returns</dt>
  <dd>

```javascript
    {
        "request_id": "07849a62-388a-f9ec-ab90-36848f90718d",
        "lease_id": "",
        "lease_duration": 0,
        "renewable": false,
        "data": {
        "policies": [
           "default",
           "p1",
           "p2"
        ]
      },
      "warnings": null
    }
```

  </dd>
</dl>

#### DELETE
<dl class="api">
  <dt> Description </dt>
  <dd>
  Delete policy mappings for a Chef client.
  </dd>

  <dt>Method</dt>
  <dd>DELETE</dd>

  <dt>URL</dt>
  <dd>/auth/chef-node/client/[client_name]</dd>

  <dt>Parameters</dt>
  <dd>None</dd>

  <dt>Returns</dt>
  <dd>204 response code</dt>
</dl>