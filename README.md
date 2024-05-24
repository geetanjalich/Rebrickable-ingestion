# Overview 
The goal of this project is to apply data engineering (DE) concepts to orchestrate scraping and upload of Rebrickable(https://rebrickable.com/) data to Azure Data lake storage by making sure the solution is configurable. 
I’ve been learning about how data moves around, and I started using Azure Data Factory, which is a cool tool for handling data. This project is aimed at demonstrating and consolidating my knowledge of ADF using best practices.

# Dataset
So, I found a website called [Rebrickable](https://rebrickable.com/).

**Type of Data Source:** It exposes two types of endpoints: a download page which exposes official Lego data (not all data is available on the downloads page), and a RestAPI where customer information about the Lego sets added to the Rebrickable website is available.

**Cloud or On-Premises:** It is present on the cloud where it is exposed to the internet. Here, we use Auto Resolve Integration runtime provided by ADF.

**Push or Pull Approach:** We need to pull the data from the Rebrickable website.

**Streaming or Batch Data:** It is batch data uploaded every day.

**Networking:** There are no issues regarding networking since it is publicly available.

**Is this Data Source Under Development?** Looking at the change log, there is no active development happening in the API.

**Security:** The API documentation provides authorization using a key. This should be generated from your profile.

**Rate Limit or Throttling:** Need to introduce some delay since the API has rate limits.

**Paginated:** Data is paginated.

**Sync or Async:** The data is returned synchronously.

**Incremental Load or Full Load:** The count of data is 14k which is pretty small. So it is always full load. The API does not allow getting data incrementally.

# Challenge
Making sure the solution is configurable using ADF and to prepare the ingestion architecture with best practices.

1. **Security Standards:**
   - Create a key vault to store secrets inside it.
   - Wherever possible, use managed identities of ADF to connect various objects.
   - Grant access to managed identities via Entra groups and grant permission to these groups.

2. **ADF Should Run According to Some Schedule (Once a Day):**
   - The ADF should connect to both endpoints (downloads page, RestAPI) and dump the data to Data Lake.
   - Ensure ADF is flexible to determine what files and from which API endpoints we would like to dump data.

3. **Error Handling:**
   - Ensure any types of errors in the pipeline are handled properly.
   - Use Logic Apps to send configurable mails to alert us in case of pipeline failure.

4. **Integration with Git Repo:**
   - Ensure it follows Azure DevOps best practices to implement CI/CD pipelines.

5. **Separate the Resource Group for Development and Production:**
   - Create resource groups under my pay-as-you-go subscription containing:
     - Azure Data Factory
     - Data Lake Storage Gen 2
     - Logic App
     - Databricks Workspace
     - Azure Key Vault

![Resources in Dev RG](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/2846cc4d-c180-4c66-97ad-b4bff917ce47)

# Description of the Process

1. **Create a Data Lake Storage in Locally-Redundant Storage (LRS) as Data Lake Storage Gen 2, utilizing a hierarchical namespace:**
   - Configure the access tier to be HOT (frequently accessed).
   - Establish containers with a hierarchical structure within the Data Lake Storage.
   
   ![Storage Account](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/7d3c7bf5-c2b4-4f2b-80a7-d83ba6c4c0ee)

2. **Creating Integration Runtime:**
   - For the purpose of this project, I have used Azure AutoResolveIntegrationRuntime.
   - Azure AutoResolveIntegrationRuntime is a dynamic integration runtime provided by Azure Data Factory. This runtime automatically handles infrastructure provisioning and scaling based on workload demands, ensuring optimal performance and resource utilization for the project’s data integration tasks.

3. **Master Data Pipeline:**
   - Ensure the pipeline is modularized without making it complicated. 
   - Below is the master pipeline which has two child pipelines with some error handling logic.
   - The master pipeline consists of only one parameter: Email Web Activity/ Logic App parameter.

   ![Master Pipeline](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/e1716597-0040-40f3-880d-d4e233f0bed5)

   The master data pipelines consist of two types of activity:
   - **Execute Pipeline:** Execute pipeline activity is used to combine child pipelines into a single pipeline. Here, there are two child pipelines (copy API data pipeline and copy downloads data pipeline).
   - **Web Activity:** Web activity is configured using Logic App URL. 
     - **Create a new Logic App:**
       - **Design Workflow:** Add an HTTP trigger for receiving requests.
       - **Add Gmail Action:** Add “Gmail – Send Email” action.
       - **Configure Email Details:** Specify recipient email, subject, and body.
       - **Save and Test:** Save the Logic App. Test it by triggering the HTTP request.

   ![Logic App](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/1299c29d-7cb0-4a0d-ace9-b1d1e84b37c1)

4. **Child Data Pipelines:**
   - **Copy Downloads Data Pipeline:**
     - There are two pipelines: one pipeline copies the data from the downloads page of the website to ADLS Gen2, and the other pipeline copies the data from RestAPI of the website to ADLS Gen2.
     - **The below pipeline extracts the data from Rebrickable website downloads page:**

     ![Downloads Pipeline](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/5326bf65-0ab1-4758-b3db-6b76957d807d)
   
     - **Databricks Notebook Activity:** Used to connect to ADLS Gen2 storage account, extract the information about the data, and keep the information in a lookup file in ADLS Gen2.
     - **Lookup Activity:** Extracts all the data from the lookup files.
     - **For Each Activity:** Iterates over the lookup data sequentially and copies the data to ADLS Gen2 raw container.
     - **Copy Activity:** Utilizes the Copy Data Activity to unzip the downloaded file. Configures the activity with a dataset representing CSV data with compression level set to Zip Deflate (.zip), pointing to the ZIP directory within the Data Lake container.

     Below shows the folder structuring of the data from ADF in ADLS Gen2.

     ![Folder Structuring](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/0b8cb27d-ec07-43c6-b3ca-140475e55c2d)
     ![Folder Structuring](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/3a255e46-bfef-4b36-8173-8674270291d8)

   - **Extracting Data from RestAPI:**

     ![RestAPI Pipeline](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/ec9f742c-489e-4ebf-8145-9e599e70b9b4)

     - **Web Activity:** Gets the Rest API authorization key and user token used to access Rest API stored in Azure Key Vault.
     - **Wait Activity:** Wait Activity delays the process by 10 seconds.
       - This Wait Activity serves as a pause mechanism, allowing for controlled timing within the pipeline execution.
       - The 10-second delay ensures proper synchronization or sequencing of subsequent pipeline activities, optimizing the overall data processing workflow.
     - **Databricks Notebook Activity:** Connects to ADLS Gen2 storage account and extracts the information about the data from Rest API, keeping the information in a lookup file in ADLS Gen2.
     - **Lookup Activity:** Extracts all the data from the lookup file.
     - **For Each Activity:** Iterates over the lookup data sequentially and copies the data to ADLS Gen2 raw container.
     - **Copy Activity:** Utilizes the Copy Data Activity to unzip the downloaded file. Configures the activity with a dataset representing CSV data with compression level set to Zip Deflate (.zip), pointing to the ZIP directory within the Data Lake container.

5) **Integration with Git Repo with following best CI/CD practices in Azure DevOps:**
	![git configuration](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/9703431c-844a-46ec-8c86-c048a5d85ab4)
		
	I created a project in Azure DevOps by following some tips and tricks to run CI/CD pipelines using ARM based approach. 
	The below image shows the files in Azure DevOps along with devops folder. 
	
	![Azure DevOps](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/f6465dfb-28b0-4e26-8146-3987ee9c580f)

	After running the pipeline the following stages were executed :
		![azure Devops 2](https://github.com/geetanjalich/Rebrickable-ingestion/assets/79563879/79c7cca9-9f2a-4c56-aeb4-1e9c9e20905a)

