## IBM Cloud Service Management and Operations
### Performance Management in IBM Cloud Private
We have created a ICP Specific Dashboards for your use. These dashboards are focused on the performance of the ICP deployment and the name spaces
As with any other Grafana import you will need to specify the Prometheus data source. During installation the default prometheus data provider was used in creating these dashboards.

* Note: It is assumed that you have a working knowledge of Grafana and capable of editing a configuration in the dashboard.

<<<<<<< HEAD
The dashboards can be downloaded via this link [Dashboards](https://github.com/ibm-cloud-architecture/CSMO-ICP/blob/master/grafana/csmodashboards/grafanaICP.tar.gz)
=======
The dashboards can be downloaded via this link [Dashboards](https://github.com/ibm-cloud-architecture/CSMO-ICP/blob/master/grafana/csmodashboards/grafanaICP.tar.gz)  

Or Download them individually:

![ICP 2.1 Performance Dashboard](https://github.com/ibm-cloud-architecture/CSMO-ICP/blob/master/grafana/csmodashboards/"ICP 2.1 Performance IBM Provided.json")

![ICP 2.1 Namespace Performance](https://github.com/ibm-cloud-architecture/CSMO-ICP/blob/master/grafana/csmodashboards/ICP%202.1%20Namespaces%20Performance%20IBM%20Provided.json)
>>>>>>> 24bb2ab1f8b5dc7f267ca6811e0e6c026a5d08ad

Instructions for importing the dashboards [Grafana_Import](Grafana_Import.md)

#### ICP 2.1 Peformance IBM Provided Dashboard
##### Current Version 2.5
This dashboard provides a summary of current performance of the ICP environment. One should be able to immediately see what components are the top 5 consumers of CPU and Memory.  When importing this dashboard, you will need to change the "dummy" Kibana link to work in your installation. Instructions are following in this document

![ICPPerformance](images/ICPperf1.png)

+ Instructions for customizing the Link to Kibana [Kibana_Link](Edit_Kibana_Link.md)
+ Detailed Dashboard Information [ICP_Peformance](ICP_Performance_Dashboard_Detail.md)


####  ICP 2.1 Namespace Performance IBM Provided Dashboard
##### Current Version 2.5
This dashboard provides a focus on the namespace memory and CPU performance in your deployment. Just download and import the json file and you are good to go
You can switch between namespaces by changed to the desired namespace at the top of the dashboard. You can only view one namespace at a time.

![ICPnamespacePerformance](images/ICPnamspperf1.png)
