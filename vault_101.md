# Secret Management with HashiCorp Vault on Debian 9-10: Part 1

Part 1 covers basic setup and admin of HashiCorp Vault on Debian 9/10 (and Ubuntu 18/19/20). 

We will deploy a production grade vault with minimal security considerations to get started. In the next part, we will take a deeper dive into security concepts and add hardening configurations based on recommendations from ```https://learn.hashicorp.com/tutorials/vault/production-hardening?in=vault/day-one-consul```.


## Introduction

Vault is a *secret management tool* which solves the problem of secret sprawl by *centralizing the storage of secrets*. 

### Secret Sprawl

A *secret* is any bit(s) of data, that if made public, will incur a significant cost to the owner.

All applications use secrets, like encryption keys, database passwords or api keys. 

The most common beginner practice is to store secrets as variables in the application.

```
# Pseudo-code
api_key="FERobaedeDosOxpUXfTwyYq/7OxZB5";
```
Although a convenient start, the developer is now restricted in their ability to share this code. Every copy of this code leads to copies of secrets being created across multiple devices. 

This is an example of secret sprawl; a side-effect of uncontrolled management of secrets. For any organization dealing with even fairly sensitive data, this slow leak of secrets have a compounding effect.

Environment variables are usually the next best choice. They shift the responsibility of secret mangement from the developer to the system admin.

```
# Pseudo-code
api_key=process.env.API_KEY;
```

Although the code can now be more freely shared, management of secrets is still difficult for the admin; especially once the system gains complexity. 


***Being able to manage all your mission-critical secrets from one place is the primary use-case of Vault.***

Vault is essentially an encrypted database, used just for secrets. It has a server/daemon component and a client component. The client can also use http endpoints to access the vault server.

In our application, we can now access secrets at various http endpoints set by the vault admin

```
# Pseudo-code
api_key = request({
  url: 'http://127.0.0.1:8200/v1/msec/api_key',
  method: 'GET',
  headers: {
      "X-Vault-Token":process.env.VAULT_TOKEN  // only a single secret env variable per application
  }
}).data.secret;

```



### Prerequisites

Basic knowledge in Linux and http is sufficient to follow this guide.

You will require *ONE* of the following to test this setup:

- Debian/Ubuntu cloud server instance * Easiest method *
- [Vagrant w/VirtualBox](https://learn.hashicorp.com/tutorials/vagrant/getting-started-index) * Preferred method * 
- Local Debian/Ubuntu setup  


If you are using Vagrant to test this setup, you must to do the following to correctly forward ports to use the Vault UI on your host browser.

- use 0.0.0.0 instead of 127.0.0.1 in the vault.hcl file
- use the Vagrantfile config provided below

```
Vagrant.configure("2") do |config|
  config.vm.box = "debian/buster64"
   config.vm.network "forwarded_port", guest: 8200, host: 8200, protocol: "tcp"

  config.vm.provider :virtualbox do |vb|
	vb.memory = 1024
	vb.cpus = 1
  end
end


```

## Installation

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

The installation creates a vault:vault user:group and generates self-signed TLS certificates which we will use in the next part covering security.

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
We will run vault on the localhost of the application as a monolith, therefore not requiring TLS just yet.

***ui*** sets whether or not to host a frontend admin panel.

***disable_mlock*** is set to false by default but recommended by hashicorp to be set to true for integrated storage.

***api_addr*** is the full address of where the vault api will be hosted.

***cluster_addr*** is the address at which other nodes within the cluster communicate with this node to maintain consensus on the state of the secrets. This will become relavant when we implment a multi-node cluster.

 
### Setup working directories and env

Create a directory for raft storage. This is where all the secrets and internal state configurations are saved.

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

```
# Paste into $HOME/.bashrc
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

### Initialize Vault

```
$ vault operator init
```
operator init is run only once during initial setup. 

Output:

```
Unseal Key 1: xxxxxxxxxxxxxxxxxxxxxxxxxx-YDzQF/ydO1SiXFgQ8n
Unseal Key 2: xxxxxxxxxxxxxxxxxxxxxxxxxx-MzSWY2M6b3pWDfqG2
Unseal Key 3: xxxxxxxxxxxxxxxxxxxxxxxxxx-6qb6AlwbuPC4TbfLP
Unseal Key 4: xxxxxxxxxxxxxxxxxxxxxxxxxx-MS6WIZqPJG78hW52m
Unseal Key 5: xxxxxxxxxxxxxxxxxxxxxxxxxx-OJ2roXUPd1k6D62jN

Initial Root Token: s.xxxxxxxxxxxxxxxxxxxxxxxxxx-UM2fJ
```

Vault starts in a sealed state. In this state nothing can be accessed from vault. 

3/5 Unseal Keys are required to unseal the vault. 

Once the vault is unsealed, it remains in this state until a *seal* command is issued OR the server is restarted. 

In the next part, we go into more detail on management of Unseal Keys and the Root Token. 
For now store them safely on your local working machine.

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

Login to vault using the Root Token.

```
$ vault login
```

### Secrets engine

The *secrets* sub-command allows us to configure vault's secret engine.

In Vault, a secret engine is simply a technique of managing and storing a specific type of secret. 

Our focus is on the general purpose ***kv*** (key-value) secret engine. 

Like variables in a program, it stores data as a ***key=vaule*** pair

Examples of kv
```
secret=pieshhh
test=somethingelse
rand=12903ikj12o3iljk213
```

Vault reads and writes secrets to paths, which can be accessed as http endpoints. 

For example:

the path ```msec/``` will serve its secret at ```http://127.0.0.1:8200/v1/msec/```



Enable the root path msec/ and declare it to be of type *kv*

```
$ vault secrets enable -path=msec/ kv
```
Output: 
```
Success! Enabled the kv secrets engine at: msec/
```

*Tip: it is easier to manage paths by using the application name as the root path. Use sub-paths to define an application specific secret.*


### Create secret

The kv sub-command is used to interact with the *kv* secrets backend

```
$ vault kv put msec/api_key secret=FERobaedeDosOxpUXfTwyYq/7OxZB5
```

Output:
  
```
Success! Data written to: msec/api_key
```
  
### Access secret

```
$ vault kv get msec/api_key
```

Output: 
  
```
== Data ==
Key    Value
---    -----
secret      FERobaedeDosOxpUXfTwyYq/7OxZB5

```

### Policies

A policy in Vault allows us to define access control to a secret path.

So far we have performed all operations as the root user that we logged in as.

To give applications access to secrets, we need to create specific policies for each of them and define their `capabilities`. Based on these policies, we create custom tokens for each application.

Create a basic_policy.hcl file in the home directory, or anywhere you wish to organize your policies. 

```
$ nano $HOME/basic_policy.hcl
```

Copy the following policy into the file.

```
path "msec/*" {
  capabilities = ["read"]
}
```

***path*** defines the secret path for which we are applying permissions. Here we are creating a policy for the path msec/ and all sub-paths as indicated by the wildcard ' * '.

***capabilities*** can include "create" , "update" , "delete", "list" or "sudo". We have opted for a strict read-only policy.

Then, write this policy file to vault.

```
$ vault policy write basic $HOME/basic_policy.hcl
```
Output:
```
Success! Uploaded policy: basic
```

This creates a policy named ***basic*** using the basic_policy.hcl file. 

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

*policy* requires a policy name for which to create a token

*no-default-policy* ensures that the token is only for the specified policy and not the *default* policy. 

The default policy has additional permissions that we do not want to give our application; so ***REMEMBER*** to always add this flag!


*ttl* (time to live) sets the time for which this token will be valid. The default value is 765h,  which is also the maximum. 

Expired tokens return a 403 Error, so keep in mind that your application will have to update its token within this timeframe to avoid downtime.
 

## Test

The *token* s.xxxxxxxxxxxxxxx-goK1VR can now be shared with a developer of the msec application.

All secrets related to this application can easily be managed by the admin at the msec/* secret path.

Finally, test the token using vault's http client via ***curl***

*Change the X-Vault-Token value to the token you just created*

*v1/$secret_path is suffixed to the api_addr*

```
$ curl -X GET --header "X-Vault-Token: s.xxxxxxxxxxxxxxx-goK1VR" http://127.0.0.1:8200/v1/msec/api_key 

```

Output:
```
{
    "request_id":"a8ac7f67-f885-85cb-8ce0-53224340e9ba",
    "lease_id":"",
    "renewable":false,"lease_duration":2764800,
    "data":{
        "secret":"FERobaedeDosOxpUXfTwyYq/7OxZB5"
    },
    "wrap_info":null,
    "warnings":null,
    "auth":null
}
```

The output JSON contains a field ***data*** with the key:value pair, with **secret** as the key.
 
*Tip: It is easier to manage your secrets by using a common name for all kv secret's key. In our example we have used `secret`. This allows unification in handling responses from the vault server as the secret's value in the JSON object response will always be contained within `response.data.secret`.*

## Conclusion

In the next part we will cover Hashicorp's recommended hardening options.

### Further steps

Try out the ui hosted at:
```
http://127.0.0.1:8200/ui/
```

All the steps covered in this tutorial can be done via the admin panel.

### References

For more details and alternative setup configurations, refer to the official docs:

https://www.vaultproject.io/docs
