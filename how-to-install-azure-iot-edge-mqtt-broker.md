# How to setup Azure IoT Edge MQTT Broker on Ubuntu 18.04

The samples below show how to use custom MQTT topics with Azure IoT Edge.

## Install Azure IoT Edge

The current version is 1.2.0~rc4.1 (public-preview).

There are apparently no IoT Edge candidade packages for Ubuntu 20.04.


```bash
sudo apt-get update

sudo apt-get install -y curl gpg vim

curl https://packages.microsoft.com/config/ubuntu/18.04/multiarch/prod.list > ./microsoft-prod.list

sudo cp ./microsoft-prod.list /etc/apt/sources.list.d/

curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg

sudo cp ./microsoft.gpg /etc/apt/trusted.gpg.d/

sudo apt-get update

sudo apt-get install -y moby-engine
```

If there is a failure installing `moby`, get more details with:

```bash
curl -sSL https://raw.githubusercontent.com/moby/moby/master/contrib/check-config.sh -o check-config.sh

chmod +x check-config.sh

./check-config.sh
```

Proceed to install Edge,

```bash
apt-get update

apt-get install -y aziot-edge
```

> Verify version installed is 1.2.0*:
> 
> `dpkg -l | grep iot`

## Configure Azure IoT Edge

```bash
cp /etc/aziot/config.toml.edge.template /etc/aziot/config.toml

vi /etc/aziot/config.toml
```

Add the configuration according to https://docs.microsoft.com/en-us/azure/iot-edge/how-to-install-iot-edge?view=iotedge-2020-11#option-1-authenticate-with-symmetric-keys

```bash
iotedge config apply

systemctl status aziot-edged

sudo iotedge system status

sudo iotedge system logs
```

Enable the public preview features:

```bash
export experimentalFeatures__mqttBrokerEnabled=true

export experimentalFeatures__enabled=true

printenv | grep -i experimental
``` 

Finally apply the configuration:

```bash
iotedge config apply
```

## Prepare for running MQTT subscribe & publish sample

### Install Azure CLI

Reference: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt

```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### Deploy the edgeHub module:

Reference: https://docs.microsoft.com/en-us/azure/iot-edge/how-to-deploy-modules-cli?view=iotedge-2020-11

```bash
export connection_string="<your Azure IoT Hub iothubowner connection string>"

az iot edge set-modules --device-id <Edge Device ID> -l $connection_string --content ./mqtt-broker-deployment.json
```

Create the `mqtt-broker-deployment.json` file with the content below.

Make sure to update the parameters `<IoT Hub Name>`.

```json
{
    "modulesContent": {
        "$edgeAgent": {
            "properties.desired": {
                "schemaVersion": "1.0",
                "runtime": {
                    "type": "docker",
                    "settings": {
                        "minDockerVersion": "v1.25",
                        "loggingOptions": "",
                        "registryCredentials": {
                            "registryName": {
                                "username": "XXX",
                                "password": "XXX",
                                "address": "XXX"
                            }
                        }
                    }
                },
                "systemModules": {
                    "edgeAgent": {
                        "type": "docker",
                        "settings": {
                            "image": "mcr.microsoft.com/azureiotedge-agent:1.2.0-rc4",
                            "createOptions": "{}"
                        }
                    },
                    "edgeHub": {
                        "type": "docker",
                        "status": "running",
                        "restartPolicy": "always",
                        "settings": {
                            "image": "mcr.microsoft.com/azureiotedge-hub:1.2.0-rc4",
                            "createOptions": "{\"HostConfig\":{\"PortBindings\":{\"5671/tcp\":[{\"HostPort\":\"5671\"}],\"8883/tcp\":[{\"HostPort\":\"8883\"}],\"1883/tcp\":[{\"HostPort\":\"1883\"}],\"443/tcp\":[{\"HostPort\":\"443\"}]}}}"
                        },
                        "env": {
                            "experimentalFeatures__mqttBrokerEnabled": {
                                "value": "true"
                            },
                            "experimentalFeatures__enabled": {
                                "value": "true"
                            }
                        }
                    }
                },
                "modules": {}
            }
        },
        "$edgeHub": {
            "properties.desired": {
                "schemaVersion": "1.2",
                "routes": {
                    "Upstream": "FROM /messages/* INTO $upstream"
                },
				"storeAndForwardConfiguration": {
					"timeToLiveSecs": 7200
				},
                "mqttBroker": {
                    "authorizations": [{
                            "identities": [
                                "{{iot:identity}}"
                            ],
                            "allow": [{
                                    "operations": [
                                        "mqtt:connect"
                                    ]
                                }, {
                                    "operations": [
                                        "mqtt:publish"
                                    ],
                                    "resources": [
                                        "$iothub/+/messages/events/#"
                                    ]
                                }
                            ]
                        }, {
                            "identities": [
                                "<IoT Hub Name>.azure-devices.net/sub_client"
                            ],
                            "allow": [{
                                    "operations": [
                                        "mqtt:subscribe"
                                    ],
                                    "resources": [
                                        "test_topic"
                                    ]
                                }
                            ]
                        }, {
                            "identities": [
                                "<IoT Hub Name>.azure-devices.net/pub_client"
                            ],
                            "allow": [{
                                    "operations": [
                                        "mqtt:publish"
                                    ],
                                    "resources": [
                                        "test_topic"
                                    ]
                                }
                            ]
                        }
                    ]
                }
            }
        }
    }
}
```


## MQTT client stuff

```bash
sudo apt-get update && sudo apt-get install mosquitto-clients

# If not set already...
export connection_string="<your Azure IoT Hub iothubowner connection string>"

az iot hub device-identity create --device-id  sub_client -l $connection_string 
az iot hub device-identity create --device-id  pub_client -l $connection_string

az iot hub device-identity parent set --device-id sub_client --parent-device-id <Edge Device ID> -l $connection_string
az iot hub device-identity parent set --device-id pub_client --parent-device-id <Edge Device ID> -l $connection_string

az iot hub generate-sas-token -l $connection_string -d pub_client --key-type primary --du 86400
az iot hub generate-sas-token -l $connection_string -d sub_client --key-type primary --du 86400
```

Get the SAS tokens generated above and set the env vars:

```bash
export pubclientsas="<pub_client SAS token>"

export subclientsas="<sub_client SAS token>"
```

Run the mqtt clients to subscribe and publish to each other.

> Make sure to update `<IoT Hub Name>` before.

```bash
mosquitto_sub \
    -t "test_topic" \
    -i "sub_client" \
    -u "<IoT Hub Name>.azure-devices.net/sub_client/?api-version=2018-06-30" \
    -P "$subclientsas" \
    -h "localhost" \
    -V mqttv311 \
    -p 1883
	
mosquitto_pub \
    -t "test_topic" \
    -i "pub_client" \
    -u "<IoT Hub Name>.azure-devices.net/pub_client/?api-version=2018-06-30" \
    -P "$pubclientsas" \
    -h "localhost" \
    -V mqttv311 \
    -p 1883 \
    -m "hello"
```



# Troubleshooting:

Some useful commands:

```bash
sudo iotedge check

sudo iotedge logs edgeHub
```