---
title: "Connect Remote Clients"
linkTitle: "Connect Remote Clients"
weight: 6
description: >
    How to install the command line and Python clients, and connect to your Altinity.Cloud ClickHouse Cluster.
---

Now that we’ve shown how to create a cluster and use ClickHouse SQL queries into your new cluster, let’s connect to it remotely.

For the following, we’re going to be using the clickhouse-client program, but the same process will help you gain access from your favorite client.

Full instructions for installing ClickHouse can be found on the[ ClickHouse Installation page](https://clickhouse.tech/docs/en/getting-started/install/).  We’ll keep this simple and assume you’re using a Linux environment like Ubuntu.  For this example, we set up a virtual machine running Ubuntu 20.04.

First, we need to know our connection details for our Altinity.Cloud ClickHouse cluster.  To view your connection details:

1. From the Altinity Cloud Manager, select **Access Point** for the cluster to manage.
2. From here, you can copy and paste the settings for the ClickHouse client.  For example: 
    ```
    clickhouse-client -h yourdataset.yourcluster.altinity.cloud --port 9440 -s --user=admin --password
    ```




## **ClickHouse Client**

The ClickHouse Client is a command line based program that will be familiar 


### **Setting Up ClickHouse Client in Linux**

If you’ve already set up ClickHouse client, then you can skip this step.

 

For those who need quick instructions on how to install ClickHouse client in their deb based Linux environment (like Ubuntu), use the following:



*   **IMPORTANT NOTE**: As of this document’s publication, version 20.13 and above of the ClickHouse client is required to connect with the SNI enabled clusters.  These instructions use the testing version of that client.  An updated official stable build is expected to be released soon.

 


```
sudo apt-get install apt-transport-https ca-certificates dirmngr
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv E0C56BD4
 
echo "deb https://repo.clickhouse.tech/deb/testing/ main/" | sudo tee \
    /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update
 
sudo apt-get install -y clickhouse-client
```


 


### **Connect With ClickHouse Client**

If your ClickHouse client is ready, then you can copy and paste your connection settings into your favorite terminal program, and you’ll be connected.



<p id="gdcalert2" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image2.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert3">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image2.png "image_tooltip")



## **ClickHouse Python Driver**

For users who prefer Python, they can use the **clickhouse-driver **to connect through Python.  These instructions are very minimal, and are intended just to get you working in Python with your new Altinity.Cloud cluster.

 

These instructions are in the `bash` shell and require the following be installed in your environment:

 



*   `Python 3.7` and above
*   The Python module venv
*   `git`

 



*   **IMPORTANT NOTE**: We will be using clickhouse-driver v0.1.6 development build, which has support for Server Name Indication (SNI) when using TLS communications.

 

To connect with the Python clickhouse-driver:

 



1. **(Optional)** Setup your local environment.  For example: \


    ```
python3 -m venv test 
. test/bin/activate
```


2. Install the driver with pip3: \


    ```
pip3 install git+https://github.com/mymarilyn/clickhouse-driver@master#egg=clickhouse-driver
```


3. Start Python.
4. Add the client and connection details.  Cluster Explore provides the necessary information to link directly to your cluster.



<p id="gdcalert3" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image3.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert4">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image3.png "image_tooltip")


Import the clickhouse_driver client and enter the connection settings:


```
from clickhouse_driver import Client
client = Client('<HOSTNAME>', user='admin', password=<PASSWORD>, port=9440, secure='y', verify=False)
```


 

5.         	Run client.execute and submit your query.  Let’s just look at the tables from within Python:


```
>>> client.execute('SHOW TABLES in default')
[('events',), ('events_local',)]
```


 

6.         	You can perform selects and inserts as you need.  For example, continuing from our[ Your First Querie](https://docs.google.com/document/d/1skoWxs8UJcfaAPW0d5hA6DkTsBN3QJRFil_BmS9_YOA/edit#heading=h.pwrx6f4309bl)s using **Cluster Explore** above:


```
>>> result = client.execute('SELECT * FROM default.events')
>>> print(result)
[(datetime.date(2020, 11, 23), 1, 13, 'Example')]
```

For more information see the article[ Clickhouse And Python: Getting To Know The Clickhouse-driver Client](https://altinity.com/blog/clickhouse-and-python-getting-to-know-the-clickhouse-driver-client).
