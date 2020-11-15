## Secret Management with HashiCorp Vault on Debian 9-10: Part 1

In this multi-part tutorial series, we will cover setup and management of HashiCorp Vault on Debian 9/10 (and Ubuntu 18/19/20). 
Part 1 will cover basic setup and usage of vault. Rather than using vault in dev mode, we will deploy a production ready vault with minimal hardening. We will then handpick security features based on our infrastructural requirements.

## Introduction

Vault is a secret management tool which solves the problem of secret sprawl by centralizing the storage of secrets. 

### Secret Sprawl

All applications use secrets, like database passwords or api keys.

The most common beginner practice is to store secrets as variables in the application.

```
const api_key="12039io120933iok312903iok12309piok12093piok12/9012i3ok12309io";
```
Although a convenient start, this practice leads to sensitive data being shared across respositories and downloaded on multiple developer machines. This leads to copies of secrets being created across multiple devices. 

This is an example of secret sprawl; a side-effect of uncontrolled management of secrets. For any organization dealing with even fairly sensitive data, this slow leak of secrets has long term consequences.

Environment variables are a step better. They shifts the responsibility of secret mangement from the developer to the system admin.

```
const api_key=process.env.API_KEY;
```

Although the code can now be shared more freely, management of secrets is still difficult for the admin; especially once the system gains complexity. 

Vault is essentially an encrypted database, used just to store secrets. It has a server/daemon component and a client component. The client can also use http endpoints to access the vault server.

Rather than storing *ALL* secrets within our system env, we will only store a *single* vault token and manage all secrets within vault. 

What we end up with in the application is something like this: 

```
const options = {
  url: 'http://127.0.0.1:8200/v1/msec/data',
  method: 'GET',
  headers: {
      "X-Vault-Token":process.env.VAULT_TOKEN  
  },
};

const response = request(options);
const api_key = response.data.secret;
```

### Prerequisites

Basic knowledge in the following areas is sufficient to follow along this guide:

1. Linux (Debian/Ubuntu) 
2. HTTP Service Development
3. SQL/NoSQL Databases 

## Installation

### Dependencies

Debian will require a few dependencies to get started. These dependencies should already be present on Ubuntu.

```
$ sudo apt-get install software-properties-common gpg curl -y
```

Add Hashicorp's PGP Key to apt

```
$ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
```

Add the Hashicorp releases to the apt repository

```
$ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
```

Update apt and install vault

```
$ sudo apt-get update && sudo apt-get install vault -y

```

The installation creates a vault:vault user:group and generates self-signed TLS certificates which we will use in the next part of the series. 

Test the installation:

```
$ vault
```

## Configuration

### systemd

systemd is the recommended way to start and keep the vault server runing as a daemon process. 

#### Unit File

Create a vault service file.

```
$ sudo nano /etc/systemd/system/vault.service
```

Paste in this official Hashicorp Unit File.

```
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
ConditionFileNotEmpty=/etc/vault.d/vault.hcl
Requires=network-online.target
After=network-online.target
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=60
StartLimitIntervalSec=60
StartLimitBurst=3
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target

```

Observe the ExecStart value
```
/usr/bin/vault server -config=/etc/vault.d/vault.hcl
```
This is the command used to start the vault server.

Also notice:
```
Requires=network-online.target
After=network-online.target
```
These directives ensure that vault only starts after the network is online. For a local setup, this is not required.

### vault.hcl

Vault uses the HCL format (similar to JSON) for its configuration files. 

```
$ sudo nano /etc/vault.d/vault.hcl
```

Change the default vault.hcl file with the config provided below:

```
storage "raft" {
  path    = "/opt/raft"
  node_id = "vnode1"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = true

}

ui = true
disable_mlock = true
api_addr = "http://127.0.0.1:8200"
cluster_addr = "http://127.0.0.1:8201"

```

***storage*** chooses the storage backend used by vault to store secrets. 
Each storage type has trade-offs. 
We will use the internal *raft* storage, which is easy to setup and scale.   

- *path* determines where in the filesystem secrets are stored. 

- *node_id* can be any name given to this node. This becomes more relavant when more nodes are added to the network.

***listener***  chooses the type of communication protocol. We will be using the tcp protocol to access vault via an http endpoint.

- *address* sets up the address on which requests to access secrets will be served.
- *tls_disable* disables tls. 
For the purpose of this tutorial, we will run vault on the same server as the application, therefore not requiring TLS just yet.
We will implement TLS in Part 2 of the series.

***ui*** sets whether or not to host a frontend admin panel.

***disable_mlock*** is set to false by default but recommended by hashicorp to be set to true for integrated storage.

***api_addr*** is the full address of where the vault api will be hosted.

***cluster_addr*** is the address at which other nodes within the cluster communicate with this node to maintain consensus on the state of the secrets. *Only relavant for multi-node clusters.*

 
### Setup working directories and env

Create a directory for raft storage. 
```
$ sudo mkdir /opt/raft
```

Adjust permissions of all working directories.
```
$ sudo chown -R vault:vault /opt/raft
$ sudo chown -R vault:vault /etc/vault.d
$ sudo chmod 640 /etc/vault.d/vault.hcl
```

Give the vault binary the ability to use the mlock syscall without running the process as root. 
```
$ sudo setcap cap_ipc_lock=+ep /usr/bin/vault
```

Add the vault http address to the env for the client

```
$ nano $HOME/.bashrc
```
Paste the following on the last line
```
export VAULT_ADDR=http://127.0.0.1:8200
```



## Usage

### Start Server 

```
$ sudo systemctl enable vault
$ sudo systemctl start  vault
```

Check server status:
```
$ sudo systemctl status vault
```
Systemd runs the server sub-command as the *vault* user.
All subsequent client sub-commands can be run by any user that provides a valid token. 

### Initialize Vault

```
$ vault operator init
```
operator init is run only once during initial setup. 

The output of the operator init command will provide a fresh set of 5 unseal keys and a root token:

```
Unseal Key 1: xxxxxxxxxxxxxxxxxxxxxxxxxx-YDzQF/ydO1SiXFgQ8n
Unseal Key 2: xxxxxxxxxxxxxxxxxxxxxxxxxx-MzSWY2M6b3pWDfqG2
Unseal Key 3: xxxxxxxxxxxxxxxxxxxxxxxxxx-6qb6AlwbuPC4TbfLP
Unseal Key 4: xxxxxxxxxxxxxxxxxxxxxxxxxx-MS6WIZqPJG78hW52m
Unseal Key 5: xxxxxxxxxxxxxxxxxxxxxxxxxx-OJ2roXUPd1k6D62jN

Initial Root Token: s.xxxxxxxxxxxxxxxxxxxxxxxxxx-UM2fJ
```

Vault starts off in a sealed state. In this state nothing can be accessed from vault. 

3/5 keys are required to unseal the vault. 

Once the vault is unsealed, it remains in this state until a *seal* command is issued to lock it down.

The core of your operational security lies in how these keys are stored and managed. It is up to you to find the right balance of convenience and security. At the bare minimum, these keys should have atleast 1 reliable offline backup and should NOT be stored on the host machine.

HashiCorp recommends complete destruction of the root token after initial setup of users and auth methods. This will be covered in the next part of the series.

Begin the unseal process :
```
$ vault operator unseal
```
You will be prompted to provide an unseal key. 
On providing a valid key, the following output can be observed.

```
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       1be94049-ffb4-b9c4-6ac6-065f8af3cf6a
Version            1.5.5
HA Enabled         true

```

Reissue the unseal command for 3/5 keys and observe the value of `Sealed` change from `true` to `false`.  

Now we can start using vault!

### Login

Login to vault using the root token.

```
$ vault login
```

### Secrets engine

The *secrets* sub-command allows us to configure vault's secret engine.

Vault allows you to create and tune various different types of secrets engines. 

We will be focussing only on the ***kv*** (key-value) secret engine, which stores a single *key=vaule* at a specific path.

Vault reads and writes secrets to paths, which can be accessed as http endpoints. 

In this example:

the path ```msec/data``` will serve a kv secret at ```http://127.0.0.1:8200/v1/msec/data```


Enable the root path msec/ and declared it to be of type *kv*

```
$ vault secrets enable -path=msec/ kv
```
Output: 
```
Success! Enabled the kv secrets engine at: msec/

```


### Create secret

The kv sub-command is used to interact with the *kv* secrets backend

```
$ vault kv put msec/data secret=26i
```

Output:
  
```
Success! Data written to: msec/data

```
  
#### Access secret

```
$ vault kv get msec/data
```

Output: 
  
```
== Data ==
Key    Value
---    -----
secret      26i

```

### Policies

A policy in vault allows us to define access control to a secret path.

So far we have performed all operations as the root user that we logged in as.

It is good practice to use an admin user to create/update/delete secrets and create application specific user policies with limited permissions - usually only read.

Navigate to the home directory and create a basic_policy.hcl file

```
$ cd $HOME
$ nano basic_policy.hcl
```

Copy the following policy into the file.

```
path "msec/*" {
  capabilities = ["read"]
}
```

***path*** defines the secret path for which we are applying permissions. Here we are creating a policy for the path msec/ and all sub-paths as indicated by the wildcard *.

***capabilities*** can include "create" , "update" , "delete", "list" or "sudo". 

Then, from the same directory, run:

```
$ vault policy write basic ./basic_policy.hcl
```
This creates a policy named *basic* using the basic_policy.hcl file. 

Output:
```
Success! Uploaded policy: basic
```

Confirm the policy by reading it.
```
$ vault policy read basic
``` 

### Tokens 

Finally, to read the secret from our application, we need to create a token for the *basic* policy.

```
$ vault token create --policy=basic --no-default-policy -ttl=360h
```

Output:

```
Key                  Value
---                  -----
token                s.xxxxxxxxxxxxxxx-goK1VR
token_accessor       KIePuV0opLqVjoQZNfbKMTAs
token_duration       360h
token_renewable      true
token_policies       ["basic"]
identity_policies    []
policies             ["basic"]

```


*no-default-policy* ensures that the token is only for the basic policy and not the *default* policy. 

The default policy has additional permissions that we do not want to give our application; so ***REMEMBER*** to always add this flag!

*ttl* (time to live) sets the time for which this token will be valid. The default value is 765h,  which is also the maximum. 

Expired tokens return a 403 Error, so keep in mind that your application will have to update its token within this timeframe to avoid downtime.
  
## Test

Finally, lets attempt requesting vault for a secret via http.

For this example we will use ***curl***

*Change the X-Vault-Token value to the token you just created*

*v1/$secret_path is suffixed to the api_addr*

```
$ curl -X GET --header "X-Vault-Token: s.xxxxxxxxxxxxxxx-goK1VR" http://127.0.0.1:8200/v1/msec/data 

```
Output:
```
{
    "request_id":"a8ac7f67-f885-85cb-8ce0-53224340e9ba",
    "lease_id":"",
    "renewable":false,"lease_duration":2764800,
    "data":{
        "secret":"26i"
    },
    "wrap_info":null,
    "warnings":null,
    "auth":null
}
```
The output JSON contains a field ***data*** with the key:value pair, with **secret** as the key.
 
## Conclusion

Great! Now that you have a vault server running in production mode, hardening and customising it to suit your needs is the next step. 

Try out the ui hosted at:

```
http://127.0.0.1:8200/ui
```
### Further steps

In the next part, we will look at alternative policies and a few hardening options.

In the final part, we we run vault as a HA cluster.

#### Tips

To keep things more organized you can create more explainatory paths, for example:
  
Consider this path to store all database related credentials
  
```
path "databases/*" {
  capabilities = ["read"]
  
}
```

Create credentials:

```
vault kv put databases/mongo/username secret=vultr
vault kv put databases/mongo/password secret=shushhhhh
vault kv put databases/sql/username secret=vultr
vault kv put databases/sql/password secret=hushushhhh
```

It is recommended to use the same name for the kv secret *key*. Here we use *secret*. This allows unification in handling responses from the vault server as the secret value in the JSON object response will always be contained within `response.data.secret`

### References

For more details and alternative setup configurations, refer to the official docs:

https://www.vaultproject.io/docs
