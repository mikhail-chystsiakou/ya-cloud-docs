---
editable: false
---

# Method listAccessBindings

 

 
## HTTP request {#https-request}
```
GET undefined/functions/v1/functions/{resourceId}:listAccessBindings
```
 
## Path parameters {#path_params}
 
Parameter | Description
--- | ---
resourceId | Required. ID of the resource to list access bindings for.  To get the resource ID, use a corresponding List request. For example, use the [list](/docs/resource-manager/api-ref/Cloud/list) request to get the Cloud resource ID.
 
## Query parameters {#query_params}
 
Parameter | Description
--- | ---
pageSize | The maximum number of results per page that should be returned. If the number of available results is larger than pageSize, the service returns a nextPageToken that can be used to get the next page of results in subsequent list requests. Default value: 100.  The maximum value is 1000.
pageToken | Page token. Set pageToken to the nextPageToken returned by a previous list request to get the next page of results.  The maximum string length in characters is 100.
 
## Response {#responses}
**HTTP Code: 200 - OK**

```json 
{
  "accessBindings": [
    {
      "roleId": "string",
      "subject": {
        "id": "string",
        "type": "string"
      }
    }
  ],
  "nextPageToken": "string"
}
```

 
Field | Description
--- | ---
accessBindings[] | **object**<br><p>List of access bindings for the specified resource.</p> 
accessBindings[].<br>roleId | **string**<br><p>ID of the <a href="/docs/iam/api-ref/Role#representation">Role</a> that is assigned to the subject.</p> <p>The maximum string length in characters is 50.</p> 
accessBindings[].<br>subject | **object**<br><p>Required. Identity for which access binding is being created. It can represent an account with a unique ID or several accounts with a system identifier.</p> 
accessBindings[].<br>subject.<br>id | **string**<br><p>ID of the subject.</p> <p>It can contain one of the following values:</p> <ul> <li> <p><code>allAuthenticatedUsers</code>: A special system identifier that represents anyone who is authenticated. It can be used only if the type is <code>system</code>.</p> </li> <li> <p><code>&lt;cloud generated id&gt;</code>: An identifier that represents a user account. It can be used only if the type is <code>userAccount</code> or <code>serviceAccount</code>.</p> </li> </ul> <p>The maximum string length in characters is 50.</p> 
accessBindings[].<br>subject.<br>type | **string**<br><p>Type of the subject.</p> <p>It can contain one of the following values:</p> <ul> <li><code>system</code>: System group. This type represents several accounts with a common system identifier.</li> <li><code>userAccount</code>: An user account (for example, &quot;alice.the.girl@yandex.ru&quot;). This type represents the <a href="/docs/iam/api-ref/UserAccount#representation">UserAccount</a> resource.</li> <li><code>serviceAccount</code>: A service account. This type represents the <a href="/docs/iam/api-ref/ServiceAccount#representation">ServiceAccount</a> resource.</li> </ul> <p>For more information, see <a href="/docs/iam/concepts/access-control/#subject">Subject to which the role is assigned</a>.</p> 
nextPageToken | **string**<br><p>This token allows you to get the next page of results for list requests. If the number of results is larger than pageSize, use the nextPageToken as the value for the pageToken query parameter in the next list request. Each subsequent list request will have its own nextPageToken to continue paging through the results.</p> 