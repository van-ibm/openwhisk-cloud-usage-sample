# IBM Cloud Usage Samples

This project is a companion to the [Reviewing IBM Cloud services, resources and usage](https://console.bluemix.net/docs/tutorials/cloud-usage.html) tutorial. It implements an API-driven approach to obtain and visualize IBM Cloud usage and billing data.

![](screenshot.png)

The samples are supported by the following IBM Cloud Functions packages:
- [OpenWhisk Cognos Dashboard](https://github.com/IBM-Cloud/openwhisk-cognos-dashboard)
- [OpenWhisk JSONetl](https://github.com/IBM-Cloud/openwhisk-jsonetl)
- [OpenWhisk SQL Query](https://github.com/IBM-Cloud/openwhisk-sql-query)
- [Cloud Object Storage](github.com/ibm-functions/package-cloud-object-storage)

## Before you begin

1. Ensure that you have the appropriate roles to view billing and usage data. For instuctions, see the [Assign permissions](https://console.bluemix.net/docs/tutorials/cloud-usage.html#assign-permissions) section of the tutorial.

2. Clone or download this repository.

3. Download and install [wskdeploy](https://github.com/apache/incubator-openwhisk-wskdeploy/releases/tag/0.9.8-incubating). Add the `wskdeploy` executable to your `PATH`. It will be used to deploy the various artifacts to IBM Cloud Functions.

## Setup

To deploy the application, use the below commands and installation scripts.

1. Using your terminal, change directory to the downloaded repo.

1. Login to IBM Cloud and target your Cloud Foundry account. See [CLI Getting Started](https://console.bluemix.net/docs/cli/reference/bluemix_cli/get_started.html#getting-started). Federated users should add the `--sso` flag.
    ```sh
    ibmcloud login [--sso]
    ```

    ```sh
    ibmcloud target --cf
    ```

2. The following commands create the needed services: `Cloud Object Storage`, `SQL Query` and `Cognos Dashboard Embedded`. If you already have a service deployed, choose the **paid** command. To change the region where these services are deployed, edit the makefile and update the `REGION` variable.
    1. Create `Cognos Dashboard Embedded` using either of the following.
        ```sh
        make create-cognos-lite
        ```

        ```sh
        make create-cognos-paid
        ```
    2. Create `Cloud Object Storage` using either of the following.
        ```sh
        make create-cos-lite
        ```

        ```sh
        make create-cos-paid
        ```
    3. Create `SQL Query` using either of the following.
        ```sh
        make create-sql-lite
        ```

        ```sh
        make create-sql-paid
        ```

3. In your browser, access your IBM Cloud [Dashboard](https://console.bluemix.net/dashboard). There should be three new services that begin with **usage-tutorial**. Access the **usage-tutorial-cos** service instance.

4. Create a new bucket to store usage data.
    - Click the **Create a bucket** button.
    - Select **Cross Region** from the **Resiliency** drop down.
    - Select a location from the **Location**.
    - Provide a bucket **Name** and click **Create** If you receive an *AccessDenied* error, try with a more unique bucket name.

5. Set the location and bucket name as an environment variable to be used later. Examples are `us-geo` and `ibmcloud-usage`.
    ```sh
    export LOCATION=<your location>
    ```

    ```sh
    export BUCKET=<your bucket name>
    ```

6. Back in the [Dashboard](https://console.bluemix.net/dashboard), access the **usage-tutorial-sql** instance.

7. Click the **Instance CRN** button (lower right corner of page) to copy it to the clipboard, and again set an environment variable.
    ```sh
    export INSTANCE_CRN=<your instance crn>
    ```

8. Set an environment variable to capture usage data for a given month. The format is YYYY-MM.
    ```sh
    export MONTH=2018-09
    ```

9. Run the following commands to create a Platform API Key and Authentication Tokens to be used with the application. The API Key will be used with SQL Query. Authentication tokens will be used to make requests to obtain billing and usage data. Run the last command to confirm all have been set.
    ```sh
    export API_KEY=`ibmcloud iam api-key-create usage-tutorial-key -d 'apiKey created for http://github.com/IBM-Cloud/cloud-usage-samples' | grep 'API Key' | awk ' {print $3} '`
    ```

    ```sh
    export IAM_TOKEN=`ibmcloud iam oauth-tokens | head -n 1 | awk ' {print $4} '`
    ```

    ```sh
    export UAA_TOKEN=`ibmcloud iam oauth-tokens | tail -n 1 | awk ' {print $4} '`
    ```

    ```sh
    echo -e API_KEY ' \t ' $API_KEY && echo -e IAM_TOKEN ' \t ' $IAM_TOKEN && echo -e UAA_TOKEN ' \t ' $UAA_TOKEN
    ```

10. Deploy the tutorial package to IBM Cloud Functions. The first command will ensure `wskdeploy` is properly initialized.
    ```sh
    ibmcloud fn package list
    ```

    ```sh
    make deploy
    ```

11. Bind your Cloud Object Storage and Cognos Dashboard Embedded service credentials to the packages.
    ```sh
    make bind-services
    ```

## Invoke OpenWhisk actions

To obtain and process billing and usage data, you'll execute several IBM Cloud Functions sequences.

1. Obtain an account GUID to fetch usage data. The account GUID used should match the **Account:** value seen in the `target` output. Copy the account GUID to the clipboard.
    ```sh
    ibmcloud target && ibmcloud iam accounts
    ```

2. Start the sequence to retrieve and process resource group's usage for the account. You can watch progress using the [Monitor](https://console.bluemix.net/openwhisk/dashboard) dashboard in the IBM Cloud Functions service. The **Actitivity Log** should list actions with a green checkmark.
    ```sh
    ibmcloud fn action invoke tutorial-etl-process/resource-groups-billing -r --param guid <your guid>
    ```

3. After all data has been collected, run the sequence to collect Cloud Foundry usage for the account.
    ```sh
    ibmcloud fn action invoke tutorial-etl-process/cf-orgs-billing -r
    ```

4. Check your Cloud Object Storage bucket, it should now contain files that contain usage data for the various services and resources.

5. Access the **usage-tutorial-sql** services from the [Dashboard](https://console.bluemix.net/dashboard/apps) and click the **Open UI** button.

6. From the command line, run the following sequence. The SQL Query UI will show progress and the result data when complete.
    ```sh
    ibmcloud fn action invoke tutorial-etl-query/billing-job -r
    ```

7. Once the SQL Query job has completed, return to the command line and copy the `job_id` returned from the sequence.

8. Run the following command to get the URL to the billing dashboard. Append the `job_id` value to the URL similar to this example `https://service.us.apiconnect.ibmcloud.com/gws/apigateway/api/<your gateway id>/tutorial/billing?job_id=<your job id>`.

    ```sh
    ibmcloud fn api list | grep 'Tutorial Dashboard'
    ```

9. Visit the dashboard URL in your browser to see a pre-created Cognos Embedded Dashboard with your usage data.

## FAQ

### Why are the Cloud Functions failing?
It's likely that the authentication tokens (UAA and IAM) have expired. Run the following to update the `tutorial-etl-request` package with more recent tokens.

```sh
export IAM_TOKEN=`ibmcloud iam oauth-tokens | head -n 1 | awk ' {print $4} '`
```

```sh
export UAA_TOKEN=`ibmcloud iam oauth-tokens | tail -n 1 | awk ' {print $4} '`
```

```sh
make update-request
```

### How do I uninstall?
Run the following make commands to remove the packages and services.

```sh
make undeploy
```

```sh
make remove-services
```

### My wskdeploy failed with an 'Invalid access token (expired)' error. What do I do?

The following indicates that your `.wskprops` file is stale.

```
Error: servicedeployer.go [1660]: [ERROR_WHISK_CLIENT_ERROR]: Error code: 246: API creation failure: Unable to obtain API(s) from the API Gateway (status code 400): Invalid access token (expired)
```

Delete your `/Users/<username>/.wskprops` file. Then issue the following command to recreate it.

```sh
ibmcloud fn package list
```

### How do I use my existing services rather than deploying new ones?

Edit the makefile's `bind-services` target. Change the `--instance` flag to reflect the names of your existing services.