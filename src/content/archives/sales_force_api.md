---
title: Salesforce REST API Guide
description: Salesforce REST API Guide
date: 2024-01-25 11:07
draft: false
---

> Oct. 2026 note: This is an old guide I wrote back in 2024. This might be out-of-date

## 1. Getting Started

Salesforce has a large [library of APIs](https://developer.salesforce.com/docs/apis), but here we are going to use their [REST API](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_rest.htm). The API access is enabled by default in Enterprise, Performance, Unlimited, and Developer Edition orgs. Professional Edition orgs can add API access as an add-on.

## 2. Getting Your User Access to Salesforce REST API

To access it, the user must have the API Enabled permission turned on in the user profile they’re assigned. This permission is enabled by default on some profiles, including many profiles available in Developer Edition orgs.

To turn on this setting:

1. Click the Gear icon and click Setup.
2. Search and go to “profiles” in the search bar 
3. Click Edit in the Profile you want to enable API access
4. Scroll down to Administrative Permissions and check the API Enabled box and click Save

## 3. Creating an App

Now that you have grant your user access to the API, the next step is to create a [Connected App](https://help.salesforce.com/s/articleView?id=sf.connected_app_create.htm&type=5).

To do so:

1. Open Setup home
2. Go to home -> Apps -> Manage apps -> new connected app
3. In the App Manager, set the Name, API Name and contact Email
4. Enable OAuth Settings, and set the callback URL to `sfdc://oauth/restapi/success`
5. Select the scope : `Access and manage your data (api)`
6. Save the connected app and wait few minutes for it to be activated.
7. Save the consumer key and consumer secret

## 4. Connect to the REST API

Salesforce uses [OAuth 2.0 Username-Password Flow](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_oauth_and_connected_apps.htm). This means that you must get an Access Token, that will be used to access all other available endpoints. To get this token, you have to make a `POST` request to the following endpoint:

```json
api_base_url + "/services/oauth2/token"
```

- Where `api_base_url` is the URL used to access the Salesforce instance through the browser

With the following Body:

```json
{
	  "grant_type": "password",
	  "client_id": {client_id},
	  "client_secret": {client_secret},
	  "username": {username},
	  "password": {password} + {security_token}
}
```

- Where Username and password are the credentials used to access the Salesforce instance through the browser
- Note that in `password`, you have to concatenate the  Account’s `password` with the `security_token`

Salesforce returns the response that contains the `access token` and `instance_url` :

```json
{
		"access_token": "{token}",
		"instance_url": "{instance_url}",
		"id": "{ID}",
		"token_type": "Bearer",
		"issued_at": "{date_stamp}",
		"signature": "{signature}"	
}
```

## 5. Using the REST API

### 5.1. Authentication

To call any endpoint, you have to use the `access_token`  we got from the last step. To do so, you have to add to your headers the following line:

```json
"Authorization": "Bearer {access_token}"
```

### 5.2. Getting The Schema of an Object

To get information from an object, you can use `GET` requests. To get the schema of an Salesforce Object, you can use the endpoint:

```
{instance_url}/services/data/v58.0/sobjects/{Object}/describe
```

it will return the information of all fields in this object, as its name in the web UI, in the API, its type, etc.

### 5.3. Getting an instance of an object

To get access to a object, you need to have their id. Once you have it, you can call the following end-point with a `GET` request to get all of their information:

```
{instance_url}/services/data/v58.0/sobjects/{Object}/{id}
```

### 5.4. Creating an Object instance

To create an object instance, you have to make a `POST` request, with the data you want filled in as the body of the request. Note that some objects have mandatory fields, that if they are not automatically created, you MUST have in your request body, otherwise it will return an error.

The end-point should be:

```json
{instance_url}/services/data/v58.0/sobjects/{Object}/
```

The body of the request should look like this:

 

```json
{
	"{Object_field_name_1}" "{value}",
	"{Object_field_name_2}" "{value}",
	"{Object_field_name_3}" "{value}"
}
```

If successful, the request will return the `id` of the newly created object:

```json
{
	"id": "{id}",
	"success": true,
	"errors": []
}
```

### 5.5. Updating Object Data

To create an object instance, you have to make a `PATCH` request, with the data you want updated  as the body of the request. 

If you try to update a field that does not exists, the request will return an error.

The endpoint to do this is

```json
{instance_url}/services/data/v58.0/sobjects/{Object}/{id}
```

## 6. [SOQL](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql.htm)

### 6.1. What is SOQL

SOQL is a query language designed for querying data within Salesforce. It's similar to SQL (Structured Query Language) but is specifically made for Salesforce data, the biggest difference being that while SQL works with tables, SOQL works with Objects (like MongoDB). 

### 6.2. Getting Access

We can access and make requests via the REST API, making a `GET` request to the endpoint

```json
{instance_url}/services/data/v58.0/query/
```

with the query as a param:

```json
?q={query}
```

The full URL of the request should look like this:

```json
{instance_url}/services/data/v58.0/query/?q={query}
```

### 6.3. SOQL Cheat Sheet

SOQL is very similar to SQL to have a quick start, bellow is a cheat sheet with the main operations

| Operation | Syntax Example |
| --- | --- |
| Selecting Fields | SELECT field1, field2 FROM ObjectName |
| Filtering Records | SELECT Name FROM Account WHERE Industry = 'Technology' |
| Sorting Results | SELECT Name FROM Account ORDER BY CreatedDate DESC |
| Limiting Results | SELECT Name FROM Account LIMIT 10 |
| Relationship Queries | SELECT Name, (SELECT LastName FROM Contacts) FROM Account |
| Aggregate Functions | SELECT COUNT(Id) FROM Account |
| Date Conditions | SELECT Name FROM Account WHERE CreatedDate > YESTERDAY |
| Logical Operators | SELECT Name FROM Account WHERE Industry = 'Tech' AND AnnualRevenue > 1000000 |
| Group By and Having | SELECT Industry, COUNT(Id) FROM Account GROUP BY Industry HAVING COUNT(Id) > 5 |
| Subqueries | SELECT Name, (SELECT LastName FROM Contacts) FROM Account |
| Relationship Queries on Custom Objects | SELECT Name, (SELECT CustomField__c FROM CustomObjectChildren__r) FROM CustomObjectParent__c |

For a more in-depth cheat sheet please [see this one by salesforceben.com](https://www.salesforceben.com/soql-cheat-sheet/)

### 6.4. Request Responses

If successful, the request response will be a JSON with the data you queried as the result. For examplie, this query:

```sql
SELECT id, Name FROM Account LIMIT 2
```

Will have a response of

```json
{
	{
		"metadata": {...},
		"id": "{id_1}",
		"name": "{Name_1}"
	},
	{
		"metadata": {...},
		"id": "{id_2}",
		"name": "{Name_2}"
	}
}
```

### 6.5. Limitations

SOQL cannot create or update data, as it is meant to be used only for getting it. To update or create new data, you have to use the previously described methods.
