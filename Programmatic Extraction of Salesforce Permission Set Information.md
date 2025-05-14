# **Programmatic Extraction of Salesforce Permission Set Information**

## **I. Introduction to Programmatic Permission Set Reporting**

Salesforce Permission Sets are a fundamental component of the Salesforce security model, designed to grant users additional permissions and access beyond their baseline profile settings.1 Salesforce strongly recommends the use of permission sets and permission set groups over profiles for managing granular user permissions, particularly for object and field-level access.2 This shift emphasizes a more modular and reusable approach to security configuration.

In environments with numerous users, complex business processes, and stringent compliance requirements, the ability to programmatically extract comprehensive permission set data is critical. Manual review of permissions across an organization is often impractical, time-consuming, and prone to errors. Programmatic extraction facilitates thorough security audits, enables automated compliance reporting, simplifies user access reviews, and allows for efficient management of permissions, especially in large or intricate Salesforce organizations. The distributed nature of permission data across various Salesforce objects necessitates a programmatic, multi-query approach to gather a complete picture. This means that a reporting program will need to interact with several distinct SObjects and potentially different APIs to consolidate all the required information.

The increasing complexity of Salesforce orgs, coupled with the drive towards more granular permission management through permission sets and permission set groups, elevates automated reporting tools from a mere convenience to an essential component for maintaining robust security and ensuring adherence to compliance mandates. This report aims to equip developers with the necessary knowledge of Salesforce SObjects, their relevant fields, inter-object relationships, and API strategies required to construct programs capable of generating detailed and comprehensive reports on permission set configurations.

## **II. Core Salesforce Objects for Permission Set Data**

To build a comprehensive report, understanding the primary Salesforce objects that store permission set information is crucial. These objects serve as the foundation for querying attributes and assignments.

### **A. PermissionSet Object**

The PermissionSet SObject is the central entity representing a permission set within Salesforce. It stores metadata about the permission set itself, including many system-level and user-level permissions directly as fields on the object.

Key identifying fields on the PermissionSet object include:

* Id: The unique 15-character or 18-character Salesforce ID for the permission set.  
* Name: The API name of the permission set, used in metadata deployments and API calls.  
* Label: The human-readable display name of the permission set as seen in the Salesforce UI.  
* Description: An optional field to describe the purpose or contents of the permission set.

Attribute fields provide further context about the permission set's nature and linkage:

* Type: Indicates the category of the permission set, such as 'regular', 'sessionBased' (for session-based permission sets), or 'group' (related to permission set groups).  
* IsOwnedByProfile: A critical boolean field that distinguishes true, assignable permission sets from the permission set records intrinsically linked to each Profile.4 When true, the PermissionSet record represents the permissions defined within a specific Profile. For reporting on standalone permission sets, filtering WHERE IsOwnedByProfile \= false is essential to avoid redundant data that merely reflects profile settings.  
* ProfileId: If IsOwnedByProfile is true, this field contains the ID of the associated Profile object.  
* LicenseId: If the permission set is associated with a specific permission set license (which may unlock certain features), this field links to the PermissionSetLicense object.6 This is important for understanding if a permission set's capabilities are tied to a licensable feature.  
* NamespacePrefix: For permission sets that are part of a managed or unmanaged package, this field indicates the namespace of that package.

A significant number of general User Permissions are stored as direct boolean fields on the PermissionSet object. These fields typically follow the naming convention Permissions\<PermissionName\>, for example, PermissionsApiEnabled for the "API Enabled" user permission or PermissionsAuthorApex for the "Author Apex" permission.7 This direct storage simplifies the extraction of these common permissions, as they can be queried without joining to related child objects. For instance, the "API Enabled" permission is a fundamental user permission required for API access.8 The ability to query these directly from the PermissionSet object is an efficiency gain for reporting programs.

*Table II.A.1: PermissionSet Object \- Key Fields and Common System/User Permission Fields*

| Field API Name | Data Type | Description/Purpose | Example Values/Notes |
| :---- | :---- | :---- | :---- |
| Id | ID | Unique identifier for the PermissionSet. | 0PSxxxxxxxxxxxxxxx |
| Name | String | API name of the PermissionSet. | My\_Custom\_Permission\_Set |
| Label | String | Display label of the PermissionSet. | My Custom Permission Set |
| Description | Text Area | Description of the PermissionSet's purpose. | Grants access to X, Y, and Z features. |
| Type | Picklist | Type of PermissionSet. | regular, sessionBased, group |
| IsCustom | Boolean | Indicates if the permission set is custom (true) or standard (false). | true, false |
| IsOwnedByProfile | Boolean | If true, this PermissionSet is associated with a Profile and represents its permissions.4 | true, false |
| ProfileId | Reference | ID of the Profile if IsOwnedByProfile is true. | References Profile.Id |
| LicenseId | Reference | ID of the PermissionSetLicense associated with this PermissionSet, if any.6 | References PermissionSetLicense.Id |
| NamespacePrefix | String | Namespace prefix if the PermissionSet is part of a package. | my\_ns |
| PermissionsApiEnabled | Boolean | User Permission: Allows API access.7 | true, false |
| PermissionsViewAllData | Boolean | User Permission: Allows viewing all data.7 | true, false |
| PermissionsModifyAllData | Boolean | User Permission: Allows modifying all data.7 | true, false |
| PermissionsAuthorApex | Boolean | User Permission: Allows authoring Apex classes and triggers.7 | true, false |
| PermissionsActivateContract | Boolean | User Permission: Allows activation of contracts.7 | true, false |
| PermissionsAssignPermissionSets | Boolean | User Permission: Allows assignment of permission sets to users.7 | true, false |
| PermissionsBulkApiHardDelete | Boolean | User Permission: Allows hard deletion of records via Bulk API.7 | true, false |
| PermissionsConvertLeads | Boolean | User Permission: Allows conversion of leads.7 | true, false |
| PermissionsCustomizeApplication | Boolean | User Permission: Allows customization of the application (e.g., creating custom fields, page layouts).7 | true, false |
| PermissionsDataExport | Boolean | User Permission: Allows weekly data export.7 | true, false |
| PermissionsEditPublicReports | Boolean | User Permission: Allows editing of public reports.7 | true, false |
| PermissionsManageUsers | Boolean | User Permission: Allows management of users (e.g., creating users, resetting passwords).7 | true, false |

### **B. PermissionSetAssignment Object**

The PermissionSetAssignment object serves as the crucial link, or junction object, that connects a User to a PermissionSet or a PermissionSetGroup. Each record in this object signifies that a specific permission set (or group) has been assigned to a particular user, thereby granting that user the permissions contained within the set or group.9

Key fields for this object include:

* Id: The unique identifier for the permission set assignment record itself.  
* AssigneeId: The ID of the User to whom the permission set or permission set group is assigned. This field links directly to the User object.  
* PermissionSetId: The ID of the assigned PermissionSet. This field links to the PermissionSet object.  
* PermissionSetGroupId: If the assignment is for a PermissionSetGroup rather than an individual PermissionSet, this field will contain the ID of the PermissionSetGroup.  
* ExpirationDate: A significant field that allows for time-bound permission assignments.11 When an expiration date is set, the user loses the permissions granted by this assignment after this date. This is critical for auditing temporary access grants and ensuring adherence to policies requiring time-limited privileges. Reports on permission sets must include this field to accurately reflect the temporal nature of some access rights.  
* IsActive: A boolean field that may indicate if the assignment is currently active. (Official documentation should be consulted for the precise behavior and availability of this field across API versions).

The PermissionSetAssignment object is the definitive source for determining which users are granted which permission sets. Any comprehensive permission report must query this object to identify the "assignees" aspect of the user's request.

*Table II.B.1: PermissionSetAssignment Object \- Key Fields*

| Field API Name | Data Type | Description/Purpose |
| :---- | :---- | :---- |
| Id | ID | Unique identifier for the PermissionSetAssignment record. |
| AssigneeId | Reference | ID of the User to whom the permission set/group is assigned. References User.Id. |
| PermissionSetId | Reference | ID of the assigned PermissionSet. References PermissionSet.Id. |
| PermissionSetGroupId | Reference | ID of the assigned PermissionSetGroup, if applicable. References PermissionSetGroup.Id. |
| ExpirationDate | DateTime | Date and time when the assignment expires, if applicable.11 |
| IsActive | Boolean | Indicates if the assignment is currently active (confirm availability/behavior). |

## **III. Extracting Specific Permission Types (Artifacts and Attributes)**

Beyond the core attributes of a permission set and its assignments, a comprehensive report must detail the specific permissions granted. These are often stored in related objects or through specific structures within the PermissionSet metadata.

### **A. Object Permissions (CRUD)**

Object-level permissions, commonly referred to as CRUD (Create, Read, Edit, Delete) permissions, along with "View All" and "Modify All" data access, are defined in the ObjectPermissions SObject. This object establishes a link between a PermissionSet (identified by ParentId) and a specific SObject type (identified by SobjectType), detailing the exact CRUD operations allowed.2

The key fields on the ObjectPermissions object are:

* Id: The unique identifier for the ObjectPermissions record.  
* ParentId: This field links to the Id of the parent PermissionSet record.  
* SobjectType: The API name of the SObject to which these permissions apply (e.g., 'Account', 'MyCustomObject\_\_c').  
* PermissionsCreate: A boolean field indicating if the permission set allows creation of new records for the SobjectType.  
* PermissionsRead: A boolean field indicating if the permission set allows reading of records for the SobjectType.  
* PermissionsEdit: A boolean field indicating if the permission set allows editing of existing records for the SobjectType.  
* PermissionsDelete: A boolean field indicating if the permission set allows deletion of records for the SobjectType.  
* PermissionsViewAllRecords: A boolean field indicating if the permission set grants "View All" access for the SobjectType, overriding sharing rules to allow visibility of all records.2  
* PermissionsModifyAllRecords: A boolean field indicating if the permission set grants "Modify All" access for the SobjectType, overriding sharing rules to allow modification of all records.2

An important characteristic of ObjectPermissions is that a record will only exist for a given PermissionSet and SobjectType combination if at least one of these six boolean permissions is granted. The absence of an ObjectPermissions record for a particular object within a permission set signifies that the permission set grants no direct access to that object.13 Reporting tools should account for this; for instance, a LEFT JOIN from PermissionSet to ObjectPermissions or post-query processing might be necessary to explicitly state "No Access" rather than just omitting the object.

Furthermore, when querying ObjectPermissions, it's crucial to consider the IsOwnedByProfile field on the related PermissionSet (via ParentId). If PermissionSet.IsOwnedByProfile is true, the object permissions are derived from a Profile definition. To report specifically on object permissions granted by assignable permission sets, queries should filter WHERE PermissionSet.IsOwnedByProfile \= false.4

*Table III.A.1: ObjectPermissions Object \- Key Fields*

| Field API Name | Data Type | Description/Purpose |
| :---- | :---- | :---- |
| Id | ID | Unique identifier for the ObjectPermissions record. |
| ParentId | Reference | ID of the parent PermissionSet. References PermissionSet.Id. |
| SobjectType | Picklist | API name of the SObject (e.g., 'Account', 'CustomObject\_\_c'). |
| PermissionsCreate | Boolean | Grants permission to create records of the SobjectType. |
| PermissionsRead | Boolean | Grants permission to read records of the SobjectType. |
| PermissionsEdit | Boolean | Grants permission to edit records of the SobjectType. |
| PermissionsDelete | Boolean | Grants permission to delete records of the SobjectType. |
| PermissionsViewAllRecords | Boolean | Grants permission to view all records of the SobjectType, overriding sharing.2 |
| PermissionsModifyAllRecords | Boolean | Grants permission to read, edit, delete, transfer, and approve all records of the SobjectType, overriding sharing.2 |

### **B. Field-Level Security (FLS)**

Field-Level Security (FLS) dictates read and edit access to individual fields on an SObject and is defined through the FieldPermissions SObject. This object connects a PermissionSet (via ParentId) to a specific field (Field) on a particular SObject (SobjectType).1

Key fields on the FieldPermissions object include:

* Id: The unique identifier for the FieldPermissions record.  
* ParentId: Links to the Id of the parent PermissionSet record.  
* SobjectType: The API name of the SObject containing the field (e.g., 'Account').  
* Field: The fully qualified API name of the field (e.g., 'Account.Name', 'Opportunity.Amount', 'MyCustomObject\_\_c.CustomField\_\_c'). It's important to note that this field stores the object name prefixed to the field name.1  
* PermissionsRead: A boolean field; if true, the permission set grants read access to the specified field.  
* PermissionsEdit: A boolean field; if true, the permission set grants edit access to the specified field.

Similar to ObjectPermissions, a FieldPermissions record for a specific field within a permission set generally implies that some FLS (either read or edit) is explicitly granted or denied by that permission set for that field. Object-level access is a prerequisite for field-level access; if a permission set doesn't grant at least read access to an object, FLS settings for fields on that object within that permission set are effectively moot. The reporting program should consider this hierarchy. Salesforce recommends using permission sets to manage users' field permissions.3

When querying FieldPermissions, the Field value includes the object's API name, such as Account.SLA\_\_c.1 This format is crucial for constructing accurate SOQL WHERE clauses or for parsing the retrieved data.

*Table III.B.1: FieldPermissions Object \- Key Fields*

| Field API Name | Data Type | Description/Purpose |
| :---- | :---- | :---- |
| Id | ID | Unique identifier for the FieldPermissions record. |
| ParentId | Reference | ID of the parent PermissionSet. References PermissionSet.Id. |
| SobjectType | Picklist | API name of the SObject containing the field (e.g., 'Account'). |
| Field | Picklist | API name of the field, prefixed with SobjectType (e.g., 'Account.Industry'). |
| PermissionsRead | Boolean | If true, grants read access to the field. |
| PermissionsEdit | Boolean | If true, grants edit access to the field. |

### **C. Setup Entity Access (SetupEntityAccess Object)**

The SetupEntityAccess object is a versatile SObject that controls access to a variety of "setup entities" within Salesforce. These entities include Apex Classes, Visualforce Pages, Custom Permissions, Connected Applications, and Service Presence Statuses, among others. It acts as a polymorphic gatekeeper for many permissions that are not covered by ObjectPermissions or FieldPermissions.

Key fields for the SetupEntityAccess object are:

* Id: The unique identifier for the SetupEntityAccess record.  
* ParentId: Links to the Id of the parent PermissionSet record that is granting the access.  
* SetupEntityId: The ID of the specific setup entity to which access is being granted (e.g., the Id of an ApexClass, ApexPage, or CustomPermission record).  
* SetupEntityType: A string picklist field that indicates the type of the setup entity. Common values include 'ApexClass', 'ApexPage', 'CustomPermission', 'ConnectedApplication', and 'ServicePresenceStatus'.17 Understanding this field is crucial for targeting queries to the correct type of permission.

Access to SetupEntityAccess records is granted via a PermissionSet. Therefore, to report on these permissions, one must first identify the relevant PermissionSet (e.g., by its name or ID) and then query SetupEntityAccess using the PermissionSet.Id as the ParentId.

A common pattern when working with SetupEntityAccess is a two-step query process to obtain meaningful names for the entities. First, query SetupEntityAccess to get the SetupEntityId for a specific SetupEntityType and ParentId. Second, use these SetupEntityId values to query the corresponding object (e.g., ApexClass, CustomPermission) to retrieve human-readable names or developer names.17

*Table III.C.1: SetupEntityAccess Object \- Key Fields*

| Field API Name | Data Type | Description/Purpose |
| :---- | :---- | :---- |
| Id | ID | Unique identifier for the SetupEntityAccess record. |
| ParentId | Reference | ID of the parent PermissionSet. References PermissionSet.Id. |
| SetupEntityId | Reference | ID of the specific setup entity (e.g., ApexClassId, CustomPermissionId). |
| SetupEntityType | Picklist | Type of the setup entity (e.g., 'ApexClass', 'CustomPermission'). |

*Table III.C.2: Common SetupEntityType Values and Corresponding Entities*

| SetupEntityType Value | Corresponding Salesforce Entity | Entity ID to Name Mapping |
| :---- | :---- | :---- |
| ApexClass | ApexClass | SetupEntityId \= ApexClass.Id; Name \= ApexClass.Name 17 |
| ApexPage | ApexPage | SetupEntityId \= ApexPage.Id; Name \= ApexPage.Name 18 |
| CustomPermission | CustomPermission | SetupEntityId \= CustomPermission.Id; Name \= CustomPermission.DeveloperName 21 |
| ConnectedApplication | ConnectedApplication | SetupEntityId \= ConnectedApplication.Id; Name \= ConnectedApplication.Name 19 |
| ServicePresenceStatus | ServicePresenceStatus | SetupEntityId \= ServicePresenceStatus.Id; Name \= ServicePresenceStatus.MasterLabel or DeveloperName 20 |
| TabSet | AppMenuItem | SetupEntityId \= AppMenuItem.Id (where AppMenuItem.Type \= 'TabSet'); Name \= AppMenuItem.Name 18 |
| CustomApplication | AppMenuItem | SetupEntityId \= AppMenuItem.Id (where AppMenuItem.Type \= 'CustomApplication'); Name \= AppMenuItem.ApplicationId (Developer Name of App) |
| CustomMetadata | Custom Metadata Type Record | SetupEntityId \= ID of the custom metadata type record; Name resolution requires knowing the specific custom metadata type. 24 |
| CustomSetting | Custom Setting Record | SetupEntityId \= ID of the custom setting record; Name resolution requires knowing the specific custom setting. 24 |
| ExternalDataSource | ExternalDataSource | SetupEntityId \= ExternalDataSource.Id; Name \= ExternalDataSource.DeveloperName 18 |
| NamedCredential | NamedCredential | SetupEntityId \= NamedCredential.Id; Name \= NamedCredential.DeveloperName 23 |

#### **1\. Apex Class Access**

Access to Apex classes is controlled via SetupEntityAccess records where the SetupEntityType is 'ApexClass'. The SetupEntityId field in these records corresponds to the ApexClass.Id.  
To generate a report of Apex classes accessible through a permission set:

1. Query SetupEntityAccess for records where ParentId matches the PermissionSet.Id and SetupEntityType \= 'ApexClass'.  
2. Collect all unique SetupEntityId values from the results.  
3. Query the ApexClass object WHERE Id IN (collected\_SetupEntityIds) to retrieve the names (ApexClass.Name) and other details of the accessible classes.17

#### **2\. Visualforce Page Access**

Similarly, Visualforce page access is managed through SetupEntityAccess with SetupEntityType \= 'ApexPage'. The SetupEntityId links to ApexPage.Id.  
The process for reporting is analogous to Apex class access:

1. Query SetupEntityAccess for records with the relevant ParentId (PermissionSet.Id) and SetupEntityType \= 'ApexPage'.  
2. Collect the SetupEntityId values.  
3. Query the ApexPage object WHERE Id IN (collected\_SetupEntityIds) to get page names (ApexPage.Name).18

#### **3\. Custom Permission Access**

Custom permissions assigned through a permission set are also recorded in SetupEntityAccess. For these, SetupEntityType \= 'CustomPermission', and SetupEntityId corresponds to CustomPermission.Id.  
To report on assigned custom permissions:

1. Query SetupEntityAccess for records with the ParentId (PermissionSet.Id) and SetupEntityType \= 'CustomPermission'.  
2. Collect the SetupEntityId values.  
3. Query the CustomPermission object WHERE Id IN (collected\_SetupEntityIds) to obtain the DeveloperName of each custom permission.21

#### **4\. Connected Application Access**

Access to Connected Applications can be granted via permission sets and is tracked using SetupEntityAccess where SetupEntityType \= 'ConnectedApplication'. The SetupEntityId in this context refers to the ConnectedApplication.Id.  
To list connected apps accessible through a permission set:

1. Query SetupEntityAccess with ParentId (PermissionSet.Id) and SetupEntityType \= 'ConnectedApplication'.  
2. Collect the SetupEntityId values.  
3. Query the ConnectedApplication object WHERE Id IN (collected\_SetupEntityIds) to get the names (ConnectedApplication.Name).19

#### **5\. Service Presence Status Access**

For organizations using Omni-Channel, access to specific Service Presence Statuses can be controlled by permission sets. This is managed via SetupEntityAccess with SetupEntityType \= 'ServicePresenceStatus'. The SetupEntityId points to the ServicePresenceStatus.Id.  
To report on these:

1. Query SetupEntityAccess with ParentId (PermissionSet.Id) and SetupEntityType \= 'ServicePresenceStatus'.  
2. Collect the SetupEntityId values.  
3. Query the ServicePresenceStatus object WHERE Id IN (collected\_SetupEntityIds) to get status details like MasterLabel or DeveloperName.20

#### **6\. Assigned Application (App Menu) Visibility**

The visibility of applications in the app launcher, when granted through a permission set, can be somewhat nuanced. While the Metadata API's applicationVisibilities section within the PermissionSet type offers a direct definition 26, a SOQL-based approach can often involve SetupEntityAccess.  
One common way this is represented is through SetupEntityType \= 'TabSet' or SetupEntityType \= 'CustomApplication'. In these cases, SetupEntityId usually refers to an AppMenuItem.Id. The AppMenuItem object represents items in the app menu, including custom and standard applications.  
To query this via SOQL:

1. Query SetupEntityAccess for the relevant ParentId (PermissionSet.Id) and SetupEntityType values like 'TabSet' or 'CustomApplication'.  
2. Collect the SetupEntityId values.  
3. Query the AppMenuItem object WHERE Id IN (collected\_SetupEntityIds) and filter by Type \= 'TabSet' or Type \= 'CustomApplication' to get application names (often from AppMenuItem.ApplicationId which is the developer name of the app, or AppMenuItem.Name). It is important to note that querying assigned app visibility purely through SOQL on standard SObjects might sometimes be indirect or incomplete compared to parsing the Metadata API. The SetupEntityAccess object with SetupEntityType \= 'TabSet' or CustomApplication (linking to AppMenuItem) is the most common SOQL path.18 However, if comprehensive details mirroring the setup UI are needed, the Metadata API's applicationVisibilities structure is more direct. This highlights a scenario where SOQL alone might not provide the full depth of information available through other API mechanisms.

### **D. Custom Tab Visibility**

Visibility settings for custom tabs (and standard object tabs) within a permission set are managed via the PermissionSetTabSetting object. This object is accessible primarily through the **Tooling API**, not standard SObject SOQL.27

Key fields for PermissionSetTabSetting (Tooling API) include:

* Id: The unique identifier for the tab setting record.  
* ParentId: Links to the Id of the PermissionSet record.  
* Name: The DeveloperName of the Tab Definition. This could be the API name of a custom object for custom object tabs (e.g., 'MyCustomObject\_\_c'), a standard object name (e.g., 'Account'), the name of a Visualforce tab (e.g., 'My\_VF\_Tab\_\_tab'), or a Web Tab name.  
* Visibility: A picklist field indicating the tab's visibility level, with common values being 'Visible' (tab is visible and in the default navigation), 'Available' (tab is available in App Launcher but not in default navigation), or 'None' (tab is hidden).

Since PermissionSetTabSetting is a Tooling API object, standard SOQL queries used for data SObjects will not work. The reporting program must be capable of making Tooling API calls (e.g., /services/data/vXX.X/tooling/query/?q=...). The Name field in PermissionSetTabSetting refers to the tab's API name. To display a user-friendly label in the report, this API name might need to be mapped by querying the TabDefinition object (also via Tooling API) or by maintaining a separate mapping.

*Table III.D.1: PermissionSetTabSetting (Tooling API) \- Key Fields*

| Field API Name | Data Type | Description/Purpose |
| :---- | :---- | :---- |
| Id | ID | Unique identifier for the PermissionSetTabSetting record. |
| ParentId | Reference | ID of the parent PermissionSet. References PermissionSet.Id. |
| Name | String | DeveloperName of the Tab Definition (e.g., 'Account', 'MyCustomObject\_\_c', 'My\_VF\_Tab\_\_tab'). |
| Visibility | Picklist | Tab visibility setting. Values: 'Visible', 'Available', 'None'. |

### **E. Record Type Visibility**

Managing and reporting on Record Type visibility within permission sets presents a different challenge compared to other permissions. Direct SOQL querying for a comprehensive list of all record type visibilities across all permission sets and objects is not straightforward using standard SObjects. The SetupEntityAccess object does not typically list 'RecordType' as a SetupEntityType for the purpose of assigning visibility within a permission set.18

While Apex Schema.DescribeSObjectResult.getRecordTypeInfos() can be used to check record type availability for the *current running user's context*, this method is not suitable for generating a global report across all permission sets.28

The most authoritative and comprehensive way to retrieve record type visibility settings for permission sets is through the **Metadata API**. The PermissionSet metadata type includes a complex type called recordTypeVisibilities. Each entry within this structure specifies the record type (e.g., Account.Business\_Account), the object it belongs to, whether it's visible, and if it's the default record type for that object within the context of that permission set.26

Therefore, a program aiming to report on record type visibilities per permission set would need to:

1. Use the Metadata API to retrieve the .permissionset XML definition for each permission set.  
2. Parse the recordTypeVisibilities section of the XML to extract the relevant details.

While the RecordType SObject itself exists and can be queried 24, its visibility assignments within permission sets are embedded within the permission set's metadata structure rather than being managed through a separate, easily queryable junction SObject like ObjectPermissions or FieldPermissions. This distinction is crucial for developers choosing the correct API strategy.

### **F. User Permissions (General/System)**

As detailed in Section II.A, many general user permissions (also known as system permissions) are stored as direct boolean fields on the PermissionSet SObject itself. These fields typically begin with the prefix "Permissions" (e.g., PermissionsApiEnabled, PermissionsViewAllData, PermissionsManageUsers).7 Querying these fields directly on the PermissionSet object is a straightforward way to determine if these specific permissions are enabled.23

Alternatively, the Metadata API provides a comprehensive way to retrieve all user permissions associated with a permission set. The PermissionSet metadata type contains a userPermissions complex type. Each element within userPermissions typically has two key child elements: name (the API name of the user permission, e.g., 'ApiEnabled') and enabled (a boolean value indicating if the permission is active).26

For reporting purposes, a dual approach can be considered:

* **SOQL on PermissionSet:** Efficient if the report needs to check for a known, limited set of user permissions.  
* **Metadata API:** More robust for retrieving *all* user permissions associated with a permission set, especially if the full list of possible permission API names is not known beforehand or to ensure future compatibility as Salesforce introduces new permissions.

A mapping between the API names of these user permissions and their user-interface (UI) labels is essential for creating human-readable reports. Resources like the list provided in snippet 7, which maps API names like ApiEnabled to "API Enabled", are invaluable for this purpose.

*Table III.F.1: Common User Permission API Names (on PermissionSet object) and UI Labels 7*

| User Permission API Name (on PermissionSet) | Corresponding UI Label |
| :---- | :---- |
| PermissionsApiEnabled | API Enabled |
| PermissionsViewAllData | View All Data |
| PermissionsModifyAllData | Modify All Data |
| PermissionsManageUsers | Manage Users |
| PermissionsCustomizeApplication | Customize Application |
| PermissionsAuthorApex | Author Apex |
| PermissionsRunReports | Run Reports |
| PermissionsExportReport | Export Reports |
| PermissionsManagePublicList Views | Manage Public List Views |
| PermissionsManageCustomReportTypes | Manage Custom Report Types |
| PermissionsPasswordNeverExpires | Password Never Expires |
| PermissionsSendSitRequests | Send Outbound Messages |

## **IV. API Strategies for Data Extraction**

Extracting a complete view of Salesforce permission set information requires leveraging multiple Salesforce APIs, as different pieces of data reside in objects or metadata structures accessible through distinct mechanisms. A hybrid approach is generally essential for comprehensive reporting.

### **A. Standard APIs (SOAP/REST)**

The standard Salesforce SOAP and REST APIs are the primary tools for querying regular SObjects. These APIs are well-suited for retrieving data from:

* PermissionSet (for its direct attributes like Name, Label, IsOwnedByProfile, and its numerous boolean Permissions\* fields for user/system permissions).  
* PermissionSetAssignment (to identify which users are assigned to which permission sets and any expiration dates).  
* ObjectPermissions (for object-level CRUD permissions).  
* FieldPermissions (for field-level read/edit permissions).

These APIs allow the use of Salesforce Object Query Language (SOQL) for precise data retrieval, including filtering with WHERE clauses and traversing relationships (e.g., Parent.Name from ObjectPermissions to get the PermissionSet name). Most of the SOQL examples provided in sources like 38 and 4 are intended for these standard APIs.

### **B. Tooling API**

The Tooling API provides access to metadata and setup objects that are not typically available through the standard data APIs. It is necessary for querying:

* SetupEntityAccess: This is a key Tooling API object for determining access to Apex classes, Visualforce pages, custom permissions, connected applications, service presence statuses, and other setup entities.17  
* PermissionSetTabSetting: This Tooling API object is used to retrieve custom tab visibility settings within a permission set.27  
* Other setup-related objects like ApexClass, ApexPage, CustomPermission, TabDefinition can also be queried via the Tooling API to resolve names and other details from IDs obtained via SetupEntityAccess or PermissionSetTabSetting.

Tooling API SOQL is syntactically similar to standard SOQL but operates on a different set of SObjects. Client programs will need to target the Tooling API endpoint (e.g., /services/data/vXX.X/tooling/).

### **C. Metadata API**

The Metadata API is essential for retrieving the complete XML definitions of permission sets, particularly when certain configurations are not easily or comprehensively queryable via SOQL through the standard or Tooling APIs. Its primary use cases in this context include:

* applicationVisibilities: To get a definitive list of applications made visible through a permission set.26  
* recordTypeVisibilities: The most reliable way to get a global view of record type access granted by a permission set.26  
* userPermissions: To retrieve a complete list of all user/system permissions and their enabled status, especially useful if not all Permissions\* boolean field names on the PermissionSet object are known or to ensure all are captured.26  
* Other nested structures within the PermissionSet metadata like classAccesses (Apex class access), pageAccesses (Visualforce page access), fieldPermissions (FLS), and objectPermissions (object CRUD) are also available, offering an alternative way to retrieve this information, though often SOQL/Tooling API is more direct for these.

Using the Metadata API typically involves making a retrieve() call for PermissionSet components, which returns XML files (e.g., MyPermissionSet.permissionset). These files must then be parsed by the client program. It's important to note that while the Metadata API can show Service Presence Status access in a Profile's metadata, it might not directly show it in a Permission Set's metadata, making SetupEntityAccess the more reliable source for this specific permission when assigned via Permission Sets.30

### **D. Considerations for Client Programs (Apex, Java, Python)**

When developing external programs (Java, Python, etc.) to interact with these APIs:

* **Authentication:** An OAuth 2.0 authorization flow is required. For server-to-server integrations, the JWT Bearer Flow is common. For applications requiring user interaction for login, the Web Server Flow or User-Agent Flow might be used. Secure storage and management of client secrets and access tokens are paramount. Apex running within Salesforce operates under the user's context, simplifying authentication for internal tools. All users, including integration users, must have the "API Enabled" permission to use Salesforce APIs.8  
* **API Calls and Parsing:**  
  * **Java:** Libraries like Apache HttpClient or OkHttp can be used for making HTTP requests to REST/Tooling/Metadata APIs. JSON parsing can be handled by Jackson or GSON, and XML parsing by JAXB or standard Java XML APIs.  
  * **Python:** The requests library is standard for HTTP calls. The built-in json module handles JSON, and xml.etree.ElementTree or lxml can parse XML. Libraries like simple-salesforce abstract many of these details for easier interaction with Salesforce APIs.  
  * **Apex:** HttpRequest and HttpResponse classes are used for callouts to external services or other Salesforce APIs (like Tooling or Metadata, though direct Apex access to retrieve full metadata XMLs is complex and often involves workarounds or specific Metadata DML methods rather than raw API calls). SOQL is natively supported. JSON parsing is available via JSON.deserialize and XML via Dom.Document.

The choice of API and programming language will depend on the specific requirements of the reporting tool, the developer's expertise, and the environment in which the tool will run.

## **V. Consolidating Data for a Comprehensive Report**

A significant challenge in generating a comprehensive permission set report is that the necessary data is fragmented across multiple Salesforce SObjects and potentially different APIs. The reporting program must therefore consolidate this information effectively.

The primary key for linking most of these datasets is the PermissionSet.Id.

* PermissionSetAssignment records link to PermissionSet via PermissionSetId.  
* ObjectPermissions, FieldPermissions, and SetupEntityAccess records all link back to the PermissionSet via their respective ParentId fields.

For data retrieved from SetupEntityAccess, the SetupEntityId often needs to be used for a secondary lookup to get the actual name or developer name of the entity. For example, after fetching SetupEntityAccess records where SetupEntityType \= 'ApexClass', the program would collect all SetupEntityId values and then query the ApexClass object WHERE Id IN (...) to get the class names.

Data obtained from the Metadata API (typically XML files) will need to be parsed. The parsed information (e.g., application visibilities, record type visibilities from a .permissionset file) must then be correlated with the appropriate PermissionSet record, usually by matching the file name (which corresponds to the PermissionSet.Name) or by ensuring the retrieve request was for specific permission sets.

The core technical task after data extraction involves "stitching" these disparate datasets together. For instance, to present all permissions for a specific user, the program might start by querying PermissionSetAssignment for that User.Id, then for each assigned PermissionSet.Id, query ObjectPermissions, FieldPermissions, SetupEntityAccess, and parse relevant Metadata API information. This data stitching requires careful handling of IDs and relationships to build a coherent view.

When structuring the output, consider:

* A primary structure organized per Permission Set.  
* Within each Permission Set, nested structures or lists for:  
  * Assigned Users (including expiration dates).  
  * Object Permissions (listing objects and their CRUD/View All/Modify All status).  
  * Field Permissions (listing object, field, and read/edit status).  
  * Apex Class Access (list of accessible classes).  
  * Visualforce Page Access (list of accessible pages).  
  * Custom Permissions (list of enabled custom permissions).  
  * Assigned/Connected Applications.  
  * Record Type Visibilities.  
  * Tab Visibilities.  
  * Service Presence Statuses.  
  * General User Permissions.

Output formats like JSON are well-suited for hierarchical data. For CSV, the data may need to be flattened, potentially resulting in multiple rows per permission set to represent its various assigned artifacts. Loading the consolidated data into a relational database could also allow for more flexible ad-hoc querying and reporting. The performance of this consolidation phase, especially if done client-side in memory after fetching raw data, can be a concern for organizations with a large number of users, permission sets, and schema elements. Efficient data structures (e.g., maps for quick lookups by ID) and algorithms will be crucial.

## **VI. Key Considerations and Best Practices**

Developing a program to extract Salesforce permission set information requires attention to several operational and technical aspects to ensure reliability, performance, and security.

* **API Governor Limits:** Salesforce enforces governor limits on API calls (per user, per org, per time window), SOQL queries (number of queries, rows returned), and other resources. Programs must be designed with bulkification in mind.31 This involves:  
  * Querying for multiple permission sets or users in a single call where feasible (e.g., using IN clauses with collections of IDs).  
  * Using selective WHERE clauses in SOQL queries to reduce the amount of data processed.  
  * Retrieving only necessary fields. The Tooling and Metadata APIs have their own specific sets of limits that differ from standard data API limits, and these must be consulted and adhered to.  
* **Query Performance:**  
  * Ensure that fields used in WHERE clauses (especially ParentId, SetupEntityType in SetupEntityAccess, SobjectType, Field in FieldPermissions) are indexed if possible, or that queries are structured to leverage standard indexes.  
  * Be mindful of relationship query limits (e.g., SOQL subqueries are limited in depth and number).  
  * For large datasets, consider using the Bulk API for extracting large volumes of SObject data if direct SOQL becomes too slow or hits timeouts, although this is more for raw data extraction than complex relational queries.  
* **Large Data Volumes (LDV):**  
  * For orgs with many users, permission sets, objects, and fields, the volume of permission-related records can be substantial.  
  * Utilize pagination mechanisms provided by the APIs (e.g., queryMore() in SOAP/REST, OFFSET in SOQL where appropriate, or cursor-based pagination for some Bulk API jobs) to process results in manageable chunks.  
* **Error Handling and Idempotency:**  
  * Implement robust error handling for API calls, including network issues, authentication failures, API-specific errors (e.g., malformed queries, invalid session IDs), and governor limit exceptions.  
  * Consider implementing retry mechanisms with exponential backoff for transient errors.  
  * For long-running extraction processes or batch jobs, design operations to be idempotent where possible, so that retrying a failed chunk does not lead to duplicate data or incorrect aggregations.  
* **Security:**  
  * The user account (integration user) whose credentials the reporting program uses must have sufficient permissions to query all the necessary SObjects (PermissionSet, PermissionSetAssignment, ObjectPermissions, FieldPermissions, User, Profile, ApexClass, ApexPage, CustomPermission, etc.) and to call the Tooling and Metadata APIs. Grant the principle of least privilege.  
  * Securely store and manage any Salesforce credentials, client secrets, or access tokens used by the program. Avoid hardcoding credentials.  
  * Be mindful of the sensitivity of the data being extracted. The generated reports will contain detailed security configuration information and should be protected accordingly.  
* **Change Management:**  
  * Salesforce is an evolving platform with three major releases per year. New permissions, SObjects, fields, or API behaviors may be introduced. The reporting program should be designed with some adaptability in mind, and regular review against release notes is advisable.

By adhering to these best practices, developers can create more robust, efficient, and secure programs for reporting on Salesforce permission sets.

## **VII. Conclusion and Further Resources**

Extracting comprehensive Salesforce Permission Set information programmatically requires a multi-faceted approach, leveraging a combination of Standard Data APIs (SOAP/REST), the Tooling API, and the Metadata API. No single API provides all the necessary details.

* The **PermissionSet** object is central, holding basic attributes and many direct user permissions.  
* **PermissionSetAssignment** links permission sets to users and includes expiration details.  
* **ObjectPermissions** and **FieldPermissions** define CRUD and FLS respectively.  
* The **SetupEntityAccess** object (queried via Tooling API) is key for Apex class, Visualforce page, custom permission, connected application, and service presence status access, using SetupEntityType to differentiate.  
* **PermissionSetTabSetting** (Tooling API) details custom tab visibility.  
* The **Metadata API** is often the most reliable source for application visibility and record type visibility configurations within permission sets, and for a comprehensive list of all user permissions.

Developers building such reporting tools must carefully consider data consolidation strategies, API governor limits, query performance, and security. The ability to accurately and efficiently report on permission set configurations is invaluable for maintaining security, ensuring compliance, and managing user access effectively in any Salesforce organization.

For the most current and detailed information, always refer to the official Salesforce Developer Documentation:

* **Salesforce Object Reference Guide:** For details on standard and Tooling API SObjects and their fields. (e.g.32)  
* **Salesforce Metadata API Developer Guide:** For the structure of metadata types, including PermissionSet. (e.g.26)  
* **Salesforce Tooling API Developer Guide:** For information on Tooling API objects and calls. (e.g.36)  
* **Salesforce Security Guide:** For general concepts on Salesforce permissions and security. (e.g.2)

Community resources such as the Salesforce Stack Exchange and developer blogs can also provide practical examples and insights into working with these APIs and objects.1

#### **Sources des citations**

1. How to Review Field-Level Security in Salesforce Using Permission ..., consulté le mai 14, 2025, [https://www.salesforceben.com/how-to-review-field-level-security-in-salesforce-using-permission-sets/](https://www.salesforceben.com/how-to-review-field-level-security-in-salesforce-using-permission-sets/)  
2. Object Permissions | Salesforce Security Guide | Salesforce ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.securityImplGuide.meta/securityImplGuide/users\_profiles\_object\_perms.htm](https://developer.salesforce.com/docs/atlas.en-us.securityImplGuide.meta/securityImplGuide/users_profiles_object_perms.htm)  
3. Set Field Permissions in Permission Sets and Profiles | Salesforce Security Guide, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.securityImplGuide.meta/securityImplGuide/users\_profiles\_fls.htm](https://developer.salesforce.com/docs/atlas.en-us.securityImplGuide.meta/securityImplGuide/users_profiles_fls.htm)  
4. soql on object permissions on a profile \- Salesforce Stack Exchange, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/237238/soql-on-object-permissions-on-a-profile](https://salesforce.stackexchange.com/questions/237238/soql-on-object-permissions-on-a-profile)  
5. Permissions starting with 'X' character in soql query results \- Salesforce Help, consulté le mai 14, 2025, [https://help.salesforce.com/s/articleView?id=000387815\&language=en\_US\&type=1](https://help.salesforce.com/s/articleView?id=000387815&language=en_US&type=1)  
6. PermissionSetLicense | Object Reference for the Salesforce Platform, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_permissionsetlicense.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetlicense.htm)  
7. Challenge: Mapping between Salesforce Permission Name and the ..., consulté le mai 14, 2025, [https://www.sfdcamplified.com/challenge-map-between-salesforce-permissionname-and-label/](https://www.sfdcamplified.com/challenge-map-between-salesforce-permissionname-and-label/)  
8. Security and the API | SOAP API Developer Guide \- Salesforce Developers, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce\_api\_concepts\_security.htm](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_concepts_security.htm)  
9. Manage Permission Set Assignments \- Salesforce Help, consulté le mai 14, 2025, [https://help.salesforce.com/s/articleView?id=platform.perm\_sets\_manage\_assignments.htm\&language=en\_US\&type=5](https://help.salesforce.com/s/articleView?id=platform.perm_sets_manage_assignments.htm&language=en_US&type=5)  
10. Salesforce : Query to find Permission sets assigned to users \- JSBC Labs, consulté le mai 14, 2025, [https://www.jsbclabs.com/technology/6277/](https://www.jsbclabs.com/technology/6277/)  
11. Enable Permission Set Expiration and Enhanced UI (Release Update) \- Salesforce Help, consulté le mai 14, 2025, [https://help.salesforce.com/s/articleView?id=release-notes.rn\_forcecom\_enable\_permset\_expiration\_ru.htm\&language=en\_US\&release=238\&type=5](https://help.salesforce.com/s/articleView?id=release-notes.rn_forcecom_enable_permset_expiration_ru.htm&language=en_US&release=238&type=5)  
12. Assign a Permission Set to Multiple Users | Salesforce Security Guide, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.securityImplGuide.meta/securityImplGuide/perm\_sets\_mass\_assign.htm](https://developer.salesforce.com/docs/atlas.en-us.securityImplGuide.meta/securityImplGuide/perm_sets_mass_assign.htm)  
13. Object Permissions \- B... | SFXD Wiki, consulté le mai 14, 2025, [https://wiki.sfxd.org/books/best-practices/page/object-permissions---basic-functionality](https://wiki.sfxd.org/books/best-practices/page/object-permissions---basic-functionality)  
14. Update custom Profile and Permission Set object level permissions with Data Loader, consulté le mai 14, 2025, [https://help.salesforce.com/s/articleView?id=000381102\&language=en\_US\&type=1](https://help.salesforce.com/s/articleView?id=000381102&language=en_US&type=1)  
15. Salesforce Standard Objects and Fields · GitHub, consulté le mai 14, 2025, [https://gist.github.com/surajp/75e4b283479e066e2044b7208f8f980d](https://gist.github.com/surajp/75e4b283479e066e2044b7208f8f980d)  
16. SOQL Query on FieldPermissions \- Salesforce Stack Exchange, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/417073/soql-query-on-fieldpermissions](https://salesforce.stackexchange.com/questions/417073/soql-query-on-fieldpermissions)  
17. Is it possible to query the Apex Class Accesses of a permission set ..., consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/270297/is-it-possible-to-query-the-apex-class-accesses-of-a-permission-set](https://salesforce.stackexchange.com/questions/270297/is-it-possible-to-query-the-apex-class-accesses-of-a-permission-set)  
18. How do you / can you assign a Record Type to a Permission Set in Apex code?, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/83576/how-do-you-can-you-assign-a-record-type-to-a-permission-set-in-apex-code](https://salesforce.stackexchange.com/questions/83576/how-do-you-can-you-assign-a-record-type-to-a-permission-set-in-apex-code)  
19. Connected Apps \- sfdc techie – Pavan's blog, consulté le mai 14, 2025, [https://sfdctechie.wordpress.com/2018/08/12/connected-apps/](https://sfdctechie.wordpress.com/2018/08/12/connected-apps/)  
20. How to retrieve the available omni | Salesforce Trailblazer Community, consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54S00000A8xf5SAB](https://trailhead.salesforce.com/trailblazer-community/feed/0D54S00000A8xf5SAB)  
21. Creating, Assigning and Checking Custom Permissions | Andy in the ..., consulté le mai 14, 2025, [https://andyinthecloud.com/2015/01/14/creating-assigning-and-checking-custom-permissions/](https://andyinthecloud.com/2015/01/14/creating-assigning-and-checking-custom-permissions/)  
22. SOQL to find All Users with a Custom Permission \- Salesforce Stack Exchange, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/138561/soql-to-find-all-users-with-a-custom-permission](https://salesforce.stackexchange.com/questions/138561/soql-to-find-all-users-with-a-custom-permission)  
23. Query Author Apex Permission? \- soql \- Salesforce StackExchange, consulté le mai 14, 2025, [https://salesforce.stackexchange.com/questions/145427/query-author-apex-permission](https://salesforce.stackexchange.com/questions/145427/query-author-apex-permission)  
24. CData MCP Server for Salesforce \- SetupEntityAccess, consulté le mai 14, 2025, [https://cdn.cdata.com/help/RFK/mcp/pg\_table-setupentityaccess.htm](https://cdn.cdata.com/help/RFK/mcp/pg_table-setupentityaccess.htm)  
25. Query to List Permission Sets with Access to Visualforce Page \- ServiceMax Knowledge, consulté le mai 14, 2025, [https://community.servicemax.com/s/article/Query-to-List-Permission-Sets-with-Access-to-Visualforce-Page](https://community.servicemax.com/s/article/Query-to-List-Permission-Sets-with-Access-to-Visualforce-Page)  
26. PermissionSet | Metadata API Developer Guide | Salesforce ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.api\_meta.meta/api\_meta/meta\_permissionset.htm](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_permissionset.htm)  
27. How to query Users who have access | Salesforce Trailblazer ..., consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T41hfSAB](https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T41hfSAB)  
28. Query recordtype access in profiles | Salesforce Trailblazer Community \- Trailhead, consulté le mai 14, 2025, [https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T4NzWSAV](https://trailhead.salesforce.com/trailblazer-community/feed/0D54V00007T4NzWSAV)  
29. SOQL – Salesforce Object Query Language \- salesforce888 \- WordPress.com, consulté le mai 14, 2025, [https://salesforce888.wordpress.com/2015/01/02/soql-salesforce-object-query-language/](https://salesforce888.wordpress.com/2015/01/02/soql-salesforce-object-query-language/)  
30. Service Presence Statuses Access on Profiles and Permission Sets, consulté le mai 14, 2025, [https://docs.copado.com/articles/copado-ci-cd-publication/service-presence-statuses-access-on-profiles-and-permission-sets](https://docs.copado.com/articles/copado-ci-cd-publication/service-presence-statuses-access-on-profiles-and-permission-sets)  
31. An Admin's Guide to SOQL \+ Examples \- Salesforce Ben, consulté le mai 14, 2025, [https://www.salesforceben.com/admins-guide-to-salesforce-soql/](https://www.salesforceben.com/admins-guide-to-salesforce-soql/)  
32. PermissionSet | Object Reference for the Salesforce Platform, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_permissionset.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionset.htm)  
33. ObjectPermissions | Object Reference for the Salesforce Platform ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_objectpermissions.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_objectpermissions.htm)  
34. PermissionSetAssignment | Object Reference for the Salesforce ..., consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_permissionsetassignment.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetassignment.htm)  
35. SetupEntityAccess | Object Reference for the Salesforce Platform, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.object\_reference.meta/object\_reference/sforce\_api\_objects\_setupentityaccess.htm](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_setupentityaccess.htm)  
36. PermissionSetGroupComponent | Tooling API \- Salesforce Developers, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.api\_tooling.meta/api\_tooling/tooling\_api\_objects\_permissionsetgroupcomponent.htm](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_permissionsetgroupcomponent.htm)  
37. User Permissions | Salesforce Security Guide, consulté le mai 14, 2025, [https://developer.salesforce.com/docs/atlas.en-us.securityImplGuide.meta/securityImplGuide/admin\_userperms.htm](https://developer.salesforce.com/docs/atlas.en-us.securityImplGuide.meta/securityImplGuide/admin_userperms.htm)  
38. Salesforce: Query Fields Permission \- SimplySfdc.com, consulté le mai 14, 2025, [https://www.simplysfdc.com/2019/03/salesforce-query-field-permission.html](https://www.simplysfdc.com/2019/03/salesforce-query-field-permission.html)
