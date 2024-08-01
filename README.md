# Incremental Data Loading from AWS S3 Bucket to Redshift Using AWS Glue ETL Job

In today's data-driven world, organizations are inundated with vast amounts of data generated at unprecedented speeds. Efficiently managing and processing this data is crucial for maintaining competitive advantage and operational efficiency. One of the most effective techniques to handle large-scale data ingestion is incremental data loading. By focusing only on new or changed data, incremental loading optimizes resource usage and minimizes processing time.

Amazon Web Services (AWS) offers a robust ecosystem to facilitate this process. Combining the storage capabilities of Amazon S3, the data warehousing power of Amazon Redshift, and the transformative capabilities of AWS Glue, you can build a seamless, scalable, and efficient ETL (Extract, Transform, Load) pipeline.

In this blog post, we will guide you through the process of setting up an incremental data-loading pipeline from an AWS S3 bucket to Amazon Redshift using an AWS Glue ETL job. Whether you're a data engineer looking to streamline your data workflows or a business analyst aiming to enhance data accuracy and timeliness, this tutorial will provide you with the knowledge and tools to implement an effective incremental data loading strategy. Join us as we explore the steps, best practices, and key considerations for leveraging AWS services to optimize your data management processes.

## Project Steps for Incremental Data Loading from AWS S3 to Redshift Using AWS Glue

Implementing an incremental data loading pipeline involves several key steps, each critical for ensuring a seamless and efficient data transfer process. Below, we outline these steps, explaining their importance and providing detailed descriptions for each task.

### Step 1: AWS S3 Bucket Creation & Data Upload

**Task**: Data Source Setup  
**Why do we need this?**: Setting up an S3 bucket as the source for data ingestion into Redshift ensures we have a scalable, reliable storage solution for our raw data.

**Detailed Description**:

1. **Access the AWS S3 Console**:
   - Open your AWS Management Console.
   - Navigate to the S3 service by clicking on "S3" under the "Storage" section.

2. **Create a New Bucket**:
   - Click on the "Buckets" section in the left-hand sidebar.
   - Click the "Create bucket" button.
   - Enter a unique name for your bucket. For this project, I named the bucket as `aws-mini-project-demo`.
     - **Note**: The bucket name must be unique across all of AWS and must adhere to the bucket naming rules (e.g., no uppercase letters, no spaces).
   - Leave all other settings at their default values.
   - Ensure the bucket is created in the same Region as other resources you'll be using (Redshift, Glue).

3. **Upload the CSV File**:
   - After creating the bucket, click on the bucket name (`aws-mini-project-demo`) to open it.
   - Click the "Upload" button.
   - Click "Add files" and select the `customer.csv` file from the `DATA-SOURCE-1`.
   - Click "Upload" to start the file upload process.

4. **Verify the Upload**:
   - Once the upload is complete, ensure that the `customer.csv` file is listed in your bucket.
   - This file will serve as the data source for our incremental data-loading process.
  
### Step 2: AWS Redshift Cluster Configuration

**Task**: Data Warehouse Setup  
**Why do we need this?**: Configuring a Redshift cluster, including the creation of workgroups and namespaces, is essential for organizing and managing data effectively in a scalable data warehouse.

**Detailed Description**:

1. **Access the AWS Redshift Console**:
   - Open your AWS Management Console.
   - Navigate to the Redshift service by clicking on "Redshift" under the "Analytics" section.

2. **Create a Workgroup**:
   - Go to the Serverless Dashboard.
   - Click on "Create Workgroup".
   - Enter a unique name for your workgroup. For this project, name it `aws-mini-project`.
   - Leave the Capacity part as the default value (128 RPU).

3. **Network and Security Configuration**:
   - In the "Network and Security" section, choose the default VPC for your account.
   - Select the default VPC security group.
   - Enable "Enhanced VPC Routing" by checking the box.

4. **Create a Namespace**:
   - Choose the option to create a new namespace.
   - Enter a name for your namespace. For this project, I named it as `my-namespace`.

5. **Customize Admin User Credentials**:
   - In the "Database name and password" section, click on "Customize admin user credentials".
   - Enter an admin username and password by selecting "Manually add the admin password".

6. **Create the Workgroup and Namespace**:
   - Click "Create" to set up your workgroup and namespace.
   - Wait for the workgroup status to change to "Available".

7. **Important Configuration for Default VPC**:
   - Once you create a Redshift Cluster, it will automatically assign an endpoint to the default VPC.
   - Navigate to the AWS VPC console and go to the "Endpoints" section.
   - **Create an Endpoint for the S3 Bucket**:
     - Click on "Create endpoint".
     - Give a Name tag, such as `my-s3-endpoint`. This endpoint is necessary to enable communication between the Redshift cluster and the S3 bucket for data loading operations.
     - Choose "AWS services" and from the services list, search for S3 and select the one with the type "Gateway".
     - Choose the default VPC in the VPC part.
     - Select the appropriate Route Table.

8. **Configure Security Group**:
   - After configuring the endpoint, go to the "Security Group" section under the VPC console.
   - Find your default-VPC security group.
   - From the "Actions" menu, click "Edit inbound rules" and ensure you have the following configuration:
     - **Type**: All TCP
     - **Source**: Your VPC Security Group ID (choose the security group of your default-VPC).

9. **Run SQL Queries to Create a Table**:
   - After the workgroup is available, navigate to the Redshift query editor.
   - Click on "Query data" to open the query editor in a new tab.

10. **Create the Employee Table**:
    - Copy and paste the following SQL query into the query editor:

      ```sql
      CREATE TABLE employee (
          customer_id int,
          first_name VARCHAR(255),
          last_name VARCHAR(255)
      );
      ```

    - Run the query to create the `employee` table under the `dev` schema in the `public` schema.

11. **Verify Table Creation**:
    - Right-click the newly created `employee` table and select the "Select Table" action.
    - Run the query that pops up in the editor to verify that the table exists and currently contains no data.

    **Note**: The table is now ready to receive data from the S3 bucket, which we will handle in subsequent steps.

### Step 3: IAM Role Creation for Glue

**Task**: Access Control  
**Why do we need this?**: Creating an IAM role is essential to grant AWS Glue the necessary permissions to securely access and manipulate S3 and Redshift resources.

**Detailed Description**:

1. **Access the AWS IAM Console**:
   - Open your AWS Management Console.
   - Navigate to the IAM service by clicking on "IAM" under the "Security, Identity, & Compliance" section.

2. **Create a New IAM Role**:
   - Under "Access Management", find and click on "Roles".
   - Click on the "Create role" button.
   - For the trusted entity type, select "AWS Service".
   - Choose the use case type as "Glue" and select it.

3. **Assign Permissions to the Role**:
   - Add the following permission policies to the role:
     - **AdministratorAccess**
     - **AmazonS3FullAccess**
     - **AWSLambda_FullAccess**
     - **CloudWatchFullAccess**
   - **Note**: While these policies grant full access to necessary AWS services, it's important to follow security best practices by implementing more granular permissions tailored to specific scenarios, ensuring the principle of least privilege to minimize security risks.

4. **Name the Role**:
   - Enter a unique name for your IAM role. For this project, name it `GLUE-ACCESS-S3-RDS-REDSHIFT-LAMBDA-FULLACCESS`.

5. **Edit the Trust Relationship**:
   - After creating the role, navigate to the "Trust relationships" tab.
   - Click "Edit trust policy".
   - Copy and paste the following JSON policy, replacing `***Your Account ID***` with your actual AWS account ID:

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Principal": {
             "AWS": "arn:aws:iam::***Your Account ID***:root",
             "Service": [
               "textract.amazonaws.com",
               "glue.amazonaws.com",
               "redshift.amazonaws.com",
               "datapipeline.amazonaws.com",
               "lakeformation.amazonaws.com",
               "cloudformation.amazonaws.com",
               "transfer.amazonaws.com",
               "dms.amazonaws.com",
               "ecs.amazonaws.com",
               "delivery.logs.amazonaws.com",
               "rds.amazonaws.com",
               "ecs-tasks.amazonaws.com",
               "events.amazonaws.com",
               "apigateway.amazonaws.com",
               "logs.amazonaws.com"
             ]
           },
           "Action": "sts:AssumeRole"
         }
       ]
     }
     ```

   - Ensure you replace `***Your Account ID***` with your actual AWS account ID in the `"AWS": "arn:aws:iam::***Your Account ID***:root"` line.

6. **Save the Trust Policy**:
   - Click "Update Trust Policy" to save the changes.

### Step 4: AWS Glue Configuration, ETL Job, Crawler, Connection & Database

**Task**: ETL Automation  
**Why do we need this?**: Configuring AWS Glue components automates the ETL process, allowing for efficient data transformation and loading between AWS S3 and Redshift.

**Detailed Description**:

1. **Create a Database in AWS Glue**:
   - Go to the AWS Glue console.
   - Navigate to the "Databases" section.
   - Click "Add database" and enter a name. For this project, name it `my-database-sateesh`.
   - Click "Create database".

2. **Create a Crawler**:
   - Navigate to the "Crawlers" section and click "Add crawler".
   - Enter a unique name for your crawler, such as `my-s3-crawler`.
   - Click "Next".

3. **Configure Data Source for the Crawler**:
   - In the "Add a data source" page, click "Add data source".
   - Select the S3 bucket and folder containing your data. For this project, select the `aws-mini-project-demo` bucket and the `customer.csv` file.

4. **Configure IAM Role for the Crawler**:
   - On the next page, select the IAM role created earlier (`GLUE-ACCESS-S3-RDS-REDSHIFT-LAMBDA-FULLACCESS`).
   - Click "Next".

5. **Set Output and Scheduling for the Crawler**:
   - Choose the target database (`my-database-sateesh`).
   - Enter a table name prefix, such as `sateesh_`.
   - Click "Next" and then "Finish".

6. **Run the Crawler**:
   - After creating the crawler, click on it and select "Run crawler".
   - The crawler will scan the S3 bucket and create a table in the Glue Data Catalog.

7. **Create a Connection between Glue and Redshift**:
   - Go to the "Connections" tab in the Glue console.
   - Click "Add connection" and select "Data Source" -> "Amazon Redshift".
   - Configure the connection:
     - Select your workgroup (`my-workgroup`).
     - Use the Redshift admin credentials you created earlier.
     - Name the connection, for example, `Redshift-connection`.
   - After creation, test the connection:
     - Click on the connection, select "Actions" -> "Test connection".
     - Choose the IAM role (`GLUE-ACCESS-S3-RDS-REDSHIFT-LAMBDA-FULLACCESS`) and run the test to ensure it is configured correctly.

8. **Create an ETL Job Using the Visual ETL Tool**:
   - Go to "ETL Jobs" -> "Visual ETL" in the Glue console.
   - Configure the ETL job:
     - **Sources**: Select "Amazon S3".
       - Name: Default
       - S3 source type: Data Catalog Table
       - Database: `my-database-sateesh`
       - Table: `sateesh_aws_mini_project_demo`
     - **Targets**: Select "Amazon Redshift".
       - Name: `Amazon Redshift`
       - Node parents: `Amazon S3`
       - Redshift access type: Direct data connection
       - Redshift connection: `Redshift-connection`
       - Schema: `public`
       - Table: `employee`
       - Handling of data and target table: Choose "MERGE data into target table"
       - Matching keys: `customer_id`
       - Save the job and name it, for example, `ETL_job`.

9. **Run the ETL Job**:
   - Go back to "ETL Jobs" and select the created job.
   - Click "Run job".
   - Monitor the job in the Job Run Monitor under the Glue dashboard to ensure it completes successfully.

10. **Verify Data in Redshift**:
    - Go to the Redshift Query Editor.
    - Select the `employee` table and run a query to verify that the data has been loaded.
    - The table should now contain data from the `customer.csv` file in the S3 bucket.

11. **Update the CSV File and Re-Run the ETL Job**:
    - Update the `customer.csv` file with new rows (you can find the updated version in the `DATA-2` folder).
    - Upload the updated `customer.csv` file to the S3 bucket (`aws-mini-project-demo`).
    - Re-run the crawler to update the Glue Data Catalog.
    - Re-run the ETL job to load the new data into Redshift.
    - Verify the updated data in the Redshift Query Editor by running the query again.

By following these steps, you have successfully configured AWS Glue, set up an ETL job, and established a connection between S3 and Redshift. You can now efficiently manage and load data incrementally into your Redshift cluster.

---

This comprehensive guide integrates all the necessary steps to set up an incremental data-loading pipeline using AWS S3, Redshift, and Glue. The details provided ensure that each component is correctly configured and that data can be seamlessly transferred and updated.
