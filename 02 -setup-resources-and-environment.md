In this lab, you will create and configure fabric resource cluster, API management resource and bot channel registration resource.

**Exercise 1: Create a Resource Group**

1. Sign in to the  [Azure portal](https://portal.azure.com/).
2. To see all the resource groups in your subscription, select  **Resource groups**

Image 1

3. To create an empty resource group, select  **Add**.

Image 2

4. Provide Resource Group Name as &quot;OneBankRG&quot;, Resource Group Location as &quot;West US&quot; and Click **Create**.

Image 3

**Exercise 2: Create a new Azure API Management service instance**

1. In the  [Azure portal](https://portal.azure.com/), select  **Create a resource**  &gt;  **Enterprise Integration**  &gt;  **API management**.
2. In the  **API Management service**  window, enter settings. **Choose ** Create**.

Image 4

3. Once the API Management is deployed, Copy the Developer Portal URL (to be sued in Bot channel registration)

Image 5

**Exercise 3: Deploy Service Fabric Cluster**

1. Click Create a resource to add a new resource template.

Image 6

2. Search for the Service Fabric Cluster template in the Marketplace under Everything.

 Image 7

3. Select Service Fabric Cluster from the list.

Image 8

4. Navigate to the Service Fabric Cluster blade, click Create,
5. The Create Service Fabric cluster blade has the following four steps:
  **a.**** Basics** - In the Basics blade, you need to provide the basic details for your cluster.
    1. Enter the name of your cluster as &quot;onebank-fabric-cluster&quot;
    2. Enter a user name and password for Remote Desktop for the VMs.
    3. Make sure to select the Subscription that you want your cluster to be deployed to, especially if you have multiple subscriptions.
    4. Select the Resource Group created in first step.
    5. Select the region in which you want to create the cluster as &quot;West US 2&quot;

Image 9

**b.**** Cluster configuration**

Image 10

    1. Choose a name for your node type as &quot;Node 0&quot;
    2. The minimum size of VMs for the primary node type is driven by the durability tier you choose for the cluster. The default for the durability tier is bronze.
    3. Select the VM size as &quot;Standards\_D1\_v2&quot;
    4. Choose the number of VMs for the node type as 1
    5. Select Three node clusters
    6. Configure custom endpoints with 80,8770

Image 11

**c.**** Security**
    1. Select Basic in Security Configuration settings. To make setting up a secure test cluster easy for you, we have provided the  **Basic**  option.  Click on Key Vault for configuring required settings. Click on Create a new vault

Image 12

    2. Create a key Vault with given values and Click on Create

Image 13

    3. Now that the certificate is added to your key vault, you may see the following screen prompting you to edit the access policies for your Key vault. click on the Edit access **policies for.**  button.

Image 14

    4. Click on the advanced access policies and enable access to the Virtual Machines for deployment. It is recommended that you enable the template deployment as well. Once you have made your selections, do not forget to click the  **Save**  button and close out of the  **Access policies**  pane.

Image 15

Image 16

    5. You are now ready to proceed to the rest of the create cluster process.

**d.**** Summary**
    1. Now you are ready to deploy the cluster. Before you do that, download the certificate, look inside the large blue informational box for the link. Make sure to keep the cert in a safe place. you need it to connect to your cluster. Since the certificate you downloaded does not have a password, it is advised that you add one.
    2. To complete the cluster creation, click  **Create**. You can optionally download the template.

Image 17

6. You can see the creation progress in the notifications.

Image 18

7. Install the downloaded certificate from previous step.
8. Select Current User and Click Next

Image 19

9. Click Next. Leave Password Blank. Select Next

Image 20

10. Click on Next and Finish the Installation.

**Exercise 4: Configure Azure API Management service**

**Task I: Create and publish a product**

**a.** Click on Products in the menu on the left to display the Products page.

Image 21

**b.** Click  **+ Product**.

Image 22

**c.** When you add a product, supply the following information:
    1. **i.** Display name
    2. **ii.** Name
    3. **iii.** Description
    4. **iv.** State as Published
    5. **v.** Requires subscription - Uncheck Require subscription checkbox
    Click Create to create the new product.

Image 23

**Task II: Add APIs to a product**

**a.** Select APIs from under API MANAGEMENT.

Image 24

**b.** Select  **Blank API**  from the list.

Image 25

**c.** Enter settings for the API.
    1. **i.** Display name
    2. **ii.** Web Service URL – Fabric cluster end point. Update the port as 8770 and suffix with /api
    3. **iii.** URL suffix as api
    4. **iv.** Products – Select the Product created form previous step

**d.** Select Create.

Image 26

**Task III: Add the operation**

**a.** Select the API you created in the previous step.
**b.** Click + Add Operation.

Image 27

**c.** In the URL, select POST and enter &quot;/messages&quot; in the resource.
**d.** Enter &quot;Post /messages&quot; for Display name.
**e.** Select Save

Image 28

**Exercise 5: Create a Bot Channels Registration**

1. Click the New button found on the upper left-hand corner of the Azure portal, then select AI + Cognitive Services &gt; Bot Channels Registration.
2. A new blade will open with information about the Bot Channels Registration. Click the Create button to start the creation process.
3. In the Bot Service blade, provide the requested information about your bot as specified in the table below the image.

Image 29

4. Click  **Create**  to create the service and register your bot&#39;s messaging end point.
5. Bot Channels Registration -** bot service does not have an app service associated with it. Because of that, this bot service only has a _MicrosoftAppID_. You need to generate the password manually and save it yourself.
   **a.** From the Settings blade, click Manage. This is the link appearing by the Microsoft App ID. This link will open a window where you can generate a new password.

Image 30

   **b.** Click  **Generate New Password**. This will generate a new password for your bot. Copy this password and save it to a file. This is the only time you will see this password. If you do not have the full password saved, you will need to repeat the process to create a new password should you need it later.

Image 31

   **c.** Click on Save at the end of the page. Close the page. In portal, Click on Save in Settings blade.

Image 32
