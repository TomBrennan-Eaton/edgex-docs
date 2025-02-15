# Getting Started using Snaps

## Introduction

[Snaps](https://snapcraft.io/docs) are application packages that are easy to install and update while being 
secure, cross‐platform and self-contained.
Snaps can be installed on any Linux distribution with [snap support](https://snapcraft.io/docs/installing-snapd).

The following sub-sections provide generic instructions for installing, configuring, and managing EdgeX services using snaps. 

For the list of EdgeX snaps and specific instructions, please refer to the **[EdgeX Snaps](#edgex-snaps)** section.

### Installation
[installation]: #installation

When using the snap CLI, the installation is possible by simply executing:
```bash
snap install <snap>
```

This is similar to setting `--channel=latest/stable` or shorthand `--stable` and will install the latest stable release of a snap. In this case, `latest/stable` is the [channel](https://snapcraft.io/docs/channels), composed of `latest` track and `stable` risk level.

To install a specific version with long term support (e.g. 2.1), or to install a beta or development release, refer to the store page for the snap, choose install, and then pick the desired channel.
The store page also provides instructions for installation on different Linux distributions as well as the list of supported CPU architectures.

### Configuration
[configuration]: #configuration

EdgeX snaps are packaged with default service configuration files. In certain cases, few configuration fields are overridden within the snap for snap-specific deployment requirements.

There are a few ways to configure snapped services. In simple cases, it should be sufficient to modify the default config files before starting the services for the first time and use config overrides to change supported settings afterwards. Please refer below to learn about the different configuration methods.

#### Config files
The default configuration files are typically placed at `/var/snap/<snap>/current/config`. Upon a successful startup of an EdgeX service, the server configuration file (typically named `configuration.toml`) is uploaded to the [Registry](../../microservices/configuration/ConfigurationAndRegistry/#registry-provider) by default. After that, the local server configuration file will no longer be read and any modifications will not be applied. At this point, the configurations can be only changed via the Registry or by setting environment variables. Refer to [config registry](#config-registry) or [config overrides](#config-overrides) for details.

For device services, the Device and Device Profile files are submitted to [Core Metadata](../../microservices/core/metadata/Ch-Metadata) upon initial startup. Refer to the documentation of [Device Services](../../microservices/device/Ch-DeviceServices/) for details.

#### Config registry
The configurations that are uploaded to the Registry (i.e. Consul by default) can be modified using Consul's UI or [kv REST API](https://www.consul.io/api/kv). The Registry is a Core services, part of the [Platform Snap](#platform-snap).

Changes to configurations in Registry are loaded by the service at startup. If the service has already started, a restart is required to load new configurations. Configurations that are in the writable section get loaded not only at startup, but also during the runtime. In other words, changes to the writable configurations are loaded automatically without a restart.

Please refer to 
[Common Configuration](../../microservices/configuration/CommonConfiguration/) and 
[Configuration and Registry Providers](../../microservices/configuration/ConfigurationAndRegistry/) for more information.

#### Config provider snap
[config-provider-snap]: #config-provider-snap

Most EdgeX snaps have a [content interface](https://snapcraft.io/docs/content-interface) which allows another snap to seed it with configuration files.
This is useful for replacing all the configuration files in a service snap via a config provider snap without manual user interaction.

A config provider snap could be a standalone package with all the necessary configurations for multiple snaps. It will expose and one or more [interface](https://snapcraft.io/docs/interface-management) *slots* to allow connections from consumer *plugs*. The config provider snap can be released to the store just like any other snap. 
Upon a connection between consumer and provider snaps, the packaged config files get mounted inside the consumer snap, to be used by services.

Please refer to [edgex-config-provider](https://github.com/canonical/edgex-config-provider), for an example.

#### Config overrides
!!! edgey "EdgeX 2.2 - app options"
    This version of EdgeX snaps introduce a new scheme for the snap configuration options:
    ```
    apps.<app>.<type>.<key>
    ```
    where:

    - `<app>` is the name of the app (service, executable)
    - `<type>` is the type of option with respect to the app
    - `<key>` is key for the option. It could contain a path to set a value inside an object, e.g. `x.y=z` sets `{"x": {"y": "z"}}`.

    The app options are **disabled by default**. This is for usability purposes and to avoid confusion with legacy options. The app options may be enabled by default in the next major release of EdgeX.

    Refer to [edgexfoundry/edgex-go#3986](https://github.com/edgexfoundry/edgex-go/pull/3986) for details.

!!! edgey "EdgeX 2.2 - config options"
    The snaps now provide an interface to set any environment variable for supported apps.
    We call these the *config options* because they use a `config` prefix for the variable names. The config options are a subset of app options, which are **disabled by default**.

    This functionality supersedes for the snap *env options* (prefix `env.`) which allow setting a pre-defined set of configurations. Existing env options will continue to work until the next major EdgeX release. Please refer to snap READMEs (`jakarta` branch) for the documentation of the deprecated options.

    Enabling app options will migrate internal env options and persistently remove any newly set env option.

The EdgeX services allow overriding server configurations using environment variables. Moreover, the services read EdgeX [Common Environment Variables](../../microservices/configuration/CommonEnvironmentVariables/) to change configurations that aren't defined in config files.
The EdgeX snaps provide an interface via [snap configuration options](https://snapcraft.io/docs/configuration-in-snaps) to set environment variables for the services.
We call these the *config options* because they a have `config` prefix for the variable names.

The snap options for setting environment variable uses the the following format:

* `apps.<app>.config.<env-var>`: setting an app-specific value (e.g. `apps.core-data.config.service-port=1000`).
* `config.<env-var>`: setting a global value (e.g. `config.service-host=localhost` or `config.writable-loglevel=DEBUG`

where:

* `<app>` is the name of the app (service, executable)
* `<env-var>` is a lowercase, dash-separated mapping to uppercase, underscore-separate environment variable name (e.g. `x-y`->`X_Y`). The reason for such mapping is that uppercase and underscore characters are not supported as config keys for snaps.

Mapping examples:

| Snap config key        | Environment Variable     | Service configuration TOML                          |
|------------------------|--------------------------|-----------------------------------------------------|
| service-port           | SERVICE_PORT             | [Service]<br>Port                                   |
| clients-core-data-host  | CLIENTS_CORE_DATA_HOST  | [Clients]<br>--[Clients.core-data]<br>--Host        |
| edgex-startup-duration | [EDGEX_STARTUP_DURATION] | -                                                   |
| add-secretstore-tokens | [ADD_SECRETSTORE_TOKENS] | -                                                   |

[EDGEX_STARTUP_DURATION]: ../../microservices/configuration/CommonEnvironmentVariables/#edgex_startup_duration
[ADD_SECRETSTORE_TOKENS]: ../../security/Ch-Configuring-Add-On-Services/#configure-the-services-secret-store-to-use

!!! note
    The config options are supported as of EdgeX 2.2 and are disabled by default!

    Setting `app-options=true` is necessary to enable their support.

    Enabling app options will migrate internal env options and persistently remove any newly set env option.

For example, to change the service port of the core-data service on `edgexfoundry` snap to 8080:
```bash
snap set edgexfoundry app-options=true
snap set edgexfoundry apps.core-data.config.service-port=8080
```

The services load the set config options on startup. If the service has already started, a restart is necessary to load them.

#### Disabling security
!!! Warning
    Disabling security is NOT recommended, unless for demonstration purposes, or when there are other means to secure the services.

    The snap will NOT allow the Secret Store to be re-enabled. The only way to re-enable the Secret Store is to re-install the snap.
    
The Secret Store is used by EdgeX for secret management (e.g. certificates, keys, passwords). Use of the Secret Store by all services can be disabled globally. Note that doing so will also disable the API Gateway, as it depends on the Secret Store.

The following command disables the Secret Store and in turn the API Gateway:
```bash
sudo snap set edgexfoundry security-secret-store=off
```

All services in the snap except for the API Gateway are restricted by default to listening on localhost (127.0.0.1).
The API Gateway proxies external requests to internal services.
Since disabling the Secret Store also disables the API Gateway, the service endpoint will no longer be accessible from other systems.
They will be still accessible on the local machine for demonstration and testing.

If you really need to make an insecure service accessible remotely, set the bind address of the service to the IP address of that networking interface on the local machine. If you trust all your interfaces and want the services to accept connections from all, set it to `0.0.0.0`.

After disabling the Secret Store, the external services should be configured such that they don't attempt to initialize the security. For this purpose, [EDGEX_SECURITY_SECRET_STORE](../../microservices/configuration/CommonEnvironmentVariables/#edgex_security_secret_store) should be set to false, using the corresponding snap option: `config.edgex-security-secret-store`.

### Managing services
[managing services]: #managing-services

The services of a snap can be started/stopped/restarted using the snap CLI.
When starting/stopping, you can additionally set them to enable/disable which configures whether or not the service should also start on boot.

To list the services and check their status:
```bash
snap services <snap>
```

To start and optionally enable services:
```bash
# all services
snap start --enable <snap>

# one service
snap start --enable <snap>.<app>
```

Similarly, a service can be stopped and optionally disabled using `snap stop --disable`.

??? info "Seeding custom service startup using snap options"
    To spin up an EdgeX instance with a different startup configuration (e.g. enabled instead of disabled), the `edgexfoundry` snap provides the following config options that accept values `"on"`/`"off"` to enable/disable a service by default:
    
    * `consul`
    * `redis`
    * `core-metadata`
    * `core-command`
    * `core-data`
    * `support-notifications`
    * `support-scheduler`
    * `device-virtual`
    * `security-secret-store`
    * `security-proxy`

    Device and app service snaps provide a similar functionality using the `autostart` option.

    This is particularly useful when seeding the snap from a [Gadget](https://snapcraft.io/docs/gadget-snap) on an [Ubuntu Core](https://ubuntu.com/core) system.

To restart services, e.g. to load the configurations:
```bash
# all services
snap restart <snap>

# one service
snap restart <snap>.<app>
```

### Debugging
[debugging]: #debugging

The service logs can be queried using the `snap log` command.

For example, to query 100 lines and follow:
```bash
# all services
snap logs -n=100 -f <snap>

# one service
snap logs -n=100 -f <snap>.<app>
```
Check `snap logs --help` for details.

To query not only the service logs, but also the snap logs (incl. hook apps such as install and configure), use `journalctl`:
```bash
sudo journalctl -n 100 -f | grep <snap>
```

!!! info
    The verbosity of service logs is INFO by default. This can be changed by overriding the log level using the `WRITABLE_LOGLEVEL` environment variable using snap config overrides `apps.<app>.config.writable-loglevel` or globally as `config.writable-loglevel`.

## EdgeX Snaps
The following snaps are maintained by the EdgeX working groups:

- [Platform Snap](#platform-snap) - containing all core and security services along with few supporting and application services.
- Tools
    - [EdgeX UI](#edgex-ui)
    - [EdgeX CLI](#edgex-cli)
- Supporting Services
    - [EdgeX eKuiper](#edgex-ekuiper)
- Application Services
    - [App Service Configurable](#app-service-configurable)
    - [App RFID LLRP Inventory](#app-rfid-llrp-inventory)
- Device Services
    - [Device Camera](#device-camera)
    - [Device GPIO](#device-gpio)
    - [Device Grove](#device-grove)
    - [Device Modbus](#device-modbus)
    - [Device MQTT](#device-mqtt)
    - [Device REST](#device-rest)
    - [Device RFID LLRP](#device-rfid-llrp)
    - [Device SNMP](#device-snmp)
    - [Device Virtual](#device-virtual)

To find all EdgeX snaps on the public Snap Store, [search by keyword](https://snapcraft.io/search?q=edgex).

### Platform Snap
| [Installation][edgexfoundry] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/edgex-go/tree/main/snap) |

The main platform snap, simply called `edgexfoundry` contains
all reference core and security services along with a few supporting and application services.

Upon installation, the following EdgeX services are automatically started:

- consul (Registry)
- core-command
- core-data
- core-metadata
- kong-daemon (API Gateway / Reverse Proxy)
- postgres (kong's database)
- redis (default Message Bus and database backend for core-data and core-metadata)
- security-bootstrapper-redis (oneshot service to setup secure Redis)
- security-consul-bootstrapper (oneshot service to setup secure Consul)
- security-proxy-setup (oneshot service to setup Kong)
- security-secretstore-setup (oneshot service to setup Vault)
- vault (Secret Store)

The following services are disabled by default:

- support-notifications
- support-scheduler
- sys-mgmt-agent - *deprecated EdgeX component*
- kuiper (Rules Engine / eKuiper) - *deprecated; use the standalone [EdgeX eKuiper snap](#edgex-ekuiper)*
- app-service-configurable (used to filter events for kuiper) - *deprecated; use the standalone [App Service Configurable snap](#app-service-configurable)*


The disabled services can be manually enabled and started; see [managing services].

For the configuration of services, refer to [configuration].

Most services are exposed and accessible on localhost without access control. However, the access from outside is restricted to authorized users.

#### Adding API Gateway users
The service endpoints can be accessed securely through the API Gateway. The API Gateway requires a JSON Web Token (JWT) to authorize requests. Please refer to [Adding EdgeX API Gateway Users Remotely](../../security/Ch-AddGatewayUserRemotely/) and use the snapped `edgexfoundry.secrets-config` utility.

To get the usage help:
```bash
edgexfoundry.secrets-config proxy adduser -h
```
You may also refer to the [secrets-config proxy](../../security/secrets-config-proxy/) documentation.


!!! example "Creating an example user"
    Create private and public keys:
    ```bash
    openssl ecparam -genkey -name prime256v1 -noout -out private.pem
    openssl ec -in private.pem -pubout -out public.pem

    ```

    Read the API Gateway token:
    ```bash
    KONG_ADMIN_JWT=`sudo cat /var/snap/edgexfoundry/current/secrets/security-proxy-setup/kong-admin-jwt`
    ```

    Use secrets-config to add a user `example` with id `1000`:
    ```bash
    edgexfoundry.secrets-config proxy adduser --token-type jwt --user example --algorithm ES256 --public_key public.pem --id 1000 --jwt $KONG_ADMIN_JWT
    ```
    On success, the above command prints the user id.

??? tip "Seeding an admin user using snap options"
    To spin up a pre-configured and securely accessible EdgeX instance, the snap provides a way to pass the public key of a single user with snap options. When requested, the user is created with user `admin`, id `1` and JWT signing algorithm `ES256`. The snap option for passing the public key is:
    `apps.secrets-config.proxy.admin.public-key`.

    This is particularly useful when seeding the snap from a [Gadget](https://snapcraft.io/docs/gadget-snap) on an [Ubuntu Core](https://ubuntu.com/core) system.

!!! example "Generating a JWT token for the example user"
    On success, a JWT token is printed out and written to `user-jwt.txt` file.
    We use the user id `1000` as set in the previous example.
    ```bash
    edgexfoundry.secrets-config proxy jwt --algorithm ES256 --private_key private.pem --id 1000 --expiration=1h | tee user-jwt.txt
    ```

It is also possible to create the JWT token using bash and openssl. But that is beyond the scope of this guide.

Once you have the token, you can access the services via the API Gateway.

!!! example "Calling an API on behalf of example user"

    ```bash
    curl --insecure https://localhost:8443/core-data/api/v2/ping -H "Authorization: Bearer $(cat user-jwt.txt)"
    ```
    Output: `{"apiVersion":"v2","timestamp":"Mon May  2 12:14:17 CEST 2022","serviceName":"core-data"}`

#### Accessing Consul
Consul API and UI can be accessed using the consul token (Secret ID). For the snap, token is the value of `SecretID` typically placed in a JSON file at `/var/snap/edgexfoundry/current/secrets/consul-acl-token/mgmt_token.json`.

!!! example
    To get the token:
    ```bash
    sudo cat /var/snap/edgexfoundry/current/secrets/consul-acl-token/mgmt_token.json | jq -r '.SecretID' | tee consul-token.txt
    ```
    The output is printed out and written to `consul-token.txt`. Example output: `ee3964d0-505f-6b62-4c88-0d29a8226daa`

    Try it out locally:
    ```bash
    curl --insecure --silent http://localhost:8500/v1/kv/edgex/core/2.0/core-data/Service/Port -H "X-Consul-Token:$(cat consul-token.txt)"
    ```

    Through the API Gateway:  
    We need to pass both the Consul token and Secret Store token obtained in [Adding API Gateway users](#adding-api-gateway-users) examples.
    ```bash
    curl --insecure --silent https://localhost:8443/consul/v1/kv/edgex/core/2.0/core-data/Service/Port -H "X-Consul-Token:$(cat consul-token.txt)" -H "Authorization: Bearer $(cat user-jwt.txt)"
    ```

#### Changing TLS certificates
The API Gateway setup generates a self-signed certificate by default. To replace that with your own certificate, refer to API Gateway guide: [Using a bring-your-own external TLS certificate for API gateway](../../security/Ch-APIGateway/#using-a-bring-your-own-external-tls-certificate-for-api-gateway) and use the snapped `edgexfoundry.secrets-config` utility.

To get the usage help:
```bash
edgexfoundry.secrets-config proxy tls -h
```
You may also refer to the [secrets-config proxy](../../security/secrets-config-proxy/) documentation.

!!! example
    Given the following files created outside the scope of this document:
    
    * `cert.pem` certificate
    * `privkey.pem` private key
    * `ca.pem` certificate authority file (if not available in root certificates)

    Read the API Gateway token:
    ```bash
    KONG_ADMIN_JWT=`sudo cat /var/snap/edgexfoundry/current/secrets/security-proxy-setup/kong-admin-jwt`
    ```

    Add the certificate, using Kong Admin JWT to authenticate:
    ```bash
    edgexfoundry.secrets-config proxy tls --incert cert.pem --inkey privkey.pem --admin_api_jwt $KONG_ADMIN_JWT
    ```

    Try it out:
    ```bash
    curl --cacert ca.pem https://localhost:8443/core-data/api/v2/ping
    ```
    Output: `{"message":"Unauthorized"}`  
    This means that TLS is setup correctly, but the request is not authorized.  
    Set the `-v` command for diagnosing TLS issues.  
    The `--cacert` can be omitted if the CA is available in root certificates (e.g. CA-signed or pre-installed CA certificate).


??? tip "Seeding a custom TLS certificate using snap options"
    To spin up an EdgeX instance with a custom certificate, the snap provides the following configuration options:
    
    * `apps.secrets-config.proxy.tls.cert`
    * `apps.secrets-config.proxy.tls.key`
    * `apps.secrets-config.proxy.tls.snis` (comma-separated values)

    This is particularly useful when seeding the snap from a [Gadget](https://snapcraft.io/docs/gadget-snap) on an [Ubuntu Core](https://ubuntu.com/core) system.

<!-- DO NOT CHANGE THE TITLE. READMEs reference the anchor -->
#### Secret Store token
The services inside standalone snaps (e.g. device, app snaps) automatically receive a [Secret Store](../../security/Ch-SecretStore/) token when:

* The standalone snap is downloaded and installed from the store
* The platform snap is downloaded and installed from the store
* Both snaps are installed on the same machine
* The service is registered as an [add-on service](../../security/Ch-Configuring-Add-On-Services/)

The `edgex-secretstore-token` [content interface](https://snapcraft.io/docs/content-interface) provides the mechanism to automatically supply tokens to connected snaps.

Execute the following command to check the status of connections:
```bash
sudo snap connections edgexfoundry
```

To manually connect the edgexfoundry's plug to a standalone snap's slot:
```bash
snap connect edgexfoundry:edgex-secretstore-token <snap>:edgex-secretstore-token
```

Note that the token has a limited expiry time of 1h by default. The connection and service startup should happen within the validity period.

To better understand the snap connections, read the [interface management](https://snapcraft.io/docs/interface-management)

!!! tip "Extend the default Secret Store token TTL"
    The [TOKENFILEPROVIDER_DEFAULTTOKENTTL](../../microservices/configuration/CommonEnvironmentVariables/#tokenfileprovider_defaulttokenttl-security-secretstore-setup-service) environment variable can be set to override the default time to live (TTL) of the Secret Store tokens.
    This is useful when the microservice consumers of the tokens are expected to start after a delay that is longer than the default TTL.

    This can be achieved in the snap by setting the equivalent `tokenfileprovider-defaulttokenttl` config option:
    ```bash
    sudo snap set edgexfoundry app-options=true
    sudo snap set edgexfoundry apps.security-secretstore-setup.config.tokenfileprovider-defaulttokenttl=72h
    
    # Re-start the oneshot setup service to re-generate tokens:
    sudo snap start edgexfoundry.security-secretstore-setup
    
    # Optional: If the Kong admin token is read from the file to interact with Kong
    # Restart Kong to load the new admin token, generated by the setup service
    sudo snap restart edgexfoundry.kong-daemon
    ```

### EdgeX UI
| [Installation][edgex-ui] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/edgex-ui-go/tree/main/snap) |

For usage instructions, please refer to the [Graphical User Interface (GUI)](../tools/Ch-GUI/) guide.

By default, the UI is reachable locally at:
[http://localhost:4000](http://localhost:4000)

A valid JWT token is required to access the UI; follow [Adding API Gateway users](#adding-api-gateway-users) steps to generate a token.

To enable all the functionalities of the UI, the following support services should be running:

* Support Scheduler
* Support Notifications
* System Management Agent
* [EdgeX eKuiper](#edgex-ekuiper)

For example, to start/install all:
```
sudo snap start edgexfoundry.support-scheduler
sudo snap start edgexfoundry.support-notifications
sudo snap start edgexfoundry.sys-mgmt-agent
sudo snap install edgex-ekuiper
```

### EdgeX CLI
| [Installation][edgex-cli] | [Source](https://github.com/edgexfoundry/edgex-cli/tree/main/snap) |

For usage instructions, refer to [Command Line Interface (CLI)](../tools/Ch-CommandLineInterface/) guide.

### EdgeX eKuiper
| [Installation][edgex-ekuiper] | [Managing Services] | [Debugging] | [Source](https://github.com/canonical/edgex-ekuiper-snap) |

!!! edgey "EdgeX 2.2"
    This version of EdgeX introduces a standalone EdgeX eKuiper snap.
    The new snap is the supported way of using eKuiper with other EdgeX snaps.

For the documentation of the standalone EdgeX eKuiper snap, visit the [README](https://github.com/canonical/edgex-ekuiper-snap).

!!! note
    The standalone EdgeX eKuiper snap documented here should not be confused with the deprecated `edgexfoundry.kuiper` and `edgexfoundry.kuiper-cli` apps built into the platform. The standalone snap can provide similar functionality.

<!-- sorted alphabetically -->
### App Service Configurable
| [Installation][edgex-app-service-configurable] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/app-service-configurable/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-app-service-configurable/current/config/
└── res
    ├── external-mqtt-trigger
    │   └── configuration.toml
    ├── functional-tests
    │   └── configuration.toml
    ├── http-export
    │   └── configuration.toml
    ├── metrics-influxdb
    │   └── configuration.toml
    ├── mqtt-export
    │   └── configuration.toml
    ├── push-to-core
    │   └── configuration.toml
    └── rules-engine
        └── configuration.toml
```

Please refer to [App Service Configurable](../../microservices/application/AppServiceConfigurable/) guide for detailed usage instructions.

**Profile**

Before you can start the service, you must select one of available profiles, 
using snap options.

For example, to set `mqtt-export` profile using the snap CLI:
```bash
sudo snap set edgex-app-service-configurable profile=mqtt-export
```

!!! note
    The standalone App Service Configurable snap documented above should not be confused with the deprecated `edgexfoundry.app-service-configurable`, built into the platform snap. The standalone snap can serve the same functionality of filtering events for eKuiper by using the rules-engine profile.

### App RFID LLRP Inventory
| [Installation][edgex-app-rfid-llrp-inventory] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/app-rfid-llrp-inventory/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-app-rfid-llrp-inventory/current/config/
└── app-rfid-llrp-inventory
    └── res
        └── configuration.toml
```

**Aliases**

The aliases need to be provided for the service to work.  See [Setting the Aliases](https://github.com/edgexfoundry/app-rfid-llrp-inventory/blob/main/README.md#setting-the-aliases).

For the snap, this can either be by:

- using a [config-provider-snap] to provide a `configuration.toml` file with the correct aliases, before startup
- setting the values manually in Consul during or after deployment

### Device Camera
| [Installation][edgex-device-camera]  | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-camera-go/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-camera/current/config/
└── device-camera
    └── res
        ├── configuration.toml
        ├── devices
        │   └── camera.toml
        └── profiles
            ├── camera-axis.yaml
            ├── camera-bosch.yaml
            └── camera.yaml
```

### Device GPIO
| [Installation][edgex-device-gpio] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-gpio/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-gpio/current/config
└── device-gpio
    └── res
        ├── configuration.toml
        ├── devices
        │   └── device.custom.gpio.toml
        └── profiles
            └── device.custom.gpio.yaml
```

**GPIO Access**

This snap is strictly confined which means that the access to interfaces are subject to various security measures.

On a Linux distribution without snap confinement for GPIO (e.g. Raspberry Pi OS 11), the snap may be able to access the GPIO directly, without any snap interface and manual connections.

On Linux distributions with snap confinement for GPIO such as Ubuntu Core, the GPIO access is possible via the [gpio interface](https://snapcraft.io/docs/gpio-interface), provided by a gadget snap. 
The official [Raspberry Pi Ubuntu Core](https://ubuntu.com/download/raspberry-pi-core) image includes that gadget.
It is NOT possible to use this snap on Linux distributions that have the GPIO confinement but not the interface (e.g. Ubuntu Server 20.04), unless for development purposes.

In development environments, it is possible to install the snap in dev mode (using `--devmode` flag which disables security confinement and automatic upgrades) to allow direct GPIO access.

The `gpio` interface provides slots for each GPIO channel. The slots can be listed using:
```bash
$ sudo snap interface gpio
name:    gpio
summary: allows access to specific GPIO pin
plugs:
  - edgex-device-gpio
slots:
  - pi:bcm-gpio-0
  - pi:bcm-gpio-1
  - pi:bcm-gpio-10
  ...
```

The slots are not connected automatically. For example, to connect GPIO-17:
```
$ sudo snap connect edgex-device-gpio:gpio pi:bcm-gpio-17
```

Check the list of connections:
```
$ sudo snap connections
Interface        Plug                            Slot              Notes
gpio             edgex-device-gpio:gpio          pi:bcm-gpio-17    manual
…
```


### Device Grove
| [Installation][edgex-device-grove] | [Source](https://github.com/edgexfoundry/device-grove-c/tree/main/snap) |

!!! warning "beta"
    Device Grove snap is released as beta for `arm64`. It is compatible with EdgeX 1.3 only.

    It does not support the snap configurations described above.

The default configuration files are under `/var/snap/edgex-device-grove/current/config/`. 

This device service is started by default. 
Changes to the configuration files require a restart to take effect:
```bash
sudo snap restart edgex-device-grove
```

### Device Modbus
| [Installation][edgex-device-modbus] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-modbus-go/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-modbus/current/config/
└── device-modbus
    └── res
        ├── configuration.toml
        ├── devices
        │   └── modbus.test.devices.toml
        └── profiles
            └── modbus.test.device.profile.yml
```


### Device MQTT
| [Installation][edgex-device-mqtt] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-mqtt-go/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-mqtt/current/config/
└── device-mqtt
    └── res
        ├── configuration.toml
        ├── devices
        │   └── mqtt.test.device.toml
        └── profiles
            └── mqtt.test.device.profile.yml
```

### Device REST
| [Installation][edgex-device-rest] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-rest-go/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-rest/current/config/
└── device-rest
    └── res
        ├── configuration.toml
        ├── devices
        │   └── sample-devices.toml
        └── profiles
            ├── sample-image-device.yaml
            ├── sample-json-device.yaml
            └── sample-numeric-device.yaml

```

### Device RFID LLRP
| [Installation][edgex-device-rfid-llrp] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-rfid-llrp-go/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-rfid-llrp/current/config/
└── device-rfid-llrp
    └── res
        ├── configuration.toml
        ├── devices
        ├── profiles
        │   ├── llrp.device.profile.yaml
        │   └── llrp.impinj.profile.yaml
        └── provision_watchers
            ├── impinj.provision.watcher.json
            └── llrp.provision.watcher.json
```

**Subnet setup**

The `DiscoverySubnets` setting needs to be provided before a device discovery can occur. This can be done in a number of ways:

- Using `snap set` to set your local subnet information. Example:

    ```bash
    sudo snap set edgex-device-rfid-llrp apps.device-rfid-llrp.config.app-custom.discovery-subnets="192.168.10.0/24"
    
    curl -X POST http://localhost:59989/api/v2/discovery
    ```

- Using a [config-provider-snap] to set device configuration


- Using the `auto-configure` command. 
    
    This command finds all local network interfaces which are online and non-virtual and sets the value of `DiscoverySubnets` 
in Consul. When running with security enabled, it requires a Consul token, so it needs to be run as follows:

    ```bash
    # get Consul ACL token
    CONSUL_TOKEN=$(sudo cat /var/snap/edgexfoundry/current/secrets/consul-acl-token/bootstrap_token.json | jq ".SecretID" | tr -d '"') 
    echo $CONSUL_TOKEN 

    # start the device service and connect the interfaces required for network interface discovery
    sudo snap start edgex-device-rfid-llrp.device-rfid-llrp 
    sudo snap connect edgex-device-rfid-llrp:network-control 
    sudo snap connect edgex-device-rfid-llrp:network-observe 

    # run the nework interface discovery, providing the Consul token
    edgex-device-rfid-llrp.auto-configure $CONSUL_TOKEN
    ```

### Device SNMP
| [Installation][edgex-device-snmp] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-snmp-go/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-snmp/current/config/
└── device-snmp
    └── res
        ├── configuration.toml
        ├── devices
        │   └── device.snmp.trendnet.TPE082WS.toml
        └── profiles
            ├── device.snmp.patlite.yaml
            ├── device.snmp.switch.dell.N1108P-ON.yaml
            └── device.snmp.trendnet.TPE082WS.yaml
```

### Device USB Camera
| [Installation][edgex-device-usb-camera] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-usb-camera/tree/main/snap) |

This snap includes two services:
- Device USB Camera service
- [Simple RTSP Server](https://github.com/aler9/rtsp-simple-server) - used as the default RTSP server by Device USB Camera service

The services are **not started** by default. Please refer to [configuration] and [managing services].

The snap uses the [camera interface](https://snapcraft.io/docs/camera-interface) to access local USB camera devices. The [interface management](https://snapcraft.io/docs/interface-management) document describes how Snap interfaces are used to control the access to resources.

The default configuration files are installed at:
```
/var/snap/edgex-device-usb-camera/current/config
├── device-usb-camera
│   └── res
│       ├── configuration.toml
│       ├── devices
│       │   └── general.usb.camera.toml.example
│       └── profiles
│           └── general.usb.camera.yaml
└── rtsp-simple-server.yml
```

### Device Virtual
| [Installation][edgex-device-virtual] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-virtual-go/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-virtual/current/config
└── device-virtual
    └── res
        ├── configuration.toml
        ├── devices
        │   └── devices.toml
        └── profiles
            ├── device.virtual.binary.yaml
            ├── device.virtual.bool.yaml
            ├── device.virtual.float.yaml
            ├── device.virtual.int.yaml
            └── device.virtual.uint.yaml
```

### Device ONVIF Camera
| [Installation][edgex-device-onvif-camera] | [Configuration] | [Managing Services] | [Debugging] | [Source](https://github.com/edgexfoundry/device-onvif-camera/tree/main/snap) |

The service is **not started** by default. Please refer to [configuration] and [managing services].

The default configuration files are installed at:
```
/var/snap/edgex-device-onvif-camera/current/config
└── device-onvif-camera
    └── res
        ├── configuration.toml
        ├── devices
        │   ├── camera.toml.example
        │   └── control-plane-device.toml
        ├── profiles
        │   ├── camera.yaml
        │   └── control-plane.profile.yaml
        └── provision_watchers
            └── generic.provision.watcher.json
```

<!-- Store Links -->
[badge]: https://snapcraft.io/static/images/badges/en/snap-store-white.svg
[edgexfoundry]: https://snapcraft.io/edgexfoundry
[edgexfoundry-src]: https://github.com/edgexfoundry/edgex-go/tree/main/snap
[edgex-ui]: https://snapcraft.io/edgex-ui
[edgex-ui-src]: https://github.com/edgexfoundry/edgex-ui-go/tree/main/snap
[edgex-cli]: https://snapcraft.io/edgex-cli
[edgex-app-service-configurable]: https://snapcraft.io/edgex-app-service-configurable
[edgex-app-rfid-llrp-inventory]: https://snapcraft.io/edgex-app-rfid-llrp-inventory
[edgex-device-camera]: https://snapcraft.io/edgex-device-camera
[edgex-device-gpio]: https://snapcraft.io/edgex-device-gpio
[edgex-device-grove]: https://snapcraft.io/edgex-device-grove
[edgex-device-modbus]: https://snapcraft.io/edgex-device-modbus
[edgex-device-mqtt]: https://snapcraft.io/edgex-device-mqtt
[edgex-device-rest]: https://snapcraft.io/edgex-device-rest
[edgex-device-rfid-llrp]: https://snapcraft.io/edgex-device-rfid-llrp
[edgex-device-snmp]: https://snapcraft.io/edgex-device-snmp
[edgex-device-usb-camera]: https://snapcraft.io/edgex-device-usb-camera
[edgex-device-virtual]: https://snapcraft.io/edgex-device-virtual
[edgex-device-onvif-camera]: https://snapcraft.io/edgex-device-onvif-camera
[edgex-ekuiper]: https://snapcraft.io/edgex-ekuiper
