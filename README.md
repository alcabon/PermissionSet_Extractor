# PermissionSet_Extractor

Okay, this is an excellent and well-defined starting point for building a Java application using the Salesforce Web Services Connector (WSC) to retrieve comprehensive permission set information. The POJO structure is clear and covers many important aspects of Salesforce permissions.

Let's break down how we can approach building this program, focusing on the two main phases you've outlined:

Phase 1: Data Retrieval using Salesforce WSC and SOQL  
Phase 2: Structuring Data into your POJOs  
I'll provide a conceptual outline of the Java code, SOQL queries, and considerations for each step.

**Prerequisites:**

1. **Salesforce WSC SDK:** Ensure you have the force-wsc.jar (and its dependencies like STAX API, JAX-WS) in your Java project's classpath.  
2. **Salesforce Connected App (Optional but Recommended for OAuth):** For authentication, especially for server-to-server integration, setting up a Connected App for OAuth 2.0 (JWT Bearer Flow or Username-Password Flow) is good practice. For simplicity in this example, I might initially show username-password, but be aware of its security implications.  
3. **Salesforce Org Access:** Credentials (username, password+security token, or OAuth details) for a Salesforce org with API access.

---

**Phase 1: Data Retrieval**

This phase involves connecting to Salesforce and executing SOQL queries to fetch the raw data.

**Core Idea:** We'll query the PermissionSet object first. Then, for each relevant PermissionSet, we'll query its related permission details (ObjectPermissions, FieldPermissions, SetupEntityAccess for Apex/VF/CustomPermissions, etc.).

**1.1. Establish Salesforce Connection**


```java
import com.sforce.soap.partner.PartnerConnection;  
import com.sforce.soap.partner.Connector;  
import com.sforce.soap.partner.DescribeSObjectResult;  
import com.sforce.soap.partner.Field;  
import com.sforce.soap.partner.FieldType;  
import com.sforce.soap.partner.QueryResult;  
import com.sforce.soap.partner.sobject.SObject;  
import com.sforce.ws.ConnectionException;  
import com.sforce.ws.ConnectorConfig;

import java.util.ArrayList;  
import java.util.HashMap;  
import java.util.List;  
import java.util.Map;  
import java.util.stream.Collectors;

// ... (Your POJO classes will be defined here or imported)  
// Main container for all information about a single Permission Set  
class ComprehensivePermissionSet {  
    // Core PermissionSet fields  
    String id;  
    String name;  
    String label;  
    String description;  
    boolean isCustom;  
    boolean isOwnedByProfile;  
    String type;  
    String namespacePrefix;  
    String licenseName; // Changed from licenseId to store License Name  
    String profileName; // If IsOwnedByProfile is true

    // User Permissions (direct boolean fields from PermissionSet object)  
    List<UserPermissionDetail> userPermissions = new ArrayList<>();

    // Detailed Permissions  
    List<ObjectPermissionDetail> objectPermissions = new ArrayList<>();  
    List<FieldPermissionDetail> fieldPermissions = new ArrayList<>();  
    List<ApexClassAccessDetail> apexClassAccesses = new ArrayList<>();  
    List<VisualForcePageAccessDetail> vfPageAccesses = new ArrayList<>();  
    List<CustomPermissionAssignmentDetail> customPermissionAssignments = new ArrayList<>();  
    List<ApplicationVisibilityDetail> applicationVisibilities = new ArrayList<>();  
    List<TabSettingDetail> tabSettings = new ArrayList<>();  
    List<ConnectedAppAccessDetail> connectedAppAccesses = new ArrayList<>();  
    List<ServicePresenceStatusAccessDetail> servicePresenceStatusAccesses = new ArrayList<>();

    // Constructors, getters, setters omitted for brevity but are essential  
    // Standard getters and setters for all fields should be implemented.  
    // Example:  
    public String getId() { return id; }  
    public void setId(String id) { this.id = id; }  
    public String getName() { return name; }  
    public void setName(String name) { this.name = name; }  
    public String getLabel() { return label; }  
    public void setLabel(String label) { this.label = label; }  
    public String getDescription() { return description; }  
    public void setDescription(String description) { this.description = description; }  
    public boolean isCustom() { return isCustom; }  
    public void setCustom(boolean custom) { isCustom = custom; }  
    public boolean isOwnedByProfile() { return isOwnedByProfile; }  
    public void setOwnedByProfile(boolean ownedByProfile) { isOwnedByProfile = ownedByProfile; }  
    public String getType() { return type; }  
    public void setType(String type) { this.type = type; }  
    public String getNamespacePrefix() { return namespacePrefix; }  
    public void setNamespacePrefix(String namespacePrefix) { this.namespacePrefix = namespacePrefix; }  
    public String getLicenseName() { return licenseName; }  
    public void setLicenseName(String licenseName) { this.licenseName = licenseName; }  
    public String getProfileName() { return profileName; }  
    public void setProfileName(String profileName) { this.profileName = profileName; }  
    public List<UserPermissionDetail> getUserPermissions() { return userPermissions; }  
    public void setUserPermissions(List<UserPermissionDetail> userPermissions) { this.userPermissions = userPermissions; }  
    public List<ObjectPermissionDetail> getObjectPermissions() { return objectPermissions; }  
    public void setObjectPermissions(List<ObjectPermissionDetail> objectPermissions) { this.objectPermissions = objectPermissions; }  
    public List<FieldPermissionDetail> getFieldPermissions() { return fieldPermissions; }  
    public void setFieldPermissions(List<FieldPermissionDetail> fieldPermissions) { this.fieldPermissions = fieldPermissions; }  
    public List<ApexClassAccessDetail> getApexClassAccesses() { return apexClassAccesses; }  
    public void setApexClassAccesses(List<ApexClassAccessDetail> apexClassAccesses) { this.apexClassAccesses = apexClassAccesses; }  
    public List<VisualForcePageAccessDetail> getVfPageAccesses() { return vfPageAccesses; }  
    public void setVfPageAccesses(List<VisualForcePageAccessDetail> vfPageAccesses) { this.vfPageAccesses = vfPageAccesses; }  
    public List<CustomPermissionAssignmentDetail> getCustomPermissionAssignments() { return customPermissionAssignments; }  
    public void setCustomPermissionAssignments(List<CustomPermissionAssignmentDetail> customPermissionAssignments) { this.customPermissionAssignments = customPermissionAssignments; }  
    public List<ApplicationVisibilityDetail> getApplicationVisibilities() { return applicationVisibilities; }  
    public void setApplicationVisibilities(List<ApplicationVisibilityDetail> applicationVisibilities) { this.applicationVisibilities = applicationVisibilities; }  
    public List<TabSettingDetail> getTabSettings() { return tabSettings; }  
    public void setTabSettings(List<TabSettingDetail> tabSettings) { this.tabSettings = tabSettings; }  
    public List<ConnectedAppAccessDetail> getConnectedAppAccesses() { return connectedAppAccesses; }  
    public void setConnectedAppAccesses(List<ConnectedAppAccessDetail> connectedAppAccesses) { this.connectedAppAccesses = connectedAppAccesses; }  
    public List<ServicePresenceStatusAccessDetail> getServicePresenceStatusAccesses() { return servicePresenceStatusAccesses; }  
    public void setServicePresenceStatusAccesses(List<ServicePresenceStatusAccessDetail> servicePresenceStatusAccesses) { this.servicePresenceStatusAccesses = servicePresenceStatusAccesses; }  
}

class UserPermissionDetail {  
    String apiName; String uiLabel; boolean isEnabled;  
    // Getters/Setters  
    public String getApiName() { return apiName; }  
    public void setApiName(String apiName) { this.apiName = apiName; }  
    public String getUiLabel() { return uiLabel; }  
    public void setUiLabel(String uiLabel) { this.uiLabel = uiLabel; }  
    public boolean isEnabled() { return isEnabled; }  
    public void setEnabled(boolean enabled) { isEnabled = enabled; }  
}  
class ObjectPermissionDetail {  
    String sobjectType; boolean canRead, canCreate, canEdit, canDelete, canViewAllRecords, canModifyAllRecords;  
    // Getters/Setters (Ensure all boolean getters are `isCanRead`, etc.)  
    public String getSobjectType() { return sobjectType; }  
    public void setSobjectType(String sobjectType) { this.sobjectType = sobjectType; }  
    public boolean isCanRead() { return canRead; }  
    public void setCanRead(boolean canRead) { this.canRead = canRead; }  
    public boolean isCanCreate() { return canCreate; }  
    public void setCanCreate(boolean canCreate) { this.canCreate = canCreate; }  
    public boolean isCanEdit() { return canEdit; }  
    public void setCanEdit(boolean canEdit) { this.canEdit = canEdit; }  
    public boolean isCanDelete() { return canDelete; }  
    public void setCanDelete(boolean canDelete) { this.canDelete = canDelete; }  
    public boolean isCanViewAllRecords() { return canViewAllRecords; }  
    public void setCanViewAllRecords(boolean canViewAllRecords) { this.canViewAllRecords = canViewAllRecords; }  
    public boolean isCanModifyAllRecords() { return canModifyAllRecords; }  
    public void setCanModifyAllRecords(boolean canModifyAllRecords) { this.canModifyAllRecords = canModifyAllRecords; }  
}  
class FieldPermissionDetail {  
    String sobjectType; String fieldApiName; boolean canRead, canEdit;  
    // Getters/Setters  
    public String getSobjectType() { return sobjectType; }  
    public void setSobjectType(String sobjectType) { this.sobjectType = sobjectType; }  
    public String getFieldApiName() { return fieldApiName; }  
    public void setFieldApiName(String fieldApiName) { this.fieldApiName = fieldApiName; }  
    public boolean isCanRead() { return canRead; }  
    public void setCanRead(boolean canRead) { this.canRead = canRead; }  
    public boolean isCanEdit() { return canEdit; }  
    public void setCanEdit(boolean canEdit) { this.canEdit = canEdit; }  
}  
class ApexClassAccessDetail {  
    String apexClassId; String apexClassName; String namespacePrefix;  
    // Getters/Setters  
    public String getApexClassId() { return apexClassId; }  
    public void setApexClassId(String apexClassId) { this.apexClassId = apexClassId; }  
    public String getApexClassName() { return apexClassName; }  
    public void setApexClassName(String apexClassName) { this.apexClassName = apexClassName; }  
    public String getNamespacePrefix() { return namespacePrefix; }  
    public void setNamespacePrefix(String namespacePrefix) { this.namespacePrefix = namespacePrefix; }  
}  
class VisualForcePageAccessDetail {  
    String vfPageId; String vfPageName; String namespacePrefix;  
    // Getters/Setters  
    public String getVfPageId() { return vfPageId; }  
    public void setVfPageId(String vfPageId) { this.vfPageId = vfPageId; }  
    public String getVfPageName() { return vfPageName; }  
    public void setVfPageName(String vfPageName) { this.vfPageName = vfPageName; }  
    public String getNamespacePrefix() { return namespacePrefix; }  
    public void setNamespacePrefix(String namespacePrefix) { this.namespacePrefix = namespacePrefix; }  
}  
class CustomPermissionAssignmentDetail {  
    String customPermissionId; String customPermissionDeveloperName; String customPermissionLabel;  
    // Getters/Setters  
    public String getCustomPermissionId() { return customPermissionId; }  
    public void setCustomPermissionId(String customPermissionId) { this.customPermissionId = customPermissionId; }  
    public String getCustomPermissionDeveloperName() { return customPermissionDeveloperName; }  
    public void setCustomPermissionDeveloperName(String customPermissionDeveloperName) { this.customPermissionDeveloperName = customPermissionDeveloperName; }  
    public String getCustomPermissionLabel() { return customPermissionLabel; }  
    public void setCustomPermissionLabel(String customPermissionLabel) { this.customPermissionLabel = customPermissionLabel; }  
}  
class ApplicationVisibilityDetail {  
    String applicationId; String applicationName; String applicationLabel; String applicationType;  
    // Getters/Setters  
    public String getApplicationId() { return applicationId; }  
    public void setApplicationId(String applicationId) { this.applicationId = applicationId; }  
    public String getApplicationName() { return applicationName; }  
    public void setApplicationName(String applicationName) { this.applicationName = applicationName; }  
    public String getApplicationLabel() { return applicationLabel; }  
    public void setApplicationLabel(String applicationLabel) { this.applicationLabel = applicationLabel; }  
    public String getApplicationType() { return applicationType; }  
    public void setApplicationType(String applicationType) { this.applicationType = applicationType; }  
}  
class TabSettingDetail {  
    String tabNameOrId; String visibility; String tabLabel; // Added TabLabel  
    // Getters/Setters  
    public String getTabNameOrId() { return tabNameOrId; }  
    public void setTabNameOrId(String tabNameOrId) { this.tabNameOrId = tabNameOrId; }  
    public String getVisibility() { return visibility; }  
    public void setVisibility(String visibility) { this.visibility = visibility; }  
    public String getTabLabel() { return tabLabel; }  
    public void setTabLabel(String tabLabel) { this.tabLabel = tabLabel; }  
}  
class ConnectedAppAccessDetail {  
    String connectedAppId; String connectedAppName; String connectedAppLabel;  
    // Getters/Setters  
    public String getConnectedAppId() { return connectedAppId; }  
    public void setConnectedAppId(String connectedAppId) { this.connectedAppId = connectedAppId; }  
    public String getConnectedAppName() { return connectedAppName; }  
    public void setConnectedAppName(String connectedAppName) { this.connectedAppName = connectedAppName; }  
    public String getConnectedAppLabel() { return connectedAppLabel; }  
    public void setConnectedAppLabel(String connectedAppLabel) { this.connectedAppLabel = connectedAppLabel; }  
}  
class ServicePresenceStatusAccessDetail {  
    String statusId; String statusDeveloperName; String statusMasterLabel;  
    // Getters/Setters  
    public String getStatusId() { return statusId; }  
    public void setStatusId(String statusId) { this.statusId = statusId; }  
    public String getStatusDeveloperName() { return statusDeveloperName; }  
    public void setStatusDeveloperName(String statusDeveloperName) { this.statusDeveloperName = statusDeveloperName; }  
    public String getStatusMasterLabel() { return statusMasterLabel; }  
    public void setStatusMasterLabel(String statusMasterLabel) { this.statusMasterLabel = statusMasterLabel; }  
}

public class SalesforcePermissionSetExtractor {

    private PartnerConnection connection;  
    private Map<String, String> userPermissionApiToLabelMap;

    public SalesforcePermissionSetExtractor(String username, String password, String authUrl) throws ConnectionException {  
        ConnectorConfig config = new ConnectorConfig();  
        config.setUsername(username);  
        config.setPassword(password);  
        config.setAuthEndpoint(authUrl);  
        // For production, consider using config.setSessionId() if you already have a session ID (e.g., from OAuth)  
        // config.setServiceEndpoint("https://yourInstance.salesforce.com/services/Soap/u/API_VERSION");  
        this.connection = Connector.newConnection(config);  
        System.out.println("Successfully connected to Salesforce. User: " + config.getUsername() + ", URL: " + connection.getConfig().getServiceEndpoint());  
        this.userPermissionApiToLabelMap = fetchUserPermissionLabels();  
    }

    private Map<String, String> fetchUserPermissionLabels() throws ConnectionException {  
        Map<String, String> labelMap = new HashMap<>();  
        DescribeSObjectResult describeSObjectResult = connection.describeSObject("PermissionSet");  
        for (Field field : describeSObjectResult.getFields()) {  
            if (field.getType() == FieldType._boolean && field.getName().startsWith("Permissions")) {  
                labelMap.put(field.getName(), field.getLabel());  
            }  
        }  
        return labelMap;  
    }

    private List<SObject> queryAll(String soql) throws ConnectionException {  
        List<SObject> records = new ArrayList<>();  
        QueryResult queryResult = connection.query(soql);  
        boolean done = false;  
        while (!done) {  
            for (SObject record : queryResult.getRecords()) {  
                records.add(record);  
            }  
            if (queryResult.isDone()) {  
                done = true;  
            } else {  
                queryResult = connection.queryMore(queryResult.getQueryLocator());  
            }  
        }  
        return records;  
    }

    public List<ComprehensivePermissionSet> fetchAllPermissionSetDetails() throws ConnectionException {  
        List<ComprehensivePermissionSet> comprehensivePermissionSets = new ArrayList<>();  
        Map<String, ComprehensivePermissionSet> permSetMapById = new HashMap<>();

        // --- Query Core PermissionSet Fields and User Permissions ---  
        StringBuilder psQueryBuilder = new StringBuilder("SELECT Id, Name, Label, Description, IsCustom, IsOwnedByProfile, Type, NamespacePrefix, License.Name, Profile.Name");  
        for (String apiName : userPermissionApiToLabelMap.keySet()) {  
            psQueryBuilder.append(", ").append(apiName);  
        }  
        psQueryBuilder.append(" FROM PermissionSet");  
        // Add a WHERE clause if you want to filter (e.g., WHERE IsCustom = true)  
        // psQueryBuilder.append(" WHERE Name = 'Your_Test_PermissionSet_Name'"); // For testing specific perm set

        List<SObject> psRecords = queryAll(psQueryBuilder.toString());

        for (SObject psRecord : psRecords) {  
            ComprehensivePermissionSet cps = new ComprehensivePermissionSet();  
            cps.setId((String) psRecord.getId());  
            cps.setName((String) psRecord.getField("Name"));  
            cps.setLabel((String) psRecord.getField("Label"));  
            cps.setDescription((String) psRecord.getField("Description"));  
            cps.setCustom((Boolean) psRecord.getField("IsCustom"));  
            cps.setOwnedByProfile((Boolean) psRecord.getField("IsOwnedByProfile"));  
            cps.setType((String) psRecord.getField("Type"));  
            cps.setNamespacePrefix((String) psRecord.getField("NamespacePrefix"));

            SObject license = (SObject) psRecord.getField("License");  
            if (license != null) {  
                cps.setLicenseName((String) license.getField("Name"));  
            }

            SObject profile = (SObject) psRecord.getField("Profile");  
            if (profile != null) {  
                cps.setProfileName((String) profile.getField("Name"));  
            }

            // Populate UserPermissions  
            for (Map.Entry<String, String> entry : userPermissionApiToLabelMap.entrySet()) {  
                String apiName = entry.getKey();  
                String uiLabel = entry.getValue();  
                Boolean isEnabled = (Boolean) psRecord.getField(apiName);  
                if (isEnabled != null) { // Salesforce returns null if the field wasn't queried or has no value  
                    UserPermissionDetail upd = new UserPermissionDetail();  
                    upd.setApiName(apiName);  
                    upd.setUiLabel(uiLabel);  
                    upd.setEnabled(isEnabled);  
                    cps.getUserPermissions().add(upd);  
                }  
            }  
            comprehensivePermissionSets.add(cps);  
            permSetMapById.put(cps.getId(), cps);  
        }

        if (permSetMapById.isEmpty()) {  
            return comprehensivePermissionSets; // No permission sets found or matched filter  
        }

        List<String> permissionSetIds = new ArrayList<>(permSetMapById.keySet());

        // --- Query Related Details in Batches if necessary ---  
        // For simplicity, this example queries all at once. In reality, batch `permissionSetIds`.  
        String joinedIds = permissionSetIds.stream().map(id -> "'" + id + "'").collect(Collectors.joining(","));

        fetchObjectPermissions(permSetMapById, joinedIds);  
        fetchFieldPermissions(permSetMapById, joinedIds);  
        fetchSetupEntityAccessDetails(permSetMapById, joinedIds); // Handles Apex, VF, Custom Perms, Connected Apps  
        fetchTabSettings(permSetMapById, joinedIds);  
        fetchServicePresenceStatusAccesses(permSetMapById, joinedIds);  
        // ApplicationVisibilities might need a different strategy or be partially covered by TabSettings

        return comprehensivePermissionSets;  
    }

    private void fetchObjectPermissions(Map<String, ComprehensivePermissionSet> permSetMapById, String joinedParentIds) throws ConnectionException {  
        if (permSetMapById.isEmpty()) return;  
        String soql = "SELECT ParentId, SobjectType, PermissionsRead, PermissionsCreate, PermissionsEdit, PermissionsDelete, PermissionsViewAllRecords, PermissionsModifyAllRecords " +  
                      "FROM ObjectPermissions WHERE ParentId IN (" + joinedParentIds + ")";  
        List<SObject> records = queryAll(soql);  
        for (SObject record : records) {  
            ComprehensivePermissionSet cps = permSetMapById.get((String) record.getField("ParentId"));  
            if (cps != null) {  
                ObjectPermissionDetail detail = new ObjectPermissionDetail();  
                detail.setSobjectType((String) record.getField("SobjectType"));  
                detail.setCanRead((Boolean) record.getField("PermissionsRead"));  
                detail.setCanCreate((Boolean) record.getField("PermissionsCreate"));  
                detail.setCanEdit((Boolean) record.getField("PermissionsEdit"));  
                detail.setCanDelete((Boolean) record.getField("PermissionsDelete"));  
                detail.setCanViewAllRecords((Boolean) record.getField("PermissionsViewAllRecords"));  
                detail.setCanModifyAllRecords((Boolean) record.getField("PermissionsModifyAllRecords"));  
                cps.getObjectPermissions().add(detail);  
            }  
        }  
    }

    private void fetchFieldPermissions(Map<String, ComprehensivePermissionSet> permSetMapById, String joinedParentIds) throws ConnectionException {  
        if (permSetMapById.isEmpty()) return;  
        String soql = "SELECT ParentId, SobjectType, Field, PermissionsRead, PermissionsEdit " +  
                      "FROM FieldPermissions WHERE ParentId IN (" + joinedParentIds + ")";  
        List<SObject> records = queryAll(soql);  
        for (SObject record : records) {  
            ComprehensivePermissionSet cps = permSetMapById.get((String) record.getField("ParentId"));  
            if (cps != null) {  
                FieldPermissionDetail detail = new FieldPermissionDetail();  
                detail.setSobjectType((String) record.getField("SobjectType"));  
                detail.setFieldApiName((String) record.getField("Field")); // This is SobjectType.FieldName  
                detail.setCanRead((Boolean) record.getField("PermissionsRead"));  
                detail.setCanEdit((Boolean) record.getField("PermissionsEdit"));  
                cps.getFieldPermissions().add(detail);  
            }  
        }  
    }

    private void fetchSetupEntityAccessDetails(Map<String, ComprehensivePermissionSet> permSetMapById, String joinedParentIds) throws ConnectionException {  
        if (permSetMapById.isEmpty()) return;  
        String soql = "SELECT ParentId, SetupEntityId, SetupEntityType FROM SetupEntityAccess " +  
                      "WHERE ParentId IN (" + joinedParentIds + ") AND " +  
                      "SetupEntityType IN ('ApexClass', 'ApexPage', 'CustomPermission', 'ConnectedApplication')"; // Add other types if needed

        List<SObject> seaRecords = queryAll(soql);  
        if (seaRecords.isEmpty()) return;

        Map<String, List<SObject>> seaByParentIdAndType = seaRecords.stream()  
            .collect(Collectors.groupingBy(sea -> (String)sea.getField("ParentId") + "_" + (String)sea.getField("SetupEntityType")));

        // Collect all SetupEntityIds by type  
        Map<String, List<String>> idsToQueryByType = new HashMap<>();  
        for (SObject sea : seaRecords) {  
            idsToQueryByType.computeIfAbsent((String) sea.getField("SetupEntityType"), k -> new ArrayList<>())  
                           .add((String) sea.getField("SetupEntityId"));  
        }

        // Fetch details for each type  
        Map<String, SObject> apexClassDetails = fetchDetailsByIds("ApexClass", "Id, Name, NamespacePrefix", idsToQueryByType.get("ApexClass"));  
        Map<String, SObject> apexPageDetails = fetchDetailsByIds("ApexPage", "Id, Name, NamespacePrefix", idsToQueryByType.get("ApexPage"));  
        Map<String, SObject> customPermissionDetails = fetchDetailsByIds("CustomPermission", "Id, DeveloperName, MasterLabel", idsToQueryByType.get("CustomPermission"));  
        Map<String, SObject> connectedAppDetails = fetchDetailsByIds("ConnectedApplication", "Id, Name, Label", idsToQueryByType.get("ConnectedApplication"));

        for (SObject seaRecord : seaRecords) {  
            String parentId = (String) seaRecord.getField("ParentId");  
            String setupEntityId = (String) seaRecord.getField("SetupEntityId");  
            String setupEntityType = (String) seaRecord.getField("SetupEntityType");  
            ComprehensivePermissionSet cps = permSetMapById.get(parentId);

            if (cps != null) {  
                switch (setupEntityType) {  
                    case "ApexClass":  
                        SObject ac = apexClassDetails.get(setupEntityId);  
                        if (ac != null) {  
                            ApexClassAccessDetail detail = new ApexClassAccessDetail();  
                            detail.setApexClassId(setupEntityId);  
                            detail.setApexClassName((String) ac.getField("Name"));  
                            detail.setNamespacePrefix((String) ac.getField("NamespacePrefix"));  
                            cps.getApexClassAccesses().add(detail);  
                        }  
                        break;  
                    case "ApexPage":  
                        SObject ap = apexPageDetails.get(setupEntityId);  
                        if (ap != null) {  
                            VisualForcePageAccessDetail detail = new VisualForcePageAccessDetail();  
                            detail.setVfPageId(setupEntityId);  
                            detail.setVfPageName((String) ap.getField("Name"));  
                            detail.setNamespacePrefix((String) ap.getField("NamespacePrefix"));  
                            cps.getVfPageAccesses().add(detail);  
                        }  
                        break;  
                    case "CustomPermission":  
                        SObject cp = customPermissionDetails.get(setupEntityId);  
                        if (cp != null) {  
                            CustomPermissionAssignmentDetail detail = new CustomPermissionAssignmentDetail();  
                            detail.setCustomPermissionId(setupEntityId);  
                            detail.setCustomPermissionDeveloperName((String) cp.getField("DeveloperName"));  
                            detail.setCustomPermissionLabel((String) cp.getField("MasterLabel"));  
                            cps.getCustomPermissionAssignments().add(detail);  
                        }  
                        break;  
                    case "ConnectedApplication":  
                        SObject ca = connectedAppDetails.get(setupEntityId);  
                        if (ca != null) {  
                            ConnectedAppAccessDetail detail = new ConnectedAppAccessDetail();  
                            detail.setConnectedAppId(setupEntityId);  
                            detail.setConnectedAppName((String) ca.getField("Name"));  
                            detail.setConnectedAppLabel((String) ca.getField("Label"));  
                            cps.getConnectedAppAccesses().add(detail);  
                        }  
                        break;  
                }  
            }  
        }  
    }

    private Map<String, SObject> fetchDetailsByIds(String sObjectType, String fields, List<String> ids) throws ConnectionException {  
        Map<String, SObject> detailsMap = new HashMap<>();  
        if (ids == null || ids.isEmpty()) return detailsMap;

        // Deduplicate IDs before querying  
        List<String> distinctIds = ids.stream().distinct().collect(Collectors.toList());  
        if (distinctIds.isEmpty()) return detailsMap;

        // Batch IDs if necessary (SOQL IN clause limit is high, but good practice for > 2000 IDs for some contexts)  
        int batchSize = 200; // Adjust as needed, can be much larger for IDs  
        for (int i = 0; i < distinctIds.size(); i += batchSize) {  
            List<String> subList = distinctIds.subList(i, Math.min(i + batchSize, distinctIds.size()));  
            String joinedEntityIds = subList.stream().map(id -> "'" + id + "'").collect(Collectors.joining(","));  
            String soql = "SELECT " + fields + " FROM " + sObjectType + " WHERE Id IN (" + joinedEntityIds + ")";  
            List<SObject> records = queryAll(soql);  
            for (SObject record : records) {  
                detailsMap.put((String) record.getId(), record);  
            }  
        }  
        return detailsMap;  
    }

    private void fetchTabSettings(Map<String, ComprehensivePermissionSet> permSetMapById, String joinedParentIds) throws ConnectionException {  
        if (permSetMapById.isEmpty()) return;  
        String soql = "SELECT ParentId, Name, Visibility FROM PermissionSetTabSetting WHERE ParentId IN (" + joinedParentIds + ")";  
        List<SObject> records = queryAll(soql);  
        if (records.isEmpty()) return;

        List<String> tabDefinitionIds = records.stream()  
                                            .map(r -> (String)r.getField("Name"))  
                                            .distinct()  
                                            .collect(Collectors.toList());  
        Map<String, SObject> tabDefinitions = fetchDetailsByIds("TabDefinition", "Id, Label, SobjectName, Name", tabDefinitionIds);

        for (SObject record : records) {  
            ComprehensivePermissionSet cps = permSetMapById.get((String) record.getField("ParentId"));  
            if (cps != null) {  
                TabSettingDetail detail = new TabSettingDetail();  
                String tabDefApiName = (String) record.getField("Name"); // This is the API name of the TabDefinition  
                detail.setTabNameOrId(tabDefApiName);  
                detail.setVisibility((String) record.getField("Visibility"));

                SObject tabDef = tabDefinitions.values().stream() // TabDefinition.Name is API name, Id is Id.  
                                     .filter(td -> tabDefApiName.equals((String)td.getField("Name")) || tabDefApiName.equals((String)td.getId()))  
                                     .findFirst().orElse(null);

                if(tabDef != null){  
                    detail.setTabLabel((String) tabDef.getField("Label"));  
                } else {  
                    detail.setTabLabel(tabDefApiName); // Fallback  
                }  
                cps.getTabSettings().add(detail);

                // Basic Application Visibility from Tabs (for custom apps where tab name = app name)  
                // This is a simplification. True app visibility can be more complex.  
                if (tabDef != null && "Visible".equals(detail.getVisibility())) {  
                    String sObjectName = (String) tabDef.getField("SobjectName");  
                    if (sObjectName == null) { // It might be a custom app tab, VF tab, etc.  
                        // Attempt to infer application from tab label or name if it's a primary app tab  
                        // This part is heuristic and might need refinement based on how apps are named/structured.  
                        ApplicationVisibilityDetail appDetail = new ApplicationVisibilityDetail();  
                        appDetail.setApplicationId((String)tabDef.getId()); // Using TabDefinitionId as a proxy  
                        appDetail.setApplicationName((String)tabDef.getField("Name")); // API Name  
                        appDetail.setApplicationLabel((String)tabDef.getField("Label"));  
                        appDetail.setApplicationType("Inferred from Tab"); // Indicate heuristic nature  
                        // Check for duplicates before adding  
                        if(cps.getApplicationVisibilities().stream().noneMatch(a -> a.applicationId.equals(appDetail.applicationId))) {  
                           // cps.getApplicationVisibilities().add(appDetail); // Decided to keep ApplicationVisibility separate  
                        }  
                    }  
                }  
            }  
        }  
        // Note: Dedicated ApplicationVisibility often refers to AppMenuItem or specific app settings.  
        // Querying AppMenuItem and linking it to PermissionSet is not direct.  
        // Often controlled by Profile/PermissionSet having the 'View' permission on the App.  
        // The POJO's ApplicationVisibilityDetail might need a more specific querying strategy  
        // possibly via Metadata API or a more complex SOQL if a direct junction object exists.  
        // For now, I'll leave ApplicationVisibilities to be populated by a more dedicated method if needed.  
    }

    private void fetchServicePresenceStatusAccesses(Map<String, ComprehensivePermissionSet> permSetMapById, String joinedParentIds) throws ConnectionException {  
        if (permSetMapById.isEmpty()) return;  
        // Querying through the relationship to get fields from ServicePresenceStatus directly  
        String soql = "SELECT ParentId, ServicePresenceStatusId, ServicePresenceStatus.DeveloperName, ServicePresenceStatus.MasterLabel " +  
                      "FROM ServicePresenceStatusAccess WHERE ParentId IN (" + joinedParentIds + ")";  
        List<SObject> records = queryAll(soql);  
        for (SObject record : records) {  
            ComprehensivePermissionSet cps = permSetMapById.get((String) record.getField("ParentId"));  
            if (cps != null) {  
                ServicePresenceStatusAccessDetail detail = new ServicePresenceStatusAccessDetail();  
                detail.setStatusId((String) record.getField("ServicePresenceStatusId"));

                SObject relatedStatus = (SObject) record.getField("ServicePresenceStatus"); // Get the related SObject  
                if (relatedStatus != null) {  
                    detail.setStatusDeveloperName((String) relatedStatus.getField("DeveloperName"));  
                    detail.setStatusMasterLabel((String) relatedStatus.getField("MasterLabel"));  
                }  
                cps.getServicePresenceStatusAccesses().add(detail);  
            }  
        }  
    }

    public static void main(String[] args) {  
        // Replace with your actual credentials and auth URL  
        // Auth URL for login.salesforce.com: "https://login.salesforce.com/services/Soap/u/XX.0" (replace XX.0 with API version e.g., 58.0)  
        // Auth URL for test.salesforce.com: "https://test.salesforce.com/services/Soap/u/XX.0"  
        String username = "YOUR_SALESFORCE_USERNAME";  
        String passwordAndToken = "YOUR_SALESFORCE_PASSWORD" + "YOUR_SECURITY_TOKEN_IF_NEEDED";  
        String authUrl = "https://YOUR_DOMAIN.my.salesforce.com/services/Soap/u/59.0"; // Or login.salesforce.com

        // --- WARNING ---  
        // Storing credentials directly in code is highly discouraged for production systems.  
        // Use environment variables, configuration files with appropriate permissions,  
        // or ideally, OAuth 2.0 flows (JWT Bearer, Authorization Code, etc.).  
        // This is for demonstration purposes only.  
        // --- END WARNING ---

        if ("YOUR_SALESFORCE_USERNAME".equals(username) || authUrl.contains("YOUR_DOMAIN")) {  
            System.err.println("Please update Salesforce credentials and auth URL in the main method before running.");  
            return;  
        }

        try {  
            SalesforcePermissionSetExtractor extractor = new SalesforcePermissionSetExtractor(username, passwordAndToken, authUrl);  
            List<ComprehensivePermissionSet> allDetails = extractor.fetchAllPermissionSetDetails();

            System.out.println("n--- Extracted Permission Set Details ---");  
            for (ComprehensivePermissionSet cps : allDetails) {  
                System.out.println("n=======================================================");  
                System.out.println("Permission Set: " + cps.getName() + " (ID: " + cps.getId() + ", Label: " + cps.getLabel() + ")");  
                System.out.println("  Is Custom: " + cps.isCustom() + ", Type: " + cps.getType());  
                if (cps.getLicenseName() != null) System.out.println("  License: " + cps.getLicenseName());  
                if (cps.isOwnedByProfile() && cps.getProfileName() != null) System.out.println("  Owned by Profile: " + cps.getProfileName());

                System.out.println("  User Permissions (" + cps.getUserPermissions().size() + "):");  
                cps.getUserPermissions().stream().filter(UserPermissionDetail::isEnabled).limit(5) // Show only enabled, limit for brevity  
                    .forEach(up -> System.out.println("    - " + up.getUiLabel() + " (API: " + up.getApiName() + "): " + up.isEnabled()));  
                if (cps.getUserPermissions().stream().filter(UserPermissionDetail::isEnabled).count() > 5) System.out.println("    ... and more.");

                System.out.println("  Object Permissions (" + cps.getObjectPermissions().size() + "):");  
                cps.getObjectPermissions().stream().limit(3).forEach(op ->  
                    System.out.println("    - Object: " + op.getSobjectType() +  
                                       " (R:" + op.isCanRead() + " C:" + op.isCanCreate() + " E:" + op.isCanEdit() + " D:" + op.isCanDelete() +  
                                       " VA:" + op.isCanViewAllRecords() + " MA:" + op.isCanModifyAllRecords() + ")")  
                );  
                 if (cps.getObjectPermissions().size() > 3) System.out.println("    ... and more.");

                System.out.println("  Field Permissions (" + cps.getFieldPermissions().size() + "):");  
                cps.getFieldPermissions().stream().limit(3).forEach(fp ->  
                    System.out.println("    - Field: " + fp.getFieldApiName() + " (R:" + fp.isCanRead() + " E:" + fp.isCanEdit() + ")")  
                );  
                if (cps.getFieldPermissions().size() > 3) System.out.println("    ... and more.");

                System.out.println("  Apex Class Accesses (" + cps.getApexClassAccesses().size() + "):");  
                cps.getApexClassAccesses().stream().limit(3).forEach(ac ->  
                    System.out.println("    - Class: " + (ac.getNamespacePrefix() != null ? ac.getNamespacePrefix() + "__" : "") + ac.getApexClassName())  
                );  
                 if (cps.getApexClassAccesses().size() > 3) System.out.println("    ... and more.");

                System.out.println("  VisualForce Page Accesses (" + cps.getVfPageAccesses().size() + "):");  
                cps.getVfPageAccesses().stream().limit(3).forEach(vf ->  
                    System.out.println("    - Page: " + (vf.getNamespacePrefix() != null ? vf.getNamespacePrefix() + "__" : "") + vf.getVfPageName())  
                );  
                if (cps.getVfPageAccesses().size() > 3) System.out.println("    ... and more.");

                System.out.println("  Custom Permission Assignments ("+ cps.getCustomPermissionAssignments().size() +"):");  
                cps.getCustomPermissionAssignments().stream().limit(3).forEach(cp ->  
                    System.out.println("    - Custom Perm: " + cp.getCustomPermissionLabel() + " (" + cp.getCustomPermissionDeveloperName() + ")")  
                );  
                if (cps.getCustomPermissionAssignments().size() > 3) System.out.println("    ... and more.");

                System.out.println("  Tab Settings ("+ cps.getTabSettings().size() +"):");  
                 cps.getTabSettings().stream().limit(3).forEach(ts ->  
                    System.out.println("    - Tab: " + ts.getTabLabel()+ " (" + ts.getTabNameOrId() + "), Visibility: " + ts.getVisibility())  
                );  
                if (cps.getTabSettings().size() > 3) System.out.println("    ... and more.");

                System.out.println("  Connected App Accesses ("+ cps.getConnectedAppAccesses().size() +"):");  
                cps.getConnectedAppAccesses().stream().limit(3).forEach(ca ->  
                    System.out.println("    - Connected App: " + ca.getConnectedAppLabel() + " (" + ca.getConnectedAppName() + ")")  
                );  
                if (cps.getConnectedAppAccesses().size() > 3) System.out.println("    ... and more.");

                System.out.println("  Service Presence Status Accesses ("+ cps.getServicePresenceStatusAccesses().size() +"):");  
                cps.getServicePresenceStatusAccesses().stream().limit(3).forEach(sps ->  
                    System.out.println("    - Status: " + sps.getStatusMasterLabel() + " (" + sps.getStatusDeveloperName() + ")")  
                );  
                 if (cps.getServicePresenceStatusAccesses().size() > 3) System.out.println("    ... and more.");

                // ApplicationVisibilities are not fully populated in this example, add print if implemented  
                if (!cps.getApplicationVisibilities().isEmpty()){  
                    System.out.println("  Application Visibilities ("+ cps.getApplicationVisibilities().size() +"):");  
                    cps.getApplicationVisibilities().stream().limit(3).forEach(app ->  
                        System.out.println("    - App: " + app.getApplicationLabel() + " (Name: " + app.getApplicationName() + ", Type: " + app.getApplicationType() + ")")  
                    );  
                    if (cps.getApplicationVisibilities().size() > 3) System.out.println("    ... and more.");  
                }

                System.out.println("=======================================================");  
            }

        } catch (ConnectionException e) {  
            System.err.println("Salesforce Connection/Query Error: " + e.getMessage());  
            e.printStackTrace();  
        } catch (Exception e) {  
            System.err.println("An unexpected error occurred: " + e.getMessage());  
            e.printStackTrace();  
        }  
    }  
}
```
Key Implementation Details and Considerations:

* Batching Queries: For fetching related details (ObjectPermissions, FieldPermissions, etc.), if you have many PermissionSet IDs, the WHERE ParentId IN (...) clause can become too long or hit SOQL limits. You'll need to batch the permissionSetIds list (e.g., into chunks of 200\) and run the queries multiple times, aggregating the results. The provided code uses queryAll for simplicity which handles pagination but not explicit batching of ID sets for IN clauses.  
* SetupEntityAccess Two-Step Fetch:  
  * First, query SetupEntityAccess to get SetupEntityId and SetupEntityType for a given ParentId (PermissionSet ID).  
  * Then, based on SetupEntityType (e.g., 'ApexClass', 'ApexPage', 'CustomPermission', 'ConnectedApplication'), query the respective object (e.g., ApexClass, ApexPage) using the collected SetupEntityIds to get names and other details.  
  * The code implements this by collecting all SetupEntityIds by type and then making batched calls to WorkspaceDetailsByIds.  
* User Permission Labels: The DescribeSObjectResult for PermissionSet is used to dynamically get the UI labels for the boolean user permission fields (PermissionsViewAllData, etc.). This avoids hardcoding.  
* ApplicationVisibilityDetail: This is the most complex one.  
  * Direct SOQL for "Application Visibility" as a distinct related list on PermissionSet is not straightforward for all app types.  
  * Standard/Custom Apps (in App Launcher): Visibility is often controlled by the PermissionSet (or Profile) having the App's associated TabDefinition (if it's a tab-based app) set to "Visible" in PermissionSetTabSettings, and the app itself being accessible. The AppMenuItem object represents items in the App Launcher. Querying AppMenuItem and trying to link it back to specific permission sets can be indirect.  
  * Connected Apps: Covered by SetupEntityAccess where SetupEntityType \= 'ConnectedApplication'.  
  * The current code for WorkspaceTabSettings makes a *simplistic inference* for application visibility based on visible tabs. A more robust solution for ApplicationVisibilityDetail might involve querying AppMenuItem or using the Metadata API for certain definitions if SOQL is insufficient for your specific needs for "Application Type". Your POJO's applicationId hints at needing to resolve to AppMenuItem.Id or CustomApplication.Id. I've added a placeholder in WorkspaceTabSettings for how you might *begin* to infer this but acknowledge its limitations. A dedicated method for ApplicationVisibility using SetupEntityAccess for AppMenu or similar might be needed.  
* TabSettingDetail: PermissionSetTabSetting.Name gives the API name of the TabDefinition. You'll likely want to query TabDefinition using these names/IDs to get the user-friendly Label for the tab (added to the POJO and code).  
* Error Handling: The example includes basic try-catch blocks. Robust applications would need more granular error handling, logging, and retry mechanisms.  
* Null Checks: Salesforce fields can be null. The code attempts to check for nulls when retrieving fields using (Type) SObject.getField("FieldName"). WSC generally handles missing fields by returning null for the getField() call on the parent SObject.  
* API Version: Ensure your auth URL and WSC JAR are compatible with the API version you intend to use.  
* queryAll Method: The helper queryAll handles SOQL query pagination (queryMore) automatically.  
* Efficiency:  
  * Minimize SOQL queries. The current structure tries to batch entity detail lookups.  
  * Use Maps for quick lookups (e.g., permSetMapById, apexClassDetails).  
* POJO Getters/Setters: Make sure all your POJO fields have public getters and setters for proper data encapsulation and use by potential frameworks or libraries. (Added comments for this).

---

Phase 2: Data Structuring & Display

This phase is largely integrated within the WorkspaceAllPermissionSetDetails method and its sub-methods in the example above.

1. Main Loop: Iterate through the SObject records from the initial PermissionSet query.  
2. Instantiate ComprehensivePermissionSet: For each PermissionSet record, create a ComprehensivePermissionSet POJO.  
3. Populate Core Fields: Map fields directly from the SObject to the POJO. Handle relationship fields like License.Name and Profile.Name by accessing the nested SObject.  
4. Populate User Permissions: Iterate through the known boolean permission fields, create UserPermissionDetail objects, and add them to the list.  
5. Populate Detail Lists:  
   * After fetching related data (e.g., ObjectPermissions), iterate through those results.  
   * For each related record, find its parent ComprehensivePermissionSet POJO (using the permSetMapById).  
   * Create the specific detail POJO (e.g., ObjectPermissionDetail), populate it, and add it to the appropriate list within the parent POJO.  
6. Display (Example): The main method includes a basic iteration and print-to-console example for the populated ComprehensivePermissionSet objects.

---

This comprehensive Java code structure should give you a solid foundation. Remember to replace placeholder credentials and thoroughly test. The ApplicationVisibilityDetail might require the most adjustment depending on exactly which "applications" you are targeting and how their visibility is defined in your Salesforce org.

**Sources**  
1\. [https://github.com/trocco-io/embulk-output-sf\_bulk\_api](https://github.com/trocco-io/embulk-output-sf_bulk_api)  
2\. [https://github.com/Kashmir9d/salesforce-connector-oauth](https://github.com/Kashmir9d/salesforce-connector-oauth)  
3\. [https://github.com/data-integrations/salesforce](https://github.com/data-integrations/salesforce)  
4\. [https://stackoverflow.com/questions/12915153/selectbooleancheckbox-not-updated-after-filtered-data-table-update](https://stackoverflow.com/questions/12915153/selectbooleancheckbox-not-updated-after-filtered-data-table-update)  
5\. [https://github.com/songyz0310/flow](https://github.com/songyz0310/flow)  
6\. [https://github.com/InseeFrLab/onyxia-api](https://github.com/InseeFrLab/onyxia-api) soumis à licence (MIT)  
7\. [https://stackoverflow.com/questions/44020978/vaadin-spring-boot-jpa-point-grid-to-use-view-and-editor-to-use-table](https://stackoverflow.com/questions/44020978/vaadin-spring-boot-jpa-point-grid-to-use-view-and-editor-to-use-table)  
8\. [http://stackoverflow.com/questions/12369369/changing-from-struts-2-x-to-spring-3x-mvc](http://stackoverflow.com/questions/12369369/changing-from-struts-2-x-to-spring-3x-mvc)  
9\. [https://github.com/riteshkonduru/demoJSonXML](https://github.com/riteshkonduru/demoJSonXML)  
10\. [https://github.com/mineco-dev/k8s\_srv\_mineco](https://github.com/mineco-dev/k8s_srv_mineco)  
11\. [https://github.com/Savys/synthetic-data-generator](https://github.com/Savys/synthetic-data-generator)  
12\. [https://www.oschina.net/informat/file%E5%86%99%E6%96%87%E4%BB%B6](https://www.oschina.net/informat/file%E5%86%99%E6%96%87%E4%BB%B6)  
13\. [https://github.com/gentics/mesh](https://github.com/gentics/mesh)  
14\. [https://github.com/jreleaser/jreleaser](https://github.com/jreleaser/jreleaser) soumis à licence (Apache \- 2.0)  
15\. [https://github.com/tauasulamerica/chatsaude\_dev](https://github.com/tauasulamerica/chatsaude_dev) soumis à licence (Apache \- 2.0)  
16\. [https://github.com/AleksandrKrav/Library](https://github.com/AleksandrKrav/Library)  
17\. [https://github.com/MasakiKawaguchi/sf-management-tool](https://github.com/MasakiKawaguchi/sf-management-tool)  
18\. [https://github.com/forcedotcom/ApexUnit](https://github.com/forcedotcom/ApexUnit) soumis à licence (BSD \- 3 \- Clause)  
19\. [https://github.com/theonedev/onedev](https://github.com/theonedev/onedev) soumis à licence (MIT)  
20\. [https://github.com/1984shekhar/fuse-examples-6.2](https://github.com/1984shekhar/fuse-examples-6.2)  
21\. [https://github.com/lvjk/springcloud-study2](https://github.com/lvjk/springcloud-study2)  
22\. [http://stackoverflow.com/questions/22212158/mapping-nested-json-and-pojos-using-spring](http://stackoverflow.com/questions/22212158/mapping-nested-json-and-pojos-using-spring)  
23\. [https://github.com/15915763299/AlgorithmAndTools](https://github.com/15915763299/AlgorithmAndTools)  
24\. [https://stackoverflow.com/questions/24279245/how-to-handle-dynamic-json-in-retrofit/46320656](https://stackoverflow.com/questions/24279245/how-to-handle-dynamic-json-in-retrofit/46320656)  
25\. [https://stackoverflow.com/questions/27488412/persist-an-entity-after-another-entity-netbeans-j2e-bean-entitymanager](https://stackoverflow.com/questions/27488412/persist-an-entity-after-another-entity-netbeans-j2e-bean-entitymanager)  
26\. [https://github.com/forcedotcom/einstein-bot-channel-connector](https://github.com/forcedotcom/einstein-bot-channel-connector)  
27\. [https://stackoverflow.com/questions/29740879/how-to-replace-null-fields-nested-at-all-levels-from-json-response-using-jacks](https://stackoverflow.com/questions/29740879/how-to-replace-null-fields-nested-at-all-levels-from-json-response-using-jacks)  
28\. [http://salesforce.stackexchange.com/questions/22498/java-lang-noclassdeffounderror-com-sforce-soap-enterprise-connector](http://salesforce.stackexchange.com/questions/22498/java-lang-noclassdeffounderror-com-sforce-soap-enterprise-connector)  
29\. [http://www.howdycoder.com/](http://www.howdycoder.com/)  
30\. [https://salesforce.stackexchange.com/questions/287783/deleting-objectpermissions-record-fails-in-apex-batch-but-succeeds-in-two-separa](https://salesforce.stackexchange.com/questions/287783/deleting-objectpermissions-record-fails-in-apex-batch-but-succeeds-in-two-separa)  
31\. [https://github.com/udayvarma0007/jobs](https://github.com/udayvarma0007/jobs)
