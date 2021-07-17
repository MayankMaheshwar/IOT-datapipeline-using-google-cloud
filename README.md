Overview
The term Internet of Things (IoT) refers to the interconnection of physical devices with the global Internet. These devices are equipped with sensors and networking hardware, and each is globally identifiable. Taken together, these capabilities afford rich data about items in the physical world.

Cloud IoT Core is a fully managed service that allows you to easily and securely connect, manage, and ingest data from millions of globally dispersed devices. The service connects IoT devices that use the standard Message Queue Telemetry Transport (MQTT) protocol to other Google Cloud data services.

Cloud IoT Core has two main components:

A device manager for registering devices with the service, so you can then monitor and configure them.
A protocol bridge that supports MQTT, which devices can use to connect to Google Cloud.


1. Ensure that the Below APIs are successfully enabled-
Cloud IoT API
Cloud Pub/Sub API
Dataflow API



2. Create a Cloud Pub/Sub topic-

Cloud Pub/Sub is an asynchronous global messaging service. By decoupling senders and receivers, it allows for secure and highly available communication between independently written applications. Cloud Pub/Sub delivers low-latency, durable messaging.

In Cloud Pub/Sub, publisher applications and subscriber applications connect with one another through the use of a shared string called a topic. A publisher application creates and sends messages to a topic. Subscriber applications create a subscription to a topic to receive messages from it.

Create topic as string 'iotlab' and made a publisher.


3. Create a BigQuery dataset-

BigQuery is a serverless data warehouse. Tables in BigQuery are organized into datasets. In this lab, messages published into Pub/Sub will be aggregated and stored in BigQuery.

Define the schema and create tables after creating the dataset named "IOTLABDATASET"


4. Create a cloud storage bucket-

Cloud Storage allows world-wide storage and retrieval of any amount of data at any time. You can use Cloud Storage for a range of scenarios including serving website content, storing data for archival and disaster recovery, or distributing large data objects to users via direct download.

Created a bucket that will provide working space for your Cloud Dataflow pipeline.


5. Set up a Cloud Dataflow Pipeline-

Cloud Dataflow is a serverless way to carry out data analysis. In this lab, you will set up a streaming data pipeline to read sensor data from Pub/Sub, compute the maximum temperature within a time window, and write this out to BigQuery.

For Dataflow template, choose Pub/Sub Topic to BigQuery. When you choose this template, the form updates to review new fields below.

For Input Pub/Sub topic, enter projects/ followed by your Project ID then add /topics/iotlab. The resulting string will look like this: projects/qwiklabs-gcp-d2e509fed105b3ed/topics/iotlab

The BigQuery output table takes the form of Project ID:dataset.table (:iotlabdataset.sensordata). The resulting string will look like this: qwiklabs-gcp-d2e509fed105b3ed:iotlabdataset.sensordata

For Temporary location, enter gs:// followed by your Cloud Storage bucket name (should be your Project ID if you followed the instructions) then /tmp/. The resulting string will look like this: gs://qwiklabs-gcp-d2e509fed105b3ed-bucket/tmp/


6. Prepare your compute engine VM-

In your project, a pre-provisioned VM instance named iot-device-simulator will let you run instances of a Python script that emulate an MQTT-connected IoT device. Before you emulate the devices, you will also use this VM instance to populate your Cloud IoT Core device registry.

Connect to VM through SSH and install important packages and download the dataset present in
git clone http://github.com/GoogleCloudPlatform/training-data-analyst

7. Create a registry for IoT devices

gcloud beta iot registries create iotlab-registry \
   --project=$PROJECT_ID \
   --region=$MY_REGION \
   --event-notification-config=topic=projects/$PROJECT_ID/topics/iotlab

8. Create a Cryptographic Keypair

To allow IoT devices to connect securely to Cloud IoT Core, you must create a cryptographic keypair.

cd $HOME/training-data-analyst/quests/iotlab/
openssl req -x509 -newkey rsa:2048 -keyout rsa_private.pem \
    -nodes -out rsa_cert.pem -subj "/CN=unused"


9. Add simulated devices to the registry

For a device to be able to connect to Cloud IoT Core, it must first be added to the registry.

In your SSH session on the iot-device-simulator VM instance, enter this command to create a device called temp-sensor-buenos-aires:

gcloud beta iot devices create temp-sensor-buenos-aires \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=iotlab-registry \
  --public-key path=rsa_cert.pem,type=rs256

Enter this command to create a device called temp-sensor-istanbul:

gcloud beta iot devices create temp-sensor-istanbul \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=iotlab-registry \
  --public-key path=rsa_cert.pem,type=rs256


10. Run simulated devices

In your SSH session on the iot-device-simulator VM instance, enter these commands to download the CA root certificates from pki.google.com to the appropriate directory:

cd $HOME/training-data-analyst/quests/iotlab/
wget https://pki.google.com/roots.pem
Enter this command to run the first simulated device:

python cloudiot_mqtt_example_json.py \
   --project_id=$PROJECT_ID \
   --cloud_region=$MY_REGION \
   --registry_id=iotlab-registry \
   --device_id=temp-sensor-buenos-aires \
   --private_key_file=rsa_private.pem \
   --message_type=event \
   --algorithm=RS256 > buenos-aires-log.txt 2>&1 &
It will continue to run in the background.

Enter this command to run the second simulated device:

python cloudiot_mqtt_example_json.py \
   --project_id=$PROJECT_ID \
   --cloud_region=$MY_REGION \
   --registry_id=iotlab-registry \
   --device_id=temp-sensor-istanbul \
   --private_key_file=rsa_private.pem \
   --message_type=event \
   --algorithm=RS256
Telemetry data will flow from the simulated devices through Cloud IoT Core to your Cloud Pub/Sub topic. In turn, your Dataflow job will read messages from your Pub/Sub topic and write their contents to your BigQuery table.


11. Analyze the Sensor Data Using BigQuery

SELECT timestamp, device, temperature from iotlabdataset.sensordata
ORDER BY timestamp DESC
LIMIT 100


12. Happy Coding


