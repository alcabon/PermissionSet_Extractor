# **Comprehensive Guide to Retrieving Salesforce Permission Set Data via Java WSC with Multithreading**

## **1\. Introduction**

This report provides an expert technical guide for developing a Java application that utilizes the Salesforce Web Service Connector (WSC). The primary objective of this application is to perform a comprehensive, multithreaded retrieval of Permission Set data from a Salesforce organization, authenticating via a pre-existing OAuth access token.

Permission Sets are a fundamental component of the Salesforce security model, offering a flexible and granular mechanism to extend user capabilities beyond those defined in their profiles. A thorough understanding of their complete configuration is paramount for effective security audits, compliance verification, and the administration of complex Salesforce environments. The task of programmatically extracting this information presents several challenges, including the intricate nature of the Salesforce permission SObject model, the need for efficient extraction of potentially large data volumes, and strict adherence to Salesforce API governance limits.

The solution detailed herein adopts a two-phase data retrieval strategy. Initially, core Permission Set attributes are fetched. Subsequently, detailed permission assignments—spanning object permissions, field-level security, user and system permissions, Apex class and Visualforce page access, custom permission assignments, application visibility, tab settings, record type visibilities, and connected application access—are concurrently gathered for each identified Permission Set using Java's multithreading capabilities. The Salesforce Object Query Language (SOQL) serves as the principal mechanism for data extraction.1 The utility of SOQL for administrators and developers in complex data extraction scenarios, extending beyond simple record viewing, is well-documented and aligns with the comprehensive data retrieval goals of this application.1

The extensive nature of the permission data requested—covering nearly all facets of a Permission Set's definition—indicates that the intended application likely serves as a backend for an in-depth security analysis tool, an auditing utility, or a system to support large-scale permission management and refactoring efforts. This context influences design considerations, emphasizing the need for well-structured data output and robust error handling to manage the processing of individual permission sets within a larger batch.

Furthermore, the stipulation that authentication will occur via a "token provided for the connexion" suggests that the Java application will operate as a component within a broader system where the OAuth authentication flow is managed externally. This simplifies the WSC setup concerning authentication initiation but places the onus of token acquisition and lifecycle management (e.g., refresh) on the calling system or process. The application must, therefore, be designed to consume this token and the associated Salesforce instance URL, rather than initiating an OAuth flow itself.

## **2\. Establishing Salesforce Connection with Java WSC and Access Token**

Interaction with Salesforce SOAP APIs from a Java environment is significantly streamlined by the Salesforce Web Service Connector (WSC). WSC is a Salesforce-provided library that handles the complexities of SOAP message generation and response parsing, offering client-side stub generation from Web Services Description Language (WSDL) files.

### **Core Configuration: com.sforce.ws.ConnectorConfig**

The cornerstone of WSC connection setup is the com.sforce.ws.ConnectorConfig class. An instance of this class encapsulates all necessary parameters for establishing a connection, including authentication details and the target service endpoint.

### **Token-Based Authentication**

For applications authenticating with a pre-existing OAuth access token, the following ConnectorConfig settings are paramount:

1. **Setting the Session ID**: The OAuth access token obtained from an external authentication process must be supplied to WSC as the session ID. This is achieved using the connectorConfig.setSessionId(String sessionId) method.3 It is critical to recognize that, in this OAuth context, the access token functions as the session identifier for WSC.  
2. **Setting the Service Endpoint**: When setSessionId() is used, WSC bypasses its standard login call, which would typically discover the Salesforce instance-specific service endpoint. Therefore, this endpoint URL must be explicitly provided using connectorConfig.setServiceEndpoint(String serviceEndpointUrl).4

### **Determining and Constructing the Correct Service Endpoint**

The service endpoint URL is unique to the Salesforce instance hosting the target organization. This URL is typically a part of the OAuth token response (often named instance\_url or a similar parameter). The Java application must be designed to receive this instance\_url alongside the access token.

* **Standard SOAP API (Partner/Enterprise WSDL)**: The service endpoint for standard API interactions (e.g., querying PermissionSet, ObjectPermissions) follows the format: https://\<YourInstanceNameFromInstanceUrl\>/services/Soap/u/\<APIVersion\>. For example, if the instance\_url is https://yourorg.my.salesforce.com, the endpoint for API version 58.0 would be https://yourorg.my.salesforce.com/services/Soap/u/58.0.  
* **Tooling API**: For accessing metadata components like PermissionSetTabSetting via the Tooling API, the endpoint format changes to: https://\<YourInstanceNameFromInstanceUrl\>/services/Soap/T/\<APIVersion\>.5 The transformation from the standard API endpoint often involves replacing the /u/ segment with /T/.5

### **Instantiating the Connection**

Once the ConnectorConfig object is appropriately populated, a connection object can be instantiated. For interactions using the Partner WSDL (covering most SObject data):  
PartnerConnection partnerConnection \= com.sforce.soap.partner.Connector.newConnection(connectorConfig);  
For Tooling API interactions, a similar approach is used with the Tooling WSDL-generated stubs and a ConnectorConfig instance pointing to the Tooling API service endpoint:  
com.sforce.soap.tooling.SoapConnection toolingConnection \= com.sforce.soap.tooling.Connector.newConnection(toolingConfig);  
The successful establishment of this connection relies on the validity of the externally provided access token and instance URL. Should the token be expired, revoked, or otherwise invalid, Salesforce will reject API calls, and WSC will typically throw an InvalidSessionFault or a similar ConnectionException. As the application receives the token externally and lacks primary credentials (username/password), it cannot independently re-authenticate or refresh the session in the way a WSC client performing a full login could.40 Consequently, the application's error handling strategy must be capable of detecting such authentication failures and clearly signaling to the calling system or operator that a new, valid token is required.

The requirement to retrieve a comprehensive set of permission data means the application might need to interact with both the standard SOAP API (for objects like PermissionSet, ObjectPermissions) and the Tooling API (for objects like PermissionSetTabSetting). Given that these APIs utilize different service endpoints, the application architecture must accommodate this. A clean approach involves instantiating and managing separate Connection objects (e.g., a PartnerConnection and a com.sforce.soap.tooling.SoapConnection), each initialized with its own ConnectorConfig instance. Both ConnectorConfig instances would use the same sessionId (the provided access token) but would be configured with their respective service endpoints (/u/ for standard API, /T/ for Tooling API). This allows the application to direct queries to the appropriate API seamlessly.

## **3\. Phase 1: Retrieving Core Permission Set Information (Setup Data)**

The initial phase of data retrieval focuses on gathering the fundamental attributes of all relevant permission sets. This includes their identifying information, type, and, significantly, all directly associated "User Permissions" (often referred to as System or App Permissions).

**Target SObject**: PermissionSet

This SObject is the central entity representing a permission set in Salesforce. It holds not only the definition of the permission set itself but also a multitude of boolean fields that correspond to various system-level and application-level permissions.

**Key Queryable Fields on PermissionSet**:

* Id: The unique 18-character Salesforce ID. This is essential for linking to detailed permission assignments in Phase 2\.  
* Name: The developer name (API name) of the permission set, used for programmatic reference.  
* Label: The human-readable label displayed in the Salesforce user interface.  
* Description: An optional text field for describing the purpose of the permission set.  
* IsCustom: A boolean field indicating whether the permission set was created by an administrator (true) or is a standard permission set provided by Salesforce or a managed package (false).6  
* IsOwnedByProfile: A critical boolean field. If true, this PermissionSet record is an internal representation of a Profile's permissions.7 Filtering on this field is necessary to isolate "true" permission sets.  
* LicenseId: A lookup field to the PermissionSetLicense object. This field is populated if the permission set is specifically associated with a permission set license, granting users access to features not included in their base user license.9  
* NamespacePrefix: A string field that contains the namespace prefix if the permission set is a component of a managed package.  
* Type: A picklist field that denotes the type of permission set, such as Regular, SessionActivated (for session-based permission sets), or Group (indicating the record represents a Permission Set Group).  
* ProfileId: A lookup to the Profile object. This field is populated if IsOwnedByProfile is true, linking the permission set record to its corresponding profile.7  
* **User Permissions / System Permissions**: The PermissionSet object contains a large number of boolean fields, typically prefixed with Permissions\* (e.g., PermissionsViewAllData, PermissionsApiEnabled, PermissionsAuthorApex, PermissionsManageUsers). Each of these fields corresponds to a specific user permission checkbox found in the "System Permissions" or "App Permissions" sections when configuring a permission set in the Salesforce UI.11

**SOQL Query Strategy for Phase 1**:

A single, comprehensive SOQL query can be constructed to retrieve all desired PermissionSet records along with their directly associated user permissions. A crucial element of this query is the WHERE clause.

A critical aspect of accurately retrieving permission set data is the utilization of the IsOwnedByProfile field on the PermissionSet object. Salesforce's internal data model also uses PermissionSet records to represent the permissions inherent in a Profile.7 Consequently, querying the PermissionSet object without filtering on IsOwnedByProfile \= false would result in the inclusion of data for every Profile, potentially misrepresenting the information as distinct permission sets and significantly increasing the data volume to be processed. Therefore, this filter is essential to align with the objective of analyzing standalone permission sets.

The following example illustrates the query structure:

SQL

SELECT Id, Name, Label, Description, IsCustom, IsOwnedByProfile, Type, NamespacePrefix, LicenseId, Profile.Name,  
       PermissionsViewAllData, PermissionsModifyAllData, PermissionsApiEnabled, PermissionsAuthorApex,  
       PermissionsActivateContract, PermissionsAddDirectMessageMembers, PermissionsApexRestServices,  
       /\*... include all other relevant Permissions\* fields... \*/  
       PermissionsEditReadonlyFields, PermissionsInstallPackaging, PermissionsManageUsers, PermissionsTransferAnyEntity  
FROM PermissionSet  
WHERE IsOwnedByProfile \= false

This query structure is adapted from examples found in Salesforce documentation and community discussions.6 The extensive list of Permissions\* fields can be derived from sources like 11 and 13, which map API names to UI labels.

The inclusion of all Permissions\* fields directly in this Phase 1 query is an efficient approach. It ensures that a substantial portion of a permission set's definition—specifically, all its user and system-level permissions—is retrieved in one operation per permission set. This optimizes the initial data gathering and allows Phase 2 to focus on permissions defined through related SObjects (like object and field permissions).

The Type field on the PermissionSet object also warrants attention. It can differentiate between regular permission sets, session-based permission sets, and importantly, permission set groups (which have Type \= 'Group'). While the user's request is for "permission sets," Permission Set Groups are a significant part of the Salesforce permission model. If a record with Type \= 'Group' is retrieved, its directly associated Permissions\* fields might not represent the final effective permissions for users assigned that group, as these are a calculated aggregate of included sets and muted permissions. For the scope of retrieving individual permission set information, it may be sufficient to note this distinction or, if only regular permission sets are desired, to add Type \= 'Regular' to the WHERE clause, though the request for "maximum information" suggests including all non-profile-owned types.

## **4\. Phase 2: Retrieving Granular Permission Set Details**

Once the core PermissionSet records are retrieved in Phase 1, Phase 2 involves fetching the detailed, granular permission assignments associated with each of these permission sets. This typically involves querying several related SObjects. The data retrieval for each PermissionSet.Id can be parallelized using the multithreading strategy discussed in Section 5\.

To provide a clear map for this phase, Table 1 summarizes the key Salesforce SObjects involved in storing various permission details.

**Table 1: Key Salesforce SObjects for Permission Data Retrieval**

| SObject API Name | Key Queryable Fields for Permissions | Purpose in Permission Context | Primary API (SOAP/Tooling) |
| :---- | :---- | :---- | :---- |
| PermissionSet | Id, Name, Label, IsOwnedByProfile, Type, Permissions\* (various boolean fields) | Core definition and user/system permissions | SOAP |
| ObjectPermissions | ParentId, SobjectType, PermissionsCreate, PermissionsRead, PermissionsEdit, PermissionsDelete, PermissionsViewAllRecords, PermissionsModifyAllRecords | Object-level CRUD and data access | SOAP |
| FieldPermissions | ParentId, SobjectType, Field, PermissionsRead, PermissionsEdit | Field-Level Security (FLS) | SOAP |
| SetupEntityAccess | ParentId, SetupEntityId, SetupEntityType | Access to Apex, VF, Custom Perms, Connected Apps, etc. | SOAP |
| PermissionSetTabSetting | ParentId, Name, Visibility | Custom Tab visibility | Tooling |
| ApexClass | Id, Name, NamespacePrefix | To get names/details for Apex class access | SOAP |
| ApexPage | Id, Name, NamespacePrefix | To get names/details for Visualforce page access | SOAP |
| CustomPermission | Id, DeveloperName, MasterLabel | To get names/details for custom permission assignments | SOAP |
| ConnectedApplication | Id, Name, Label | To get names/details for connected app access | SOAP |
| ServicePresenceStatus | Id, MasterLabel, DeveloperName | To get names/details for service presence status access | SOAP |
| AppMenuItem | Id, ApplicationId, Label, Name, Type | To resolve application visibility from SetupEntityAccess (TabSet) | SOAP |
| CustomApplication | Id, DeveloperName, MasterLabel | To resolve application visibility from SetupEntityAccess (CustomApplication) | Tooling/SOAP |

This table serves as a quick reference, guiding the developer in targeting the correct SObjects and APIs for each type of permission data.

### **4.1. Object Permissions**

Object permissions define the fundamental access rights (Create, Read, Edit, Delete, View All, Modify All) a user has to records of a specific SObject, as granted by a particular permission set.

* **Target SObject**: ObjectPermissions  
* **Key Queryable Fields**:  
  * Id: Unique identifier of the ObjectPermissions record.  
  * ParentId: Foreign key to PermissionSet.Id, linking this record to the parent permission set. This is the primary filter field.  
  * SobjectType: The API name of the SObject (e.g., Account, MyCustomObject\_\_c) for which permissions are defined.14  
  * PermissionsCreate: Boolean, allows record creation.  
  * PermissionsRead: Boolean, allows record viewing.  
  * PermissionsEdit: Boolean, allows record modification.  
  * PermissionsDelete: Boolean, allows record deletion.  
  * PermissionsViewAllRecords: Boolean, grants view access to all records of the SobjectType, irrespective of sharing rules.  
  * PermissionsModifyAllRecords: Boolean, grants read, edit, and delete access to all records of the SobjectType, irrespective of sharing rules. 14  
* **SOQL Query Strategy**: For each PermissionSet.Id from Phase 1, query ObjectPermissions:  
  SQL  
  SELECT SobjectType, PermissionsCreate, PermissionsRead, PermissionsEdit, PermissionsDelete,  
         PermissionsViewAllRecords, PermissionsModifyAllRecords  
  FROM ObjectPermissions  
  WHERE ParentId \= :currentPermissionSetId  
  8

It is important to note that ObjectPermissions records only exist if at least one of the boolean permission flags is set to true for a given PermissionSet and SobjectType combination.14 The absence of an ObjectPermissions record for a specific SObject under a permission set implies that the permission set grants no explicit object-level permissions for that SObject; access would then be determined by the user's profile or other assigned permission sets.

### **4.2. Field Permissions**

Field-Level Security (FLS) is managed through the FieldPermissions object, which specifies read and edit access to individual fields within an SObject, as granted by a permission set.

* **Target SObject**: FieldPermissions  
* **Key Queryable Fields**:  
  * Id: Unique identifier of the FieldPermissions record.  
  * ParentId: Foreign key to PermissionSet.Id.  
  * SobjectType: The API name of the SObject containing the field.  
  * Field: The API name of the field itself. This is often in the format SobjectType.FieldName (e.g., Account.Industry).16  
  * PermissionsRead: Boolean, grants read access to the field.  
  * PermissionsEdit: Boolean, grants edit access to the field. 6  
* **SOQL Query Strategy**: For each PermissionSet.Id, query FieldPermissions:  
  SQL  
  SELECT SobjectType, Field, PermissionsRead, PermissionsEdit  
  FROM FieldPermissions  
  WHERE ParentId \= :currentPermissionSetId  
  6

Similar to ObjectPermissions, FieldPermissions records are typically created only when a permission set explicitly grants read or edit access to a field. If no record exists for a particular field under a permission set, that permission set does not alter the FLS for that field; access is then governed by the profile or other permission settings.

### **4.3. User Permissions (System & App Permissions)**

As detailed in Phase 1 (Section 3), user permissions (also known as system permissions or app permissions) are represented by numerous boolean fields directly on the PermissionSet SObject itself. These fields typically have names starting with Permissions\*, such as PermissionsApiEnabled, PermissionsViewAllData, PermissionsAuthorApex, and PermissionsManageUsers.

Since these were retrieved as part of the Phase 1 query for the PermissionSet object, no additional queries are needed in Phase 2 for this specific category of permissions. The data collected in Phase 1 for these fields should be incorporated into the comprehensive data structure for each permission set. Resources like 11 and 13 provide valuable mappings from these API field names to their user-friendly UI labels, which will be essential for displaying the information meaningfully.

### **4.4. Setup Entity Access (Apex Classes, Visualforce Pages, Custom Permissions, Connected Apps, Service Presence Statuses)**

Access to various setup entities like Apex classes, Visualforce pages, custom permissions, connected applications, and service presence statuses is controlled via the SetupEntityAccess object. This object acts as a junction, linking a PermissionSet (via ParentId) to a specific setup entity (via SetupEntityId) of a defined type (SetupEntityType).

* **Target SObject**: SetupEntityAccess 18  
* **Key Queryable Fields**:  
  * Id: Unique identifier of the SetupEntityAccess record.  
  * ParentId: Foreign key to PermissionSet.Id.  
  * SetupEntityId: The ID of the specific setup entity being permissioned (e.g., an ApexClass.Id, ApexPage.Id, CustomPermission.Id).  
  * SetupEntityType: A string picklist value that defines the type of the setup entity.  
* **SOQL Query Strategy (Initial Query)**: For each PermissionSet.Id:  
  SQL  
  SELECT SetupEntityId, SetupEntityType  
  FROM SetupEntityAccess  
  WHERE ParentId \= :currentPermissionSetId  
  18

This initial query retrieves all SetupEntityAccess records for a given permission set. However, SetupEntityId is just an ID. To obtain human-readable names (e.g., the name of an Apex class), secondary queries are required based on the SetupEntityType. This two-step process is a prime candidate for optimization using batched secondary lookups, as discussed in Section 7\.

**Table 2: Common SetupEntityType Values in SetupEntityAccess and Corresponding Entity Details**

| SetupEntityType Value | Corresponding Salesforce Entity | SObject to Query for Name/Label | Key Field(s) in SObject for Name/Label | Supporting Snippets |
| :---- | :---- | :---- | :---- | :---- |
| ApexClass | Apex Class | ApexClass | Name, NamespacePrefix | 18 |
| ApexPage | Visualforce Page | ApexPage | Name, NamespacePrefix | 20 |
| CustomPermission | Custom Permission | CustomPermission | DeveloperName, MasterLabel | 23 |
| ConnectedApplication | Connected Application | ConnectedApplication | Name, Label | 12 |
| ServicePresenceStatus | Service Presence Status | ServicePresenceStatus | MasterLabel, DeveloperName | 26 |
| TabSet | Application (App) | AppMenuItem | Label, Name (related to ApplicationId) | 28 |
| CustomApplication | Custom Application (alternative for App) | CustomApplication (Tooling) / AppDefinition (SOAP) | DeveloperName or MasterLabel | 28 |

Secondary Query Example (for ApexClass access):  
If a SetupEntityAccess record has SetupEntityType \= 'ApexClass', a subsequent query would be:

SQL

SELECT Name, NamespacePrefix FROM ApexClass WHERE Id \= :setupEntityIdFromPreviousQuery

Similar queries would be constructed for other SetupEntityType values, targeting the SObjects listed in Table 2\.

The SetupEntityType picklist is not always exhaustively documented in a single official source in the provided materials.29 Community-driven lists 28 or context from specific feature documentation often provide these values. The application should be designed to handle known types and potentially log or flag unknown SetupEntityType values encountered, as Salesforce may introduce new permissionable entities over time.

### **4.5. Application Visibility**

Determining which applications (standard or custom) are made visible through a permission set can be approached via SetupEntityAccess.

* **SetupEntityType Values**: 'TabSet' (often representing an application in this context) or 'CustomApplication'.28  
* **Resolving SetupEntityId**:  
  * If SetupEntityType \= 'TabSet', the SetupEntityId typically corresponds to an AppMenuItem.Id. A query on AppMenuItem can then yield the application's label or name. The AppMenuItem.ApplicationId can further link to an AppDefinition for more details if needed.  
  * If SetupEntityType \= 'CustomApplication', the SetupEntityId would be the ID of the CustomApplication (Tooling API object) or AppDefinition (SOAP API object).  
* **SOQL Query Strategy**:  
  1. Query SetupEntityAccess:  
     SQL  
     SELECT SetupEntityId, SetupEntityType  
     FROM SetupEntityAccess  
     WHERE ParentId \= :currentPermissionSetId AND SetupEntityType IN ('TabSet', 'CustomApplication')

  2. Based on the SetupEntityType and SetupEntityId retrieved, query AppMenuItem or CustomApplication/AppDefinition for application names/labels.

It's worth noting that while the Metadata API offers a direct applicationVisibilities component on the PermissionSet type 31, achieving the same comprehensiveness via SOQL alone can be more complex. Some community discussions suggest direct SOQL for app assignments might have limitations compared to the Metadata API.32 While the SetupEntityAccess approach is the most viable SOQL path, its completeness should be validated against org configurations.

### **4.6. Custom Tab Settings**

Visibility of custom tabs within a permission set is managed by the PermissionSetTabSetting object, which is accessible via the Tooling API.33

* **Target SObject (Tooling API)**: PermissionSetTabSetting  
* **Key Queryable Fields**:  
  * Id: Unique identifier of the PermissionSetTabSetting record.  
  * ParentId: Foreign key to PermissionSet.Id.  
  * Name: The API name or ID of the CustomTabDefinition (e.g., MyCustomTab\_\_c).  
  * Visibility: A picklist field indicating the tab's visibility (e.g., Visible, Available, None).  
* **Accessing via WSC**: This requires configuring the WSC ConnectorConfig to use the Tooling API service endpoint (e.g., https://\<YourInstanceName\>/services/Soap/T/\<APIVersion\>).5  
* **SOQL Query Strategy (Tooling API)**: For each PermissionSet.Id:  
  SQL  
  SELECT Name, Visibility  
  FROM PermissionSetTabSetting  
  WHERE ParentId \= :currentPermissionSetId  
  34

This is a clear instance where the application must switch its WSC connection context to the Tooling API to retrieve the desired information, reinforcing the need for managing multiple API endpoint configurations.

### **4.7. Record Type Visibility**

Defining which record types are visible or available for an SObject within a permission set is primarily a metadata-level configuration.

* **Challenges with Direct SOQL**: Directly querying the link between a PermissionSet and its RecordTypeVisibility settings using standard data SObjects via SOQL is not straightforward. The SetupEntityAccess object does not typically include RecordType as a SetupEntityType.28  
* **Metadata API**: The PermissionSet metadata type includes a recordTypeVisibilities component, which is the standard way to define and retrieve this information programmatically.28  
* **Apex Describe**: While Apex DescribeSObjectResult.getRecordTypeInfos() can determine record type availability for a *user's profile context* 35, it does not directly provide a way to list the record type visibility definitions *for a given permission set* via SOQL.  
* **Approach for this Report**: Given the constraint of using WSC and SOQL, it must be acknowledged that retrieving RecordTypeVisibility assignments *within a specific permission set* is a limitation of this approach. While the RecordType SObject itself is queryable to get a list of all record types in the org, the specific assignments to permission sets are not easily queryable through standard SOQL against data SObjects. This detail is typically managed and retrieved via the Metadata API. The application can list all available record types in the org, but cannot directly state which ones are enabled by a specific permission set using only the WSC/SOQL data retrieval methods outlined.

### **4.8. Connected App Access**

Access to Connected Applications granted through a permission set is managed via the SetupEntityAccess object.

* **SetupEntityType Value**: 'ConnectedApplication'.12  
* **SetupEntityId**: This field will contain the Id of the ConnectedApplication SObject.  
* **SOQL Query Strategy**:  
  1. Query SetupEntityAccess for the specific PermissionSet.Id and SetupEntityType:  
     SQL  
     SELECT SetupEntityId  
     FROM SetupEntityAccess  
     WHERE ParentId \= :currentPermissionSetId AND SetupEntityType \= 'ConnectedApplication'  
     25  
  2. For each SetupEntityId returned, query the ConnectedApplication SObject to get its name and label:  
     SQL  
     SELECT Name, Label  
     FROM ConnectedApplication  
     WHERE Id \= :setupEntityIdFromStep1

This follows the standard two-step pattern for SetupEntityAccess: first, identify the linked entity ID, and second, query the entity's own SObject for descriptive details.

## **5\. Implementing Multithreaded Data Retrieval in Java**

To efficiently retrieve the extensive permission set details, particularly for organizations with numerous permission sets, a multithreaded approach is essential. Java's concurrency utilities provide the tools to manage this.

### **Core Java Concurrency Utilities**

* **java.util.concurrent.ExecutorService**: This is the preferred mechanism for managing a pool of threads and executing tasks asynchronously. It decouples task submission from thread creation and management.36  
* **java.util.concurrent.Executors**: This factory class provides static methods for creating various types of ExecutorService instances, such as newFixedThreadPool(int nThreads) which creates a pool with a fixed number of threads.36  
* **java.util.concurrent.Callable\<V\>**: An interface for tasks that are intended to return a result upon completion. This is suitable for tasks that fetch permission details for a specific PermissionSet.Id.  
* **java.util.concurrent.Future\<V\>**: Represents the result of an asynchronous computation. When a Callable is submitted to an ExecutorService, a Future is returned, which can be used to check task completion and retrieve its result.

### **Strategies for Task Decomposition**

The primary strategy for decomposing the data retrieval task is as follows:

1. The main application thread will execute the Phase 1 query to retrieve all PermissionSet Ids (and their core data).  
2. For each PermissionSet.Id obtained, a new Callable task will be created and submitted to the ExecutorService.  
3. Each Callable task will be responsible for performing all the Phase 2 queries (ObjectPermissions, FieldPermissions, SetupEntityAccess lookups, Tooling API calls for TabSettings, etc.) for its assigned PermissionSet.Id. The result of this Callable will be a comprehensive data object representing all details for that single permission set.

### **Thread Pool Sizing and API Limit Considerations**

The size of the thread pool is a critical parameter. Too few threads will underutilize client-side resources and fail to achieve significant parallelism. Conversely, too many threads can overwhelm the client machine or, more importantly, lead to exceeding Salesforce API concurrency limits.

**Table 3: Salesforce API Concurrency and Allocation Limits Overview**

| Limit Type | Default Limit (Production/Sandbox Orgs) | Scope | Key Considerations for this Application |
| :---- | :---- | :---- | :---- |
| Concurrent API Request Limits (Long-Running) | 25 (for requests \> 20 seconds) | Org-wide | If detail-fetching tasks become long-running, the thread pool size should not exceed this. Each thread might make multiple serial API calls per Permission Set. 38 |
| Total API Request Allocations | Varies by edition and licenses | Org-wide | The application can consume a large number of API calls. Efficient querying and batching are crucial to stay within daily limits. 38 |

It is recommended to start with a conservative, fixed-size thread pool (e.g., 5 to 10 threads) and make this value configurable. Performance testing and monitoring of Salesforce API usage (via Event Monitoring or API usage reports) are crucial to fine-tune this setting. If individual tasks primarily consist of short, selective SOQL queries, the 20-second long-running request limit might not be frequently encountered for each individual call. However, a high volume of concurrent fast queries can still place a significant load on the Salesforce platform and contribute to hitting overall org limits.

The design must be acutely aware of these limits. A naive approach where each thread independently fires off numerous SOQL queries for a single permission set could quickly lead to API throttling or errors. Strategies such as batching secondary lookups (discussed in Section 7\) become even more critical in a multithreaded context to ensure that the parallel execution does not lead to an unmanageable spike in API call volume.

### **Data Aggregation and Thread Safety**

Each Callable task will construct and return a data object (e.g., an instance of ComprehensivePermissionSet as defined in Section 6\) containing all retrieved permissions for one PermissionSet. The main thread will submit these tasks and receive a List\<Future\<ComprehensivePermissionSet\>\>. It will then iterate through this list, calling future.get() (potentially with a timeout) on each Future to retrieve the results. Since the results are collected and processed by the main thread after each Future completes, complex thread-safe collections for storing the final aggregated list are often not necessary if this pattern is followed. The primary list of results can be a standard ArrayList populated by the main thread.

### **Error Handling in Threads**

Each Callable task must implement robust error handling. WSC operations can throw various exceptions (e.g., com.sforce.ws.ConnectionException, com.sforce.soap.partner.fault.ApiFault, com.sforce.soap.partner.fault.InvalidSessionFault). These should be caught within the call() method.

* A failure in processing one PermissionSet.Id (e.g., due to an unexpected API error or data issue) should not necessarily halt the entire application. The Callable can log the error and return a partial result, a specific error indicator, or null.  
* If an exception is thrown from within the call() method, it will be wrapped in an java.util.concurrent.ExecutionException when Future.get() is called on the main thread. The main thread must catch ExecutionException (and InterruptedException) and inspect the cause to determine the nature of the failure for a specific task, allowing for granular error reporting and potentially continuing with other tasks.

The performance bottleneck in such a system can shift. While client-side multithreading addresses local processing limitations, the application's overall speed will still be heavily influenced by Salesforce API response times and adherence to governor limits. If queries are not optimized or if too many concurrent requests are made, threads may spend most of their time waiting for Salesforce, diminishing the benefits of parallelism. Thus, query optimization remains a critical factor even in a multithreaded design.

## **6\. Structuring Data for Display**

To effectively manage and ultimately display the comprehensive permission set information retrieved, a well-defined Java object model is necessary. This model should mirror the hierarchical nature of Salesforce permissions.

### **Proposed Java POJO Structure**

The following Plain Old Java Objects (POJOs) illustrate a potential structure:

Java

// Main container for all information about a single Permission Set  
public class ComprehensivePermissionSet {  
    // Core PermissionSet fields  
    String id;  
    String name;  
    String label;  
    String description;  
    boolean isCustom;  
    boolean isOwnedByProfile;  
    String type;  
    String namespacePrefix;  
    String licenseId; // Could be an object if more license details are needed  
    String profileName; // If IsOwnedByProfile is true

    // User Permissions (direct boolean fields from PermissionSet object)  
    List\<UserPermissionDetail\> userPermissions;

    // Detailed Permissions  
    List\<ObjectPermissionDetail\> objectPermissions;  
    List\<FieldPermissionDetail\> fieldPermissions;  
    List\<ApexClassAccessDetail\> apexClassAccesses;  
    List\<VisualForcePageAccessDetail\> vfPageAccesses;  
    List\<CustomPermissionAssignmentDetail\> customPermissionAssignments;  
    List\<ApplicationVisibilityDetail\> applicationVisibilities;  
    List\<TabSettingDetail\> tabSettings;  
    // List\<RecordTypeVisibilityDetail\> recordTypeVisibilities; // Acknowledging retrieval limitations via SOQL  
    List\<ConnectedAppAccessDetail\> connectedAppAccesses;  
    List\<ServicePresenceStatusAccessDetail\> servicePresenceStatusAccesses;

    // Constructors, getters, setters  
}

// Detail classes  
public class UserPermissionDetail {  
    String apiName; // e.g., "PermissionsViewAllData"  
    String uiLabel; // e.g., "View All Data"  
    boolean isEnabled;  
}

public class ObjectPermissionDetail {  
    String sobjectType; // API name of the object  
    boolean canRead;  
    boolean canCreate;  
    boolean canEdit;  
    boolean canDelete;  
    boolean canViewAllRecords;  
    boolean canModifyAllRecords;  
}

public class FieldPermissionDetail {  
    String sobjectType;  
    String fieldApiName; // e.g., "Account.Industry"  
    boolean canRead;  
    boolean canEdit;  
}

public class ApexClassAccessDetail {  
    String apexClassId;  
    String apexClassName;  
    String namespacePrefix;  
}

public class VisualForcePageAccessDetail {  
    String vfPageId;  
    String vfPageName;  
    String namespacePrefix;  
}

public class CustomPermissionAssignmentDetail {  
    String customPermissionId;  
    String customPermissionDeveloperName;  
    String customPermissionLabel;  
}

public class ApplicationVisibilityDetail {  
    String applicationId; // Could be AppMenuItem ID or CustomApplication ID  
    String applicationName;  
    String applicationLabel;  
    String applicationType; // e.g., "Custom", "Standard"  
}

public class TabSettingDetail {  
    String tabNameOrId; // API name of the CustomTabDefinition  
    String visibility; // e.g., "Visible", "Available", "None"  
}

public class ConnectedAppAccessDetail {  
    String connectedAppId;  
    String connectedAppName;  
    String connectedAppLabel;  
}

public class ServicePresenceStatusAccessDetail {  
    String statusId;  
    String statusDeveloperName;  
    String statusMasterLabel;  
}

The main application would then produce a List\<ComprehensivePermissionSet\>. The population of UserPermissionDetail would involve iterating through the known Permissions\* fields on the PermissionSet SObject and using a mapping 11 to get their UI labels.

The actual display of this complex, nested data is beyond the scope of this report, which focuses on data retrieval and structuring. However, effective presentation could involve a tree view in a UI, expandable sections, or exporting the data to formats like CSV or JSON for consumption by other reporting or analysis tools. The sheer volume and hierarchical depth of data for a single permission set, especially in complex orgs, means that any system consuming this output must be prepared to handle a potentially large and intricate object graph. Summarization or flattened views might be necessary depending on the specific display or analytical requirements.

## **7\. Key Considerations and Advanced Topics**

Developing a robust application for extracting comprehensive Salesforce permission set data requires attention to several critical areas beyond basic query execution and threading.

### **Error Handling and Retry Mechanisms**

* **WSC Exceptions**: The application must gracefully handle various exceptions thrown by the WSC library, such as com.sforce.ws.ConnectionException (for network issues or endpoint unavailability), com.sforce.soap.partner.fault.ApiFault (for general API errors), and particularly com.sforce.soap.partner.fault.InvalidSessionFault (if the provided access token is invalid or expired).  
* **Retry Logic**: For transient errors (e.g., temporary network glitches, intermittent API unavailability), implementing a retry mechanism with exponential backoff can improve application resilience. However, for InvalidSessionFault, retries are futile without a new token; the application should instead report the authentication failure. The concept of session renewal, though not directly applicable here due to the external token, highlights the importance of handling session-related issues.40

### **SOQL Query Optimization and Batching**

* **Selectivity**: Always use selective WHERE clauses, especially filtering on indexed fields like ParentId in child permission objects (ObjectPermissions, FieldPermissions, SetupEntityAccess). This significantly improves query performance.1  
* **Field Minimization**: Query only the fields that are strictly necessary for each step.  
* **Batching Secondary Lookups**: A crucial optimization, especially for SetupEntityAccess. Instead of querying the name of each ApexClass, ApexPage, etc., individually for every SetupEntityAccess record, a more efficient approach is:  
  1. After retrieving all SetupEntityAccess records for a batch of permission sets (or all of them).  
  2. Group these records by SetupEntityType.  
  3. For each type (e.g., 'ApexClass'), collect all unique SetupEntityId values.  
  4. Perform a single bulk SOQL query for that entity type (e.g., SELECT Id, Name, NamespacePrefix FROM ApexClass WHERE Id IN :setOfUniqueApexClassIds).  
  5. Map these results back to the original SetupEntityAccess records in memory. This strategy dramatically reduces the number of API calls from potentially N\*M (N permission sets \* M entities per set) to N \+ K (N permission sets \+ K entity types with one bulk query each), significantly improving performance and reducing API consumption.

### **Salesforce API Versioning**

* **WSC JAR Compatibility**: Ensure that the version of the WSC JAR files (e.g., force-wsc.jar, partner.jar, tooling.jar) is compatible with the target Salesforce API version being used in the service endpoints.  
* **Endpoint Versioning**: Service endpoint URLs explicitly include the API version (e.g., /services/Soap/u/58.0). This should be configurable.  
* **SObject and Field Changes**: Be aware that SObject structures and available fields can change between Salesforce API versions. The application should be developed and tested against a specific API version, and consideration given to maintenance if it needs to support multiple or future API versions.9

### **Security Best Practices for Token Handling**

* **Sensitivity**: The provided OAuth access token is a sensitive credential.  
* **Logging**: Avoid logging the full access token in application logs.  
* **Storage**: If the token needs to be stored temporarily (e.g., in memory), ensure it is handled securely. Avoid persistent storage on disk unless adequately encrypted.  
* **Transmission**: WSC uses HTTPS by default for API calls, ensuring secure transmission of the token in request headers.

### **Tooling API vs. Partner API Considerations**

* **Distinct Endpoints**: As established, certain data (e.g., PermissionSetTabSetting) requires the Tooling API, which uses a different service endpoint (/T/) than the Partner API (/u/).5  
* **Connection Management**: The application must manage separate ConnectorConfig and Connection objects (or reconfigure a single one, though separate objects are cleaner) to interact with both APIs using the same session ID.

### **Handling Large Data Volumes**

* **SOQL Query Limits**: Salesforce imposes limits on SOQL queries, such as a maximum of 50,000 records returned in a single transaction (though WSC handles queryMore() transparently for a single large result set).  
* **Heap Space**: Retrieving and holding data for a very large number of permission sets, each with extensive details, can consume significant Java heap space. Efficient data structures and processing are important.  
* **Application Scalability**: The application's design, particularly its batching strategies for secondary lookups and thread management, must be flexible enough to perform adequately in Salesforce orgs with vastly different numbers of permission sets and associated metadata components. Hardcoded batch sizes or thread pool configurations might be suboptimal. Making these parameters configurable is highly recommended to allow tuning for different environments.

## **8\. Conclusion and Recommendations**

This report has detailed a comprehensive approach for developing a Java application using the Salesforce Web Service Connector (WSC) to retrieve extensive Permission Set information from a Salesforce organization. The methodology leverages token-based authentication, a multi-phase SOQL-driven data retrieval process targeting various SObjects (PermissionSet, ObjectPermissions, FieldPermissions, SetupEntityAccess, and Tooling API's PermissionSetTabSetting), and employs Java's ExecutorService for multithreaded processing to enhance efficiency.

The primary challenges identified include navigating the complex Salesforce permission model, managing API call consumption within Salesforce governor limits, and handling the intricacies of different API endpoints (SOAP Partner API vs. Tooling API).

**Key Recommendations for Implementation:**

1. **Robust Error Handling and Logging**: Implement comprehensive try-catch blocks for all WSC calls and within threaded tasks. Log errors effectively, distinguishing between transient issues (potentially retryable) and critical failures (e.g., invalid session ID).  
2. **API Call Optimization**:  
   * Prioritize the batching strategy for secondary lookups (e.g., fetching names for SetupEntityIds). Collect unique IDs across multiple permission sets and perform bulk queries for each entity type. This is the single most impactful optimization for reducing API calls.  
   * Ensure all SOQL queries are selective, using WHERE clauses on indexed fields wherever possible, and retrieve only necessary fields.  
3. **Configurable Concurrency and Batching**: Make the thread pool size and any internal batch sizes (e.g., for batched secondary lookups) configurable parameters. This allows the application to be tuned for different Salesforce org sizes and API limit profiles.  
4. **API Limit Adherence**: Closely monitor Salesforce API usage (concurrent requests and total daily allocations) during development and testing. Adjust concurrency and batching parameters to operate reliably within these limits.  
5. **Manage API Contexts**: Implement clean management of ConnectorConfig and Connection objects for both Partner and Tooling APIs, ensuring the correct service endpoint is used for each type of query.  
6. **Secure Token Handling**: Treat the input access token as a highly sensitive credential. Avoid logging it, and if temporary in-memory storage is required, ensure it is handled securely.  
7. **Acknowledge Limitations**: Clearly document any limitations of the pure SOQL/WSC approach, particularly concerning the retrieval of RecordTypeVisibility assignments within permission sets, which are more directly accessible via the Metadata API.  
8. **Modular Design**: Consider designing the core data retrieval logic as a reusable library or service. Given the complexity encapsulated, such a component could be valuable for various internal security analysis or compliance tools that require deep insight into Salesforce permissions.

By adhering to these guidelines and recommendations, the developed Java application can serve as a powerful and efficient tool for extracting and analyzing comprehensive Salesforce Permission Set data, providing valuable insights into an organization's security posture.

#### **Sources des citations**

1. An Admin's Guide to SOQL \+ Examples \- Salesforce Ben, consulté le mai 14, 2025, [https://www.salesforceben.com/admins-guide-to-salesforce-soql/](https://www.salesforceben.com/admins-guide-to-salesforce-soql/)  
2. SOQL and SOSL Reference \- Salesforce, consulté le mai 14, 2025, [https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/salesforce\_soql\_sosl.pdf](https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/salesforce_soql_sosl.pdf)  
3. Automate Session ID (access token generation). Integrate Salesforce with Postman Part 2\!\!, consulté le mai 14, 2025, [https://www.youtube.com/watch?v=IZX9Nlo6pYk](https://www.youtube.com/watch?v=IZX9Nlo6pYk)  
4. Rest API Session ID SOAP Partner Connection \- Salesforce Stack ..., consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/9376/rest-api-session-id-soap-partner-connection](https://salesforce.stackexchange.com/questions/9376/rest-api-session-id-soap-partner-connection)  
5. Connecting to Tooling API using Java SOAP \- Salesforce Stack ..., consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/50622/connecting-to-tooling-api-using-java-soap](https://salesforce.stackexchange.com/questions/50622/connecting-to-tooling-api-using-java-soap)  
6. Salesforce: Query Fields Permission \- SimplySfdc.com, consulté le mai 14, 2025, [https://www.simplysfdc.com/2019/03/salesforce-query-field-permission.html](https://www.simplysfdc.com/2019/03/salesforce-query-field-permission.html)  
7. Permissions starting with 'X' character in soql query results \- Salesforce Help, consulté le mai 14, 2025, [https://help.salesforce.com/s/articleView?id=000387815\&language=en\_US\&type=1](https://help.salesforce.com/s/articleView?id=000387815&language=en_US&type=1)  
8. soql on object permissions on a profile \- Salesforce Stack Exchange, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/237238/soql-on-object-permissions-on-a-profile](https://salesforce.stackexchange.com/questions/237238/soql-on-object-permissions-on-a-profile)  
9. PermissionSet | Salesforce Field Reference Guide, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.sfFieldRef.meta/sfFieldRef/salesforce\_field\_reference\_PermissionSet.htm](https://developer.salesforce.com/docs/atlas.en-us.sfFieldRef.meta/sfFieldRef/salesforce_field_reference_PermissionSet.htm)  
10. PermissionSet | Object Reference for the Salesforce Platform ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_permissionset.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionset.htm)  
11. Challenge: Mapping between Salesforce Permission Name and the ..., consulté le mai 14, 2025, [https://www.sfdcamplified.com/challenge-map-between-salesforce-permissionname-and-label/](https://www.sfdcamplified.com/challenge-map-between-salesforce-permissionname-and-label/)  
12. Query Author Apex Permission? \- soql \- Salesforce StackExchange, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/145427/query-author-apex-permission](https://salesforce.stackexchange.com/questions/145427/query-author-apex-permission)  
13. Mapping between Salesforce Permission Name and the Label, consulté le mai 14, 2025, [http://www.fishofprey.com/2020/10/mappings-between-salesforce-permission.html](http://www.fishofprey.com/2020/10/mappings-between-salesforce-permission.html)  
14. Mass Update Access to Objects And Fields For Profiles And Permission Sets \- SFXD Wiki, consulté le mai 14, 2025, [https://wiki.sfxd.org/books/best-practices/chapter/mass-update-access-to-objects-and-fields-for-profiles-and-permission-sets/export/html](https://wiki.sfxd.org/books/best-practices/chapter/mass-update-access-to-objects-and-fields-for-profiles-and-permission-sets/export/html)  
15. Object Permissions \- B... | SFXD Wiki, consulté le mai 14, 2025, [https://wiki.sfxd.org/books/best-practices/page/object-permissions---basic-functionality](https://wiki.sfxd.org/books/best-practices/page/object-permissions---basic-functionality)  
16. SOQL Query on FieldPermissions \- Salesforce Stack Exchange, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/417073/soql-query-on-fieldpermissions](https://salesforce.stackexchange.com/questions/417073/soql-query-on-fieldpermissions)  
17. Salesforce Standard Objects and Fields · GitHub, consulté le mai 14, 2025, [https://gist.github.com/surajp/75e4b283479e066e2044b7208f8f980d](https://gist.github.com/surajp/75e4b283479e066e2044b7208f8f980d)  
18. SOQL – Salesforce Object Query Language \- salesforce888 \- WordPress.com, consulté le mai 14, 2025, [https://salesforce888.wordpress.com/2015/01/02/soql-salesforce-object-query-language/](https://salesforce888.wordpress.com/2015/01/02/soql-salesforce-object-query-language/)  
19. SetupEntityAccess | Object Reference for the Salesforce Platform, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_setupentityaccess.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_setupentityaccess.htm)  
20. Is it possible to query the Apex Class Accesses of a permission set ..., consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/270297/is-it-possible-to-query-the-apex-class-accesses-of-a-permission-set](https://salesforce.stackexchange.com/questions/270297/is-it-possible-to-query-the-apex-class-accesses-of-a-permission-set)  
21. User Level Visualforce UI Control w/ Custom Permissions \- CRM Science, consulté le mai 14, 2025, [https://www.crmscience.com/single-post/2015/06/07/user-level-visualforce-ui-control-w-custom-permissions](https://www.crmscience.com/single-post/2015/06/07/user-level-visualforce-ui-control-w-custom-permissions)  
22. Query to List Permission Sets with Access to Visualforce Page \- ServiceMax Knowledge, consulté le mai 14, 2025, [https://community.servicemax.com/s/article/Query-to-List-Permission-Sets-with-Access-to-Visualforce-Page](https://community.servicemax.com/s/article/Query-to-List-Permission-Sets-with-Access-to-Visualforce-Page)  
23. Creating, Assigning and Checking Custom Permissions | Andy in the ..., consulté le mai 14, 2025, [https://andyinthecloud.com/2015/01/14/creating-assigning-and-checking-custom-permissions/](https://andyinthecloud.com/2015/01/14/creating-assigning-and-checking-custom-permissions/)  
24. Using SOQL Queries to Discover Users with Custom Permissions | Blog \- Forceshark, consulté le mai 14, 2025, [https://forceshark.com/blog/using-soql-queries-to-discover-users-with-custom-permissions](https://forceshark.com/blog/using-soql-queries-to-discover-users-with-custom-permissions)  
25. Connected Apps \- sfdc techie – Pavan's blog, consulté le mai 14, 2025, [https://sfdctechie.wordpress.com/2018/08/12/connected-apps/](https://sfdctechie.wordpress.com/2018/08/12/connected-apps/)  
26. How to retrieve the available omni | Salesforce Trailblazer Community, consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54S00000A8xf5SAB](https://trailhead.salesforce.com/trailblazer-community/feed/0D54S00000A8xf5SAB)  
27. Service Presence Statuses Access on Profiles and Permission Sets, consulté le mai 14, 2025, [https://docs.copado.com/articles/copado-ci-cd-publication/service-presence-statuses-access-on-profiles-and-permission-sets](https://docs.copado.com/articles/copado-ci-cd-publication/service-presence-statuses-access-on-profiles-and-permission-sets)  
28. How do you / can you assign a Record Type to a Permission Set in Apex code?, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/83576/how-do-you-can-you-assign-a-record-type-to-a-permission-set-in-apex-code](https://salesforce.stackexchange.com/questions/83576/how-do-you-can-you-assign-a-record-type-to-a-permission-set-in-apex-code)  
29. SetupEntityAccess | Salesforce Field Reference Guide | Salesforce ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.sfFieldRef.meta/sfFieldRef/salesforce\_field\_reference\_SetupEntityAccess.htm](https://developer.salesforce.com/docs/atlas.en-us.sfFieldRef.meta/sfFieldRef/salesforce_field_reference_SetupEntityAccess.htm)  
30. New and Changed Objects \- Salesforce Help, consulté le mai 14, 2025, [https://help.salesforce.com/s/articleView?id=release-notes.rn\_api\_objects.htm\&language=en\_US\&release=244\&type=5](https://help.salesforce.com/s/articleView?id=release-notes.rn_api_objects.htm&language=en_US&release=244&type=5)  
31. PermissionSet | Metadata API Developer Guide, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.api\_meta.meta/api\_meta/meta\_permissionset.htm](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_permissionset.htm)  
32. Can I query apps assigned to a | Salesforce Trailblazer Community \- Trailhead, consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007X96b0SAB](https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007X96b0SAB)  
33. PermissionSetTabSetting | Tooling API | Salesforce Developers, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.api\_tooling.meta/api\_tooling/tooling\_api\_objects\_permissionsettabsetting.htm](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_permissionsettabsetting.htm)  
34. How to query Users who have access | Salesforce Trailblazer ..., consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T41hfSAB](https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T41hfSAB)  
35. Query recordtype access in profiles | Salesforce Trailblazer Community \- Trailhead, consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T4NzWSAV](https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T4NzWSAV)  
36. Java Multi-Threading With the ExecutorService \- DZone, consulté le mai 14, 2025, [https://dzone.com/articles/java-concurrency-multi-threading-with-executorserv](https://dzone.com/articles/java-concurrency-multi-threading-with-executorserv)  
37. Java Thread Pool Implementation and Best Practices in Business Applications, consulté le mai 14, 2025, [https://www.alibabacloud.com/blog/java-thread-pool-implementation-and-best-practices-in-business-applications\_601528](https://www.alibabacloud.com/blog/java-thread-pool-implementation-and-best-practices-in-business-applications_601528)  
38. Implementation Considerations | SOAP API Developer Guide ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/implementation\_considerations.htm](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/implementation_considerations.htm)  
39. How many simultaneous calls Api? | Salesforce Trailblazer Community \- Trailhead, consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T4SCcSAN](https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T4SCcSAN)  
40. Salesforce WSC added support for Session Timeout handling\! \- Concret.io, consulté le mai 14, 2025, [https://www.concret.io/blog/salesforce-wsc-added-support-for-session-timeout-handling](https://www.concret.io/blog/salesforce-wsc-added-support-for-session-timeout-handling)  
41. Update custom Profile and Permission Set object level permissions with Data Loader, consulté le mai 14, 2025, [https://help.salesforce.com/s/articleView?id=000381102\&language=en\_US\&type=1](https://help.salesforce.com/s/articleView?id=000381102&language=en_US&type=1)  
42. Provide Object permission using | Salesforce Trailblazer Community \- Trailhead, consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T4WWwSAN](https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T4WWwSAN)  
43. ObjectPermissions | Object Reference for the Salesforce Platform ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_objectpermissions.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_objectpermissions.htm)  
44. How to extract object permissions for all profiles? : r/salesforce \- Reddit, consulté le mai 14, 2025, [https://www.reddit.com/r/salesforce/comments/ssdu69/how\_to\_extract\_object\_permissions\_for\_all\_profiles/](https://www.reddit.com/r/salesforce/comments/ssdu69/how_to_extract_object_permissions_for_all_profiles/)  
45. How to Review Field-Level Security in Salesforce Using Permission ..., consulté le mai 14, 2025, [https://www.salesforceben.com/how-to-review-field-level-security-in-salesforce-using-permission-sets/](https://www.salesforceben.com/how-to-review-field-level-security-in-salesforce-using-permission-sets/)  
46. FieldPermissions | Object Reference for the Salesforce Platform ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_fieldpermissions.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_fieldpermissions.htm)
