# Example: Configuring mutual TLS

This example deploys a queue manager that requires client TLS authentication.

## Preparation

Open a terminal and login to the OpenShift cluster where you installed the MQ Operator.

# Configure and deploy the queue manager

Deploy the queuemanager that you will be using to test mTLS.

## Setup TLS for the queue manager

### In this example we will use self signed certificates. This is helpful to get a known good working mTLS setup when your own certificates do not work.
<br>

### Create a private key and a self-signed certificate for the queue manager

```
openssl req -newkey rsa:2048 -nodes -keyout qm2.key -subj "/CN=qm2" -x509 -days 3650 -out qm2.crt
```
This creates two files:

* Private key: `qm2.key`

* Certificate: `qm2.crt`
## Setup TLS for the MQ client application

### Create a private key and a self-signed certificate for the client application

```
openssl req -newkey rsa:2048 -nodes -keyout app1.key -subj "/CN=app1" -x509 -days 3650 -out app1.crt

```
This creates two files:

* Private key: `app1.key`

* Certificate: `app1.crt`

Check:

```
ls app1.*

```

You should see:

```
app1.crt	app1.key
```

You can also inspect the certificate:

```
openssl x509 -text -noout -in app1.crt

```

You'll see (truncated for redability):

```
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 11959216796104839727 (0xa5f7ac503bb6d22f)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=app1
        Validity
            Not Before: Jul 26 15:29:37 2021 GMT
            Not After : Jul 24 15:29:37 2031 GMT
        Subject: CN=app1
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
...
```

Note this is a self-signed certificate (Issuer is the same as Subject).

### Set up the client key database (keystore)
#### In this example we will use a p12 database instead of a KDB database.
#### Create the client key database:

```
openssl pkcs12 -export -out application.p12 -inkey $app1.key -in app1.crt -passout pass:password

```

#### Add the queue manager public key to the client key database (truststore):

```
keytool -keystore clientkey.jks -storetype jks -importcert -file qm2.crt -alias server-certificate
password = password

```

To check, list the database certificates:

```

keytool -list -v -keystore keystore.jks

```

Expected output:

```
Enter keystore password:  
Keystore type: JKS
Keystore provider: SUN

Your keystore contains 1 entry

Alias name: server-certificate
Creation date: Oct 11, 2022
Entry type: trustedCertEntry

Owner: CN=localhost
Issuer: CN=localhost
Serial number: eb40d77d9815aa4c
Valid from: Mon Aug 15 07:18:12 CDT 2022 until: Thu Aug 12 07:18:12 CDT 2032
Certificate fingerprints:
	 SHA1: E3:DB:A9:F2:79:28:B6:1B:C8:6A:C7:57:7F:2F:5D:F4:0E:A3:E6:A0
	 SHA256: D3:C1:58:02:4A:35:62:E1:78:1B:1F:00:8D:2F:F0:DC:B0:66:A1:36:A5:E3:98:CB:FC:C0:B7:51:00:44:73:3C
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 1

```

#### Add the client's certificate and key to the client key database:

First, put the key (`app1.key`) and certificate (`app1.crt`) into a PKCS12 file. PKCS12 is a format suitable for using in your MQ client application.

```
openssl pkcs12 -export -out app1.p12 -inkey app1.key -in app1.crt -password pass:password

```

#### The app1.p12 and clientkey.jks can now be used by the MQ client application. It is assumed the the client application is written in Java.

### Create TLS Secret for the Queue Manager

Our OpenShift project is called ibm-mq-poc.
```
oc create secret tls example-02-qm2-secret -n ibm-mq-poc --key="qm2.key" --cert="qm2.crt"

```

### Create TLS Secret with the client's certificate

```
oc create secret generic example-02-app1-secret -n ibm-mq-poc --from-file=app1.crt=app1.crt

```

## Setup and deploy the queue manager

### Create a config map containing MQSC commands

#### Create the config map yaml file

```
cat > qm2-configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-02-qm2-configmap
data:
  qm2.mqsc: |
    DEFINE QLOCAL('Q1') REPLACE DEFPSIST(YES) 
    DEFINE CHANNEL(QM2CHL) CHLTYPE(SVRCONN) REPLACE TRPTYPE(TCP) SSLCAUTH(REQUIRED) SSLCIPH('ANY_TLS12_OR_HIGHER')
    SET CHLAUTH(QM2CHL) TYPE(BLOCKUSER) USERLIST('nobody') ACTION(ADD)
  qm2.ini: |
    SSL:
    OutboundSNI=HOSTNAME
EOF
#
cat qm2-configmap.yaml

```

#### Notes:

The qm2.ini is added to configure the OutboundSNI option. The default value is CHANNEL. But in our case we are setting it to HOSTNAME. We will explain this setting in the next few sections.

The only difference with one-way TLS is `SSLCAUTH(REQUIRED)`. This is what mandates mutual TLS (the client must present its certificate).

#### Create the config map

```
oc apply -n ibm-mq-poc -f qm2-configmap.yaml

```

### Create the required route for SNI

#### This step is optional if using OutboundSNI=CHANNEL which is the default. Since we set the OutboundSNI=HOSTNAME in our qm2.ini, there is no longer a need for creating extra routes for each channel.

```
cat > qm2chl-route.yaml << EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: example-02-qm2-route
spec:
  host: qm2chl.chl.mq.ibm.com
  to:
    kind: Service
    name: qm2-ibm-mq
  port:
    targetPort: 1414
  tls:
    termination: passthrough
EOF
#
cat qm2chl-route.yaml

```

```
oc apply -n ibm-mq-poc -f qm2chl-route.yaml

```

Check:

```
oc describe route example-02-qm2-route

```

### Deploy the queue manager

#### Create the queue manager's yaml file

```
cat > qm2-qmgr.yaml << EOF
apiVersion: mq.ibm.com/v1beta1
kind: QueueManager
metadata:
  name: qm2
spec:
  license:
    accept: true
    license: L-RJON-CD3JKX
    use: NonProduction
  queueManager:
    name: QM2
    mqsc:
    - configMap:
        name: example-02-qm2-configmap
        items:
        - qm2.mqsc
    ini: 
    - configMap:
        name: example-02-qm2-configmap
        items:
        - qm2.ini
    storage:
      queueManager:
        type: ephemeral
  template:
    pod:
      containers:
        - env:
            - name: MQSNOAUT
              value: 'yes'
          name: qmgr
  version: 9.3.0.0-r2
  web:
    enabled: false
  pki:
    keys:
      - name: example
        secret:
          secretName: example-02-qm2-secret
          items: 
          - tls.key
          - tls.crt
    trust:
    - name: app1
      secret:
        secretName: example-02-app1-secret
        items:
          - app1.crt
EOF
#
cat qm2-qmgr.yaml

```
#### Note:

The only difference with one-way TLS is the `trust` section in the yaml file:

```
    trust:
    - name: app1
      secret:
        secretName: example-02-app1-secret
        items:
          - app1.crt
```
This adds the client certificate (from the secret we created earlier) to the queue manager's truststore. It is what allows the queue manager to verify the client.

#### Create the queue manager

```
oc apply -n ibm-mq-poc -f qm2-qmgr.yaml

```

# Set up and run the clients

Now that the queuemanager is running test the put and get messages with your MQ client application. Remember to use the app1.p12 and clientkey.jks as your keystore and truststore respectively.

You also need to set com.ibm.mq.cfg.SSL.outboundSNI system property to HOSTNAME. This allows your MQ client to be in sync with the MQ server configuration. 


## Test the connection

### Confirm that the queue manager is running

```
oc get qmgr -n ibm-mq-poc qm2

```

### Find the queue manager host name

```
qmhostname=`oc get route -n cp4i qm2-ibm-mq-qm -o jsonpath="{.spec.host}"`
echo $qmhostname

```

Test (optional):
```
ping -c 3 $qmhostname

```
### Test with your application
#### The application should be able to put and get messages
