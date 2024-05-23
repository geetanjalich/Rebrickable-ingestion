# Introduction 
I’ve been learning about how data moves around, and I started using Azure Data Factory, 
which is a cool tool for handling data. This project is aimed at demonstrating and 
consolidating my knowledge of ADF using best practices.

# Dataset
So, I found a website called rebrickable.(https://rebrickable.com/)

**Type of Data source:** It exposes two types of endpoints: There is a download page which exposes official lego data (Not all data is available on downloads page). There is also a RestAPI where customer information about the lego sets that he added to the rebrickable website.

**Cloud or on premises:** It is present on cloud where it is exposed to the internet. Here, we use Auto resolve Integration runtime provided by ADF.

**Push or pull approach:** We need to pull the data from the rebrickable website.

**Is it streaming or batch data ?** It is batch data uploaded everyday.

**Networking**:There is no issues regarding networking since it is publicly available.

**Is this data source under development?** Looking at the change log there is no active development having in the api .

**Security:** The api documentation provides authorization using a key. This should be generated from your profile.

**Rate limit or throttling:** Need to introduce som edelay since api has rate limits.

**Paginated:** Data is paginated

**Sync or Async:** The data is returned synchronously

**Incremental load or full load:** Count of data is 14k which is pretty small. So it is always full load. The api does not allow to get data incrementally.


# Challenge
Making sure the solution is **configurable** using ADF and to prepare the ingestion architecture with best practices.

1) **security standards:**
	a)creating a key vault to store secrets inside it.
	b) Wherever possible use managed identities of ADF to connect various objects.
	c) Granting access to managed identities via entra groups and grant permission to these groups.

1) **ADF should run according to some schedule(Once a day):**
	a) The ADF should connect to both endpoints(downloads page, RestAPI) and dump the data to data lake.
	b) Making sure ADF is flexible to determine what files and from which API endpoints we would like to dump data.

3) **Error handling:**
	a)Making sure the any types of errors in the pipeline is handled properly.
	b)Using logic Apps to send configurable mails to alert us in case of pipeline failure.

4) **Integration with Git Repo:**
	Making sure it follows Azure DevOps best practices to implement CI/CD pipelines.

5) **Making sure to separate the Resource group for Development and production**

I have created resource group under my pay-as-you-go subscription containing

- Azure Data Factory
- Data Lake Storage gen 2
- Logic app
- Databricks workspace
- Azure Key vault

obsidian://open?vault=Geetanjali&file=resources%20in%20dev%20rg.png
# Description of the process 

1) **Create a Data Lake Storage in Locally-Redundant Storage (LRS) as Data Lake Storage Gen 2, utilizing a hierarchical namespace.**
   a)Configure the access tier to be HOT (frequently accessed).
   b)Establish containers with a hierarchical structure within the Data Lake Storage.
   ![[storage account.png]]

2) **Creating Integration Runtime:**
	For purpose of this project I have used  Azure AutoResolveIntegrationRuntime
	Azure AutoResolveIntegrationRuntime, is a dynamic integration runtime provided by Azure Data Factory. This runtime automatically handles infrastructure provisioning and scaling based on workload demands, ensuring optimal performance and resource utilization for the project’s data integration tasks.

3) **Master data pipeline:**
	Here, i have make sure the pipeline is modularized without making it complicated.  
	Below is the master pipeline which has two child pipelines with some error handling logic.
	The master pipeline consist of only one parameter: Email Web Activity/ Logic App parameter.
![[Screenshot 2024-05-23 070845.png]]
The master data pipelines consists of two types of activity:
	a) **Execute pipeline:** Execute pipeline activity is used to combine child pipeline into single pipeline. Here, there are two child pipeline(copy api data pipeline and copy downloads data pipeline)
	b) **Web Activity:** Here web activity is configured using logic app url. 
		### **Create a new Logic App:**
			**Design Workflow:**
			- Add an HTTP trigger for receiving requests.
			**Add Gmail Action:**
			- Add “Gmail – Send Email” action.
			**Configure Email Details:**
			- Specify recipient email, subject, and body.
			**Save and Test:**
			- Save the Logic App.
			- Test it by triggering the HTTP request.
		
![[logic app.png]]

4) **Child data pipelines:**
	a) **Copy downloads data pipeline**
		There are two pipelines where one pipeline copies the data from downloads page of the website to ADLS gen2 and other pipeline copies the data from RestAPI of the website to ADLS gen2.
		**The below pipeline extracts the data from rebrickable website downloads page** .
	![[Screenshot 2024-05-23 071332.png]]
	
	 b) **Databricks notebook activity** 
	  I have used databricks notebook activity here in order to connect to adls gen2 storage account and extract the information about the data and keep the information in a lookup file in adls gen2.
	 c)  **Lookup activity**
	  The lookup activity will extract all the data from the lookup files 
	 d) **For each activity**
	  This will iterate over the lookup data sequentially and copies the data data to adls gen2 raw container. 
	 e)  **Copy activity** 
		Utilize the Copy Data Activity to unzip the downloaded file. Configure the activity with a dataset representing csv data with compression level set to Zip Deflate (.zip), pointing to the ZIP directory within the Data Lake container.
	 
	 Below shows the folder structuring of the data from ADF in ADLS gen2.
	![[Pasted image 20240523074844.png]]

**The below pipeline extracts the data from RESTAPI of website** 

![[Screenshot 2024-05-23 071404.png]]
1) **web activity** : Web activity gets the  Rest API authorization key and user token used to access Rest API stored in azure key vault.
2)  **wait activity:** Wait Activity into the pipeline to delay the process by 10 seconds.
	- This Wait Activity serves as a pause mechanism, allowing for controlled timing within the pipeline execution.
	- The 10-second delay ensures proper synchronization or sequencing of subsequent pipeline activities, optimizing the overall data processing workflow. 
3) **Databricks notebook activity** : In order to connect to adls gen2 storage account and extract the information about the data from Rest API and keep the information in a lookup file in adls gen2.
4)  **Lookup activity:** The lookup activity will extract all the data from the lookup file. 
5) **For each activity**: This will iterate over the lookup data sequentially and copies the data  to adls gen2 raw container.   
6) **Copy activity:** Utilize the Copy Data Activity to unzip the downloaded file. Configure the activity with a dataset representing csv data with compression level set to Zip Deflate (.zip), pointing to the ZIP directory within the Data Lake container.
