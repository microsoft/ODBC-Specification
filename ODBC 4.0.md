ODBC 4.0 Specification
=

Table of Contents
<!-- TOC depthFrom:1 depthTo:3 withLinks:1 updateOnSave:1 orderedList:0 -->

- [1 Overview](#1-overview)
- [2 Compatibility with ODBC 3.x](#2-compatibility-with-odbc-3x)
   - [2.1 ODBC 3.x Errata](#21-odbc-3x-errata)
- [3 Design](#3-design)
   - [3.1 Private Drivers](#31-private-drivers)
   - [3.2 Web Authorization](#32-web-authorization)
   - [3.3 SQL Syntax Support](#33-sql-syntax-support)
   - [3.4 String Format](#34-string-format)
   - [3.5 Schema Inference](#35-schema-inference)
   - [3.6 Getting Data during Fetch](#36-getting-data-during-fetch)
   - [3.7 Variable Typed Columns](#37-variable-typed-columns)
   - [3.8 Variable Length Columns](#38-variable-length-columns)
   - [3.9 Dynamic Columns](#39-dynamic-columns)
   - [3.10 Structured Columns](#310-structured-columns)
   - [3.11 Collection-valued Columns](#311-collection-valued-columns)
   - [3.12 New SQL C types](#312-new-sql-c-types)
- [4 Change Tracking](#4-change-tracking)
- [5 Compatibility](#5-compatibility)
	- [5.1 ODBC 3.x Clients](#51-odbc-3x-clients)
	- [5.2 ODBC 3.x Drivers](#52-odbc-3x-drivers)
	- [5.3 ODBC 4.0 Drivers](#53-odbc-40-drivers)
	- [5.4 ODBC 4.0 Clients](#54-odbc-40-clients)
	- [5.5 ODBC 4.0 Driver Manager](#55-odbc-40-driver-manager)
- [6 New Functions](#6-new-functions)
	- [6.1 SQLNextColumn](#61-sqlnextcolumn)
	- [6.2 SQLGetNestedHandle](#62-sqlgetnestedhandle)
	- [6.3 SQLStructuredTypes](#63-sqlstructuredtypes)
	- [6.4 SQLStructuredTypeColumns](#64-sqlstructuredtypecolumns)

# 1 Overview

ODBC is a widely adopted, standard, client-side API that works well for relational (i.e., tabular) data. However, many types of today's data may not fit well into a relational model. ODBC 4.0 defines extensions to support non-relational concepts such as:

1.  Dynamic columns (columns not defined in schema)
2.  Structured columns
3.  Collection-valued columns
4.  Untyped/Varying typed columns

In addition, general enhancements are added in order to improve interoperability of generic ODBC clients across diverse data sources, including enhanced discovery, authentication, syntax, and capability reporting.

# 2 Compatibility with ODBC 3.x

ODBC 3.x is a relational API. Existing clients expect data to be modeled, queried, represented and updated relationally.

In order to be advertised as an ODBC 3.x driver, and returned from the control panel/driver manager, drivers must be 100% compatible with ODBC 3.x. Such drivers must expose a relational view of the data through the standard schema functions (SQLTables, SQLColumns, etc.) and continue to support standard Entry SQL-92 operations against this flattened view.

The flattened view may not be comprehensive (there may be data not available through the relational view, data modeled as strings representing structured content of a particular format, the data may not be updatable through that view, etc.) but existing ODBC 3.x clients must be able to meaningfully interact with the data.

The convenience to existing clients of representing hierarchical data through a relational lens sacrifices fidelity. Applications that specify a value for the `SQL_ATTR_ODBC_VERSION` environment attribute of 4.x or greater may see a hierarchical structure containing row, collection, and variant-typed data values.

For more information on compatibility, see [Section 5, Compatibility](#5-compatibility)

# 2.1 ODBC 3.x Errata
The following errata are not intended to describe new functionality but to clarify existing expected ODBC behavior.

## 2.1.1 Types of Data Supported by SQLGetData
While the current ODBC Specification calls out the use of *TargetType* for specifying the type of data to be retrieved in SQLGetData, which may differ from the type in the APD or ARD, it is not clear that, when retrieving string or binary data in parts, the same C data type should be specified for each part. 

## 2.1.1 SQLGetData and Variable Length Data
The current ODBC Specification states the following:

> When the driver returns fixed-length data, such as an integer or a date structure, the driver ignores BufferLength and assumes the buffer is large enough to hold the data. It is therefore important for the application to allocate a large enough buffer for fixed-length data or the driver will write past the end of the buffer.

In this context, "fixed-length data" refers to "non-character or binary data". As described in "Retrieving Variable-Length Data In Parts", SQLGetData can be used to retrieve any character or binary data in pieces, and drivers should always respect BufferLength when writing string or binary data.

## 2.1.2 Use of Substitution Character
Drivers should use a substitution character (HEX 1A), rather than raise an exception, if a character is not available in the target codepage. Clients should be aware of this behavior, for example, when attempting to round-trip values.

# 3 Design

ODBC 4.0 is extended to support common features across all types of data sources, as well as specific new features to support new schema concepts.

# 3.1 Private Drivers

Drivers registered through the ODBC setup utility are available to all applications. All applications using the driver use the same version of the driver.

ODBC 4.0 enables applications to use their own private drivers. Such drivers are not visible to other applications and may support different versions from applications using the same driver.

Private drivers are not registered using the ODBC setup utility. The new `SQL_ATTR_PRIVATE_DRIVER_PATH` environment attribute specifies a path which will be implicitly checked for private drivers. By default, this path is set by appending "ODBC Drivers" to the path provided by [GetModuleFileName](https://msdn.microsoft.com/en-us/library/windows/desktop/ms683197) with a NULL hModule. Each private driver is represented in that directory by an .ini file whose name is the *driver-name* with the “.ini” suffix.

The .ini file follows the [windows .ini file format](https://en.wikipedia.org/wiki/INI_file). It is a text file whose first line is a section header containing the name of the driver, enclosed in square brackets, and followed by keyword=value pairs, one per line. Valid keyword-value pairs are the [subkeys](https://msdn.microsoft.com/en-us/library/ms709431(v=vs.85).aspx) allowed for the driver specific section under ODBCINST.INI, along with a new “ApplicationKey” key-value pair which, if specified, is passed by the driver manager to the driver in a call to SetEnvAttr with the new `SQL_ATTR_APPLICATION_KEY` attribute. The driver developer can use this to validate that the calling application is permitted to use the driver, for example, by using a signed hash of the application’s publisher validated against the calling application at runtime.

SQLDrivers enumerates the drivers registered under the ODBCINST.INI registry key as well as the drivers bin-deployed within the current application’s install directory as described above. The *DriverAttributes* returned by SQLDrivers for a bin-deployed driver is the set of key-value pairs read from the corresponding .ini file.

This expanded set of drivers is also available in the data source configuration dialog for creating new DSNs.

SQLDataSources filters the set of DSNs based on the underlying driver, enumerating only those data sources whose driver is available to the application. This also restricts the list of available data sources that are enumerated in the data source connection dialog.

When resolving a driver for a registered DSN, the Driver Manager first looks to the Driver keyword within the named DSN that is a direct child of the ODBC.INI subkey. If this string matches the name of a directory under the current “ODBC Drivers” directory, then the driver is loaded according to the .ini file defined in that directory. Otherwise, the Driver Manager looks for a registry entry under the “Drivers” section within ODBCINST.INI. Failing that, the Driver Manager treats the string as a file path and attempts to load the driver using that path.

This allows bin-deployed drivers to work like any other driver from the application’s perspective, including the creation of DSNs, and allows DSNs to be shared across applications that each have their own copy of the driver.

## 3.2 Web Authorization

Applications can connect to drivers that require web-based authentication through SQLDriverConnect or SQLBrowseConnect.

### 3.2.1 Web Authorization Flow with SQLDriverConnect

Applications can connect to services requiring web authentication by calling SQLDriverConnect.

If the input connection string contains the required information (for example, access token, scope, refresh token, and expiration, as appropriate) in addition to any driver-specific properties, and the access token has not expired, no additional information should be required. If a provided access token has expired and the connection string contains a refresh token, the driver attempts to refresh the connection using the supplied refresh token. If the access token has expired and the connection string does not contain a refresh token, SQLDriverConnect fails as described in [Token Expiration](#324-token-expiration).

Applications that don’t have a complete connection string, but are willing to allow drivers to pop up dialogs to obtain necessary information from the user, can call SQLDriverConnect, specifying a DriverCompletion value of `SQL_DRIVER_PROMPT`, `SQL_DRIVER_COMPLETE`, or `SQL_DRIVER_COMPLETE_REQUIRED`.

When SQLDriverConnect is called with a DriverCompletion value other than `SQL_DRIVER_NOPROMPT`, drivers that require web authentication not present in the input connection string conduct the necessary authentication flow, including presenting browser windows as necessary, and return to the application a completed connection string containing driver-specific properties, an access token, scope (as appropriate) and, if the access token has an expiration, the expiration (as an ISO 8601 datetime with timezone) and refresh token:

    "Auth_AccessToken=xxx;Auth_Expires=xxx;Auth_RefreshToken=xxx;Auth_Scope=ZZZ;Prop1=xxx;"

This connection string contains all the information necessary to be used in a subsequent call to SQLDriverConnect or SQLBrowseConnect to reconnect to the service. Note that this connection string may contain sensitive information and should be protected by the application.

### 3.2.2 Web-based Authentication Flow with SQLBrowseConnect

Applications that are unable to allow drivers to pop up dialogs can call SQLBrowseConnect to connect to the service.

SQLBrowseConnect provides an iterative dialog between the driver and the application where the application passes in an initial input connection string. If the connection string contains sufficient information to connect, the driver responds with `SQL_SUCCESS` and an output connection string containing the complete set of connection attributes used.

If the initial input connection string does not contain sufficient information to connect to the source, the driver responds with `SQL_NEED_DATA` and an output connection string specifying informational attributes for the application (such as the authorization url) as well as required attributes to be specified in a subsequent call to SQLBrowseConnect. Attribute names returned by SQLBrowseConnect may include a colon followed by a localized identifier, and the value of the requested attribute is either a single question mark or a comma-separated list of valid values (optionally including localized identifiers) enclosed in curly braces. Optional attributes are returned preceded with an asterix (`*`).

In a Web-based authentication scenario, if SQLBrowseConnect is called with an input connection string containing an access token that has not expired, along with any other required properties, no additional information should be required. If the access token has expired and the connection string contains a refresh token, the driver attempts to refresh the connection using the supplied refresh token.

If SQLBrowseConnect is called without an access token, with an expired access token and the driver is unable to refresh the token, or with other required information missing, SQLBrowseConnect returns with `SQL_NEED_DATA`.

In cases where different authentication methods may be supported, the driver can initially return an OutConnectionString allowing the application to select from an enumerated set of authentication methods.

    "Auth_Type={OAuth_1.0, OAuth_2.0}"

Common keywords for `Auth_Type` include “`Basic`”, “`Integrated`”, “`OAuth_1.0`”, “`OAuth_2.0`”, and “`None`”. Drivers should use these keywords for the corresponding authentication types, where supported, and may specify other driver-specific authentication types.

In order to specify an authentication method, the application calls SQLBrowseConnect with a connection string specifying one of those allowed authentication methods, and SQLBrowseConnect returns an output connection string requesting the connection attributes required for that particular choice.

For OAuth 2.0, SQLBrowseConnect returns an OutConnectionString requesting a redirect uri, along with a client ID, a client secret, and a scope, as required, along with any required provider-specific properties.

    "Auth_BaseRedirectUri:Redirect Uri=?;Auth_Client_ID:ClientID=?;Auth_Client_Secret:Client Secret=?;*Auth_Scope:Scope={XXX:xxx,YYY:yyy,ZZZ:zzz}"

If the service supports scopes but the driver is unable to enumerate those scopes, it simply returns a question mark (?) as the value for `Auth_Scope` in the output connection string:

    "Auth_BaseRedirectUri:Redirect Uri=?;Auth_Client_ID:ClientID=?;Auth_Client_Secret:Client Secret=?;*Auth_Scope:Scope=?"

For an OAuth 1.0, the output connection string would also ask for `Auth_Realm` and `Auth_Token_Secret`.

    "Auth_BaseRedirectUri:Redirect Uri=?;Auth_Client_ID:ClientID=?;Auth_Client_Secret:Client Secret=?;*Auth_Scope:Scope=?;Auth_Realm:Realm=?;Auth_Token_Secret:Token Secret=?"

The application next calls SQLBrowseConnect with a connection string containing the redirect uri, the client ID, and client secret, along with scope, realm, and token secret, as appropriate:

    "Auth_BaseRedirectUri=https://abc.com/auth;Auth_Client_ID=myapp;Auth_Client_Secret=xyz;Auth_Scope=ZZZ"

The driver responds with a connection string containing the authorization url and requesting the final redirect uri. For the best client UI experience, the output connection string should also include suggested window height and width (in pixels) for the browser window:

    “AuthorizationUrl=xxx;Auth_WindowHeight=xxx;Auth_WindowWidth=xxx;Auth_CompletedRedirectUri=?”

The application navigates the browser to the Url specified by the authorization url. Auth flow is complete once the browser reaches the redirect uri.

At that point, the application calls SQLBrowseConnect again, supplying the completed redirect uri (including any query string parameters). The driver uses the query string parameters returned in this uri to complete any additional flow required and returns `SQL_SUCCESS` with an OutConnectionString containing driver-specific properties, an access token, and scope, expiration (as an ISO 8601 datetime with timezone), and refresh token as appropriate:

    "Auth_AccessToken=xxx;Auth_Expires=xxx;Auth_RefreshToken=xxx;Auth_Scope=ZZZ;Prop1=xxx; "

This connection string contains all the information necessary to be used in a subsequent call to SQLDriverConnect or SQLBrowseConnect to reconnect to the service. Note that this connection string may contain sensitive information and should be protected by the application.

### 3.2.3 New Connection Keywords

ODBC 4.0 defines the following new Connection string key/value pairs:

#### 3.2.3.1 `Auth_Type`

`Auth_Type` defines common types of authentication. Drivers that support multiple types of authentication can request `Auth_Type` in the output connection string returned by BrowseConnect in order to allow the client to select from an enumerated list of authentication mechanisms.

While the driver can return custom values, the following common values should be used where applicable:

 **Keyword** | **Description** 
-------------|---------------
 `None`        | No authentication is required; the driver supports anonymous access
 `Basic`       | Basic authentication using User ID (UID) and Password (PWD) is supported by the driver
 `Trusted`     | The driver supports connecting using trusted (i.e., Kerberos or NTLM) Authentication.
 `OAuth_1.0`  | The driver supports an OAuth 1.0 authentication flow, as described in [Web Authorization](#32-web-authorization).
 `OAuth_2.0`  | The driver supports an OAuth 2.0 authentication flow, as described in [Web Authorization](#32-web-authorization)
 `LDAP`        | The driver supports authentication through LDAP 

#### 3.2.3.2 `Auth_BaseRedirectUri`

In a [Web Authorization Flow](#32-web-authorization), the driver may request the application to provide a Uri to be called when the authentication completes. This is typically a URI to an endpoint supported by the application. The driver will construct an [AuthorizationUrl](#3233-authorizationurl) using this callback URL.

#### 3.2.3.3 `AuthorizationUrl`

In a [Web Authorization Flow](#32-web-authorization), the driver may provide an AuthorizationUrl that the application must invoke in order to initiate the authorization.

#### 3.2.3.4 `Auth_CompletedRedirectUri` 

In a [Web Authorization Flow](#32-web-authorization), the driver may ask for a CompletedRedirectUri that is the result of invoking the AuthorizationUrl. This CompletedRedirectUri will generally contain information, i.e. through query string parameters, used by the driver to complete the authorization flow.

#### 3.2.3.5 `Auth_Client_ID`, `Auth_Client_Secret`

In a [Web Authorization Flow](#32-web-authorization), the driver may request a Client ID and Client Secret from the application. This information is typically used to construct the [AuthorizationUrl](#3233-authorizationurl) used to initiate authorization.

#### 3.2.3.6 `Auth_Scope`

In a [Web Authorization Flow](#32-web-authorization), the driver may request a Scope from the application. The driver may provide an enumerated list of known scopes that the application can pick from, or may allow the application to provide an arbitrary scope. This information is typically used to construct the [AuthorizationUrl](#3233-authorizationurl) used to initiate authorization.

#### 3.2.3.7 `Auth_Realm`

In a [OAuth 1.0 Authorization Flow](#32-web-authorization), the driver may request a Realm from the application. This information is typically used to construct the [AuthorizationUrl](#3233-authorizationurl) used to initiate authorization.

#### 3.2.3.8 `Auth_Token_Secret`.

In a [OAuth 1.0 Authorization Flow](#32-web-authorization), the driver may request a Secret from the application. This information is typically used to construct the [AuthorizationUrl](#3233-authorizationurl) used to initiate authorization.

#### 3.2.3.9 `Auth_WindowHeight`, `Auth_WindowWidth`

In a [Web Authorization Flow](#32-web-authorization), the application may have to provide a window (i.e., through a browser) to collect input from the user. The driver may provide WindowHeight and WindowWidth keywords in the output query string, for example along with the [AuthorizationUrl](#3233-authorizationurl), as hints to the application as to the size of the window to present to the user.

### 3.2.4 Token Expiration

Applications can determine whether or not the access token has expired by calling SQLGetConnectAttr with the `SQL_ATTR_CONNECTION_DEAD` attribute. As this connection attribute is also used by the Driver Manager in connection pooling, getting this attribute is performance critical and the driver SHOULD NOT make a round trip to determine the status of the connection, but should return `SQL_CD_TRUE` if it has previously determined that the connection is dead.

If [SQL_ATTR_REFRESH_CONNECTION](#3252-sql-attr-refresh-connection) is `SQL_REFRESH_MANUAL`, or the driver is unable to refresh the token, attempting to call an operation that requires a connection after the access token has expired returns `SQL_ERROR` with a SQLState value of 08006 (Connection Failure). Applications can attempt to refresh the connection by calling SQLSetConnectAttr with `SQL_ATTR_REFRESH_CONNECTION` and, if successful, retry the failed operation. If refreshing the connection fails, the application must close the connection (resulting in any dependent statement or descriptor handles being implicitly freed) and start over.

### 3.2.5 Token Refresh

Access tokens can be refreshed automatically by the driver, or manually by the application.

#### 3.2.5.1 `SQL_ATTR_CREDENTIALS`

Applications can retrieve the current connection information, including (for web authorization scenarios) the current access token, along with refresh token and expiration, as appropriate, by calling SQLGetConnectAttr for the new `SQL_ATTR_CREDENTIALS` connection attribute.

Applications can update credentials by calling SQLSetConnectAttr for `SQL_ATTR_CREDENTIALS` and specifying the value retrieved in a previous call to GetConnectionAttr for `SQL_ATTR_CREDENTIALS` to the same data source.

This information returned by `SQL_ATTR_CREDENTIALS` is an encoding of the necessary connection information sufficient to create a new connection to the same source by passing to SQLDriverConnect or SQLBrowseConnect, or enable refreshing credentials for the same source by calling SQLSetConnectAttr with the new `SQL_ATTR_CREDENTIALS` connection attribute.

Note that, while drivers should encode sensitive information within the string, applications should treat the credentials as sensitive information.

#### 3.2.5.2 `SQL_ATTR_REFRESH_CONNECTION`

A new connection attribute, `SQL_ATTR_REFRESH_CONNECTION`, is added to allow applications to control how and when drivers refresh a connection.

If `SQL_ATTR_REFRESH_CONNECTION` is `SQL_REFRESH_AUTO` (the default), the driver attempts to automatically refresh connections (i.e., access token) as required, using the current value of `SQL_ATTR_CREDENTIALS`. Upon successful refresh, the value of `SQL_ATTR_CREDENTIALS` is updated with the new credentials (i.e., access token, and refresh token and expiration, as appropriate.)

If `SQL_ATTR_CONNECTION` is set to `SQL_REFRESH_NOW`, the driver attempts to refresh the connection (i.e., access token) using the current value of `SQL_ATTR_CREDENTIALS`. Upon successful refresh, the value of `SQL_ATTR_CREDENTIALS` is updated with the new credentials (i.e., access token, and refresh token and expiration, as appropriate.) Setting `SQL_ATTR_CONNECTION` to `SQL_REFRESH_NOW` does an immediate refresh of the connection but does not change the persisted value of connection option. Drivers may, but are not required to, return `SQL_SUCCESS_WITH_INFO` and `01S02`, Optional Value Changed, since the value of `SQL_ATTR_REFRESH_CONNECTION` is not actually changed to the specified value.

If `SQL_ATTR_REFRESH_CONNECTION` is set to `SQL_REFRESH_MANUAL`, the driver should not attempt to automatically refresh access tokens.

## 3.3 SQL Syntax Support

ODBC is designed as an API to provide a common way to connect to, describe, execute a command, and retrieve results from a relational store. ODBC relies on ANSI/ISO SQL to provide a common grammar across relational stores, and supplies a way to report levels of conformance and support for optional syntax elements.

### 3.3.1 SQL Compliance

Drivers that support ANSI SQL syntax must follow SQL semantics for that syntax or fail the request.

### 3.3.2 String escaping

As per ANSI, a single-quote character within a string literal is escaped by a single-quote character. Drivers must translate double-single quotes to whatever is the native syntax for an escaped single quote.

### 3.3.3 Nesting Escape Clauses

Drivers should support nesting of escape clauses, as appropriate (for example, using an escape clause within an expression passed to a scalar function).

### 3.3.4 SQL Syntax Capabilities

ODBC Drivers report supported SQL syntax through SQLGetInfo. The following SQLGetInfo information types and values are added to help drivers describe support for additional SQL constructs

#### 3.3.4.1 Binary Functions

ODBC 4.0 drivers may support the [canonical string functions](https://msdn.microsoft.com/en-us/library/ms710249(v=vs.85).aspx) against binary strings.

##### 3.3.4.1.1 `SQL_BINARY_FUNCTIONS`

A new SQLGetInfo *information type,* `SQL_BINARY_FUNCTIONS`, is added to allow a server to report which of the string functions can be used with binary strings.

The following new bitmask values may be returned from `SQL_BINARY_FUNCTIONS`:

`SQL_FN_BIN_BIT_LENGTH`, `SQL_FN_BIN_CONCAT`, `SQL_FN_BIN_INSERT`, `SQL_FN_BIN_LTRIM`, `SQL_FN_BIN_OCTET_LENGTH`, `SQL_FN_BIN_POSITION`, `SQL_FN_BIN_RTRIM`, `SQL_FN_BIN_SUBSTRING`

Note: Each of these is defined with the same value as their existing `SQL_FN_STR` counterpart.

##### 3.3.4.1.2 `SQL_ISO_BINARY_FUNCTIONS`

A new GetInfo *information type*, `SQL_ISO_BINARY_FUNCTIONS`, is added to allow a server to report which of the ANSI SQL Binary functions can be used.

The following bitmasks are used to determine which string functions are supported. Their values are the same as their `SQL_SFF` counterparts for `SQL_SQLSTRING_FUNCTIONS`.

`SQL_SBF_CONVERT`, `SQL_SBF_SUBSTRING`, `SQL_SBF_TRIM_BOTH`, `SQL_SBF_TRIM_LEADING`, `SQL_SBF_TRIM_TRAILING`, `SQL_SBF_OVERLAY`, `SQL_SBF_LENGTH`, `SQL_SBF_POSITION`, `SQL_SBF_CONCAT`

#### 3.3.4.2 `SQL_ISO_STRING_FUNCTIONS`

A new SQLGetInfo *information type*, `SQL_ISO_STRING_FUNCTIONS`, is added to allow a server to report which of the ANSI SQL String functions can be used. Its value is equal to the existing `SQL_SQL92_STRING_FUNCTIONS`. Values are the existing `SQL_SSF` values from `SQL_SQL92_STRING_FUNCTIONS`, plus the following values:

`SQL_SSF_OVERLAY`, `SQL_SSF_LENGTH`, `SQL_SSF_POSITION`, `SQL_SSF_CONCAT`

#### 3.3.4.3 `SQL_SQL92` Equivalents

The following SQLGetInfo *information type*s are defined equivalent to existing `SQL_SQL92` defines, but not specific to SQL92:

    SQL_ISO_DATETIME_FUNCTIONS = SQL_SQL92_DATETIME_FUNCTIONS
    SQL_ISO_FOREIGN_KEY_DELETE_RULE = SQL_SQL92_FOREIGN_KEY_DELETE_RULE
    SQL_ISO_FOREIGN_KEY_UPDATE_RULE = SQL_SQL92_FOREIGN_KEY_UPDATE_RULE
    SQL_ISO_GRANT = SQL_SQL92_GRANT
    SQL_ISO_NUMERIC_VALUE_FUNCTIONS = SQL_SQL92_NUMERIC_VALUE_FUNCTIONS
    SQL_ISO_PREDICATES = SQL_SQL92_PREDICATES
    SQL_ISO_RELATIONAL_JOIN_OPERATORS = SQL_SQL92_RELATIONAL_JOIN_OPERATORS
    SQL_ISO_REVOKE = SQL_SQL92_REVOKE
    SQL_ISO_ROW_VALUE_CONSTRUCTOR = SQL_SQL92_ROW_VALUE_CONSTRUCTOR
    SQL_ISO_VALUE_EXPRESSIONS = SQL_SQL92_VALUE_EXPRESSIONS

#### 3.3.4.4  Added `SQL_AGGREGATION_FUNCTIONS` bitmasks

The following bitmasks are added to `SQL_AGGREGATION_FUNCTIONS` to advertise support for additional ANSI SQL-defined aggregation functions:

| **BitMask**           | **ANSI SQL Function** |
|-----------------------|-----------------------|
| `SQL_AF_EVERY`        | EVERY                 |
| `SQL_AF_ANY`          | ANY                   |
| `SQL_AF_STDEV_OP`     | STDEV\_OP             |
| `SQL_AF_STDEV_SAMP`   | STDEV\_SAMP           |
| `SQL_AF_VAR_SAMP`     | VAR\_SAMP             |
| `SQL_AF_VAR_POP`      | VAR\_POP              |
| `SQL_AF_ARRAY_AGG`    | ARRAY\_AGG            |
| `SQL_AF_COLLECT`      | COLLECT               |
| `SQL_AF_FUSION`       | FUSION                |
| `SQL_AF_INTERSECTION` | INTERSECTION          |

`SQL_AF_EVERY` is a synonym for the current `SQL_AF_ALL`.

#### 3.3.4.5 Added Scalar Function bitmasks

Todo…

### 3.3.5 Language Extensions

For common constructs that are not part of SQL-92 (i.e., outer joins, function execution, datetime literals), ODBC defines escape clauses as a way to embed functionality within native commands that can easily be scanned, tweaked, and then passed-through to the back-end. These escape clauses provide a level of interoperability for basic operations while allowing clients familiar with a native syntax to leverage constructs within that syntax.

The following new ODBC escape clauses are added to the ODBC grammar. These clauses are not dependent upon the version of ODBC; applications determine support for the clauses through SQLGetInfo.

#### 3.3.5.1 Limit Clause

Interactive applications typically need a way to “preview” results, for example by showing the first n rows. Applications may also “page” by selecting the next n rows after skipping m.

Different databases support this through different syntax (top, first, skip, start, fetch first, limit, etc.) ANSI SQL 2003 defines window functions for defining a `ROW_NUMBER` that can then be filtered on and then selected out.

ODBC adds a new *ODBC-limit-escape*, defined as follows:

*ODBC-limit-escape* ::=
     *ODBC-esc-initiator* limit \[*skip-value comma*\] *take-value ODBC-esc-terminator*

Where *skip-value* is an integer specifying the number of rows to skip and *take-value* is an integer specifying the number of rows to return.

The optional *ODBC-limit-escape* clause immediately follows the *order-by-clause*, if present.

For example:

>Select \* from table {limit 10}

would fetch the first ten rows of the table, while

>Select \* from table {limit 20,10}

would skip twenty rows and then fetch the next ten.

Drivers advertise support for this escape clause through the new `SQL_LIMIT_ESCAPE_CLAUSE` *InfoType*. The value of the `SQL_LIMIT_ESCAPE_CLAUSE` is a bitmask made up of the following values:

* **SQL\_LC\_NONE** = 0x00000000L – the driver has no support for the limit escape clause
* **SQL\_LC\_TAKE** = 0x00000001L - the driver supports only the take portion of the limit escape clause
* **SQL\_LC\_TAKE\_AND\_SKIP** = 0x00000003L – the driver supports both take and skip in the limit escape clause

#### 3.3.5.2 Selecting Inserted Updated and Deleted Values

The *ODBC-return-escape* clause returns a table containing information from inserted, updated, and deleted records:

*ODBC-return-escape* ::=
     *ODBC-esc-initiator* return *select-list* from '*vendor-dml-statement*' *ODBC-esc-terminator*

For Inserts and Updates, the returned values are the values of the affected rows following the insert or update operation. For delete operations, the returned values represent the values of the rows at the time they were deleted.

If no rows are affected by the statement, an empty result is returned.

For example, the following statement returns a table containing the columns named "OrderId", "Item" and "Total" of a newly inserted row:

> {return OrderId, Item, Total from 'INSERT INTO Orders(Item, Price, Quantity) VALUES (''San Juan Islands Map'',24,2)'}

The following statement retrieves the new total of all rows with ids above 10 that were updated:

> {return total from 'UPDATE table SET amount=amount\*10 WHERE id &gt; 10'}

The *ODBC-return-escape* should not be used as a nested query within another statement.

Single quotation marks within *vendor-dml-statement* must be doubled. The driver replaces two adjacent single quotations marks within the dml statement with a single quotation mark. A single quotation mark terminates the dml statement.

If an array of parameter values is specified, then whether there is a result set for each set of parameter values or a single result that merges the results from each set of parameter values is defined by the `SQL_PARAM_ARRAY_SELECTS` SQLGetInfo *InfoType*.

If multiple rows are specified in an INSERT INTO...VALUES statement, or an array of parameter values are specified, then the order of rows returned matches the order of values specified. In all other cases, the order of returned rows is indeterminate.  

The driver may limit the columns in the select-list to include only the best row id columns returned by SQLSpecialColumns, as specified in the `SQL_RETURN_ESCAPE_CLAUSE` *InfoType*.

If the application requests columns that don't exist in the row and [`SQL_ATTR_DYNAMIC_COLUMNS`](#391-SQL_ATTR_DYNAMIC_COLUMNS) is *false*, or the driver only supports returning row id columns and the client specifies non-row id columns in the *select-list*, the driver returns `42S22` Column not found.

Drivers advertise support for this escape clause through the `SQL_RETURN_ESCAPE_CLAUSE` *InfoType* whose value is a bitmask made up of the following values.

| Value                          | Description                                                         |
|--------------------------------|---------------------------------------------------------------------|
| `SQL_RC_NONE` = 0              | The driver has no support for the return escape clause                                                |
| `SQL_RC_INSERT_SINGLE_ROWID`   | The driver supports returning row id fields for a single row from an INSERT INTO...VALUES statement   |
| `SQL_RC_INSERT_SINGLE_ANY`     | The driver supports returning any fields for a single row from an INSERT INTO...VALUES statement      |
| `SQL_RC_INSERT_MULTIPLE_ROWID` | The driver supports returning row id fields for multiple rows from an INSERT INTO...VALUES statement  |
| `SQL_RC_INSERT_MULTIPLE_ANY`   | The driver supports returning any fields for multiple rows from an INSERT INTO...VALUES statement     |
| `SQL_RC_INSERT_SELECT_ROWID`   | The driver supports returning row id fields from an INSERT INTO...SELECT statement                    |
| `SQL_RC_INSERT_SELECT_ANY`     | The driver supports returning any fields from an INSERT INTO...SELECT statement                       |
| `SQL_RC_UPDATE_ROWID`          | The driver supports returning row id fields from an UPDATE statement                                  |
| `SQL_RC_UPDATE_ANY`            | The driver supports returning any fields from a UPDATE statement                                      |
| `SQL_RC_DELETE_ROWID`          | The driver supports returning row id fields from a DELETE statement                                   |
| `SQL_RC_DELETE_ANY`            | The driver supports returning any fields from a DELETE statement                                      |
| `SQL_RC_SELECT_INTO_ROWID`     | The driver supports returning row id fields from a SELECT INTO statement                              |
| `SQL_RC_SELECT_INTO_ANY`       | The driver supports returning any fields from a SELECT INTO statement                                 |

#### 3.3.5.3 Format Clause

The *ODBC-format-escape* clause enables clients to return results as a JSON-formatted string.

*ODBC-format-escape* ::= *ODBC-esc-initiator* RETURNING *data-type* *json-format-clause* *ODBC-esc-terminator*

*json-format-clause* ::= FORMAT JSON \[ENCODING {UTF8 | UTF16 | UTF32}\]

For results returned as JSON, *data-type* must be a character string or binary type. If ENCODING is specified, then *data-type* must be a binary type.

Drivers advertise support for this escape clause through the `SQL_FORMAT_ESCAPE_CLAUSE` *InfoType* whose value is a bitmask made up of the following values.

| Value                              | Description                                                       |
|------------------------------------|-------------------------------------------------------------------|
| `SQL_FC_NONE` = 0x00000000L        | The driver has no support for the format escape clause            |
| `SQL_FC_JSON` = 0x00000001L        | The driver supports returning results as a JSON character string  |
| `SQL_FC_JSON_BINARY` = 0x00000002L | The driver supports returning results as a JSON binary string     |

#### 3.3.5.4 Native Syntax

ODBC clients use the `SQL_NOSCAN` statement attribute to specify that the command text is in a native syntax and should not be parsed by the driver. In this case, the entire statement must be in a native syntax and escape clauses must not be used as the driver will not scan the statement but instead pass it directly to the server.

The *ODBC-native-escape* clause enables clients to embed native syntax within a SQL92 statement:

*ODBC-native-escape* ::= *ODBC-esc-initiator* native '*command-text*' \[*passing-clause*\] \[*results-clause*\] *ODBC-esc-terminator*

*passing-clause* ::= PASSING ( *argument-definition* \[, *argument-definition*\]… )

*argument-definition* ::= *argument-value* \[(*native-type*\)] \[AS *identifier*\]

*argument-value* ::= *literal* | *parameter-marker* | *expression*

*results-clause* ::= *columns-clause* | *scalar-returning-clause* 

*columns-clause* ::= COLUMNS ( *field-definition* \[, *field-definition*\]… )

*scalar-returning-clause* ::= RETURNING *type* \[*json-format-clause*\] 

*type* ::= {*data-type* | ROW ( *field-definition* \[, *field-definition*\]… ) } \[ARRAY | MULTISET\]

*field-definition* ::= *field-name type*

Where *command-text* is a textual query in the native language of the service, and *native-type* is the native type of the argument.  If *native-type* is not provided, the driver should attempt to use a default type (generally string) which may fail if the native syntax is unable to perform the necessary conversion.

Whether arguments in the *passing-clause* must be named is dependent upon the underlying native syntax.

The *results-clause* is required for retrieving results from the native syntax and when embedding the native syntax in place of a query expression within a standard SQL statement.

If *json-format-clause* is specified, *type* must be a string or binary type.

Single quotation marks within the native syntax must be doubled. The driver replaces two adjacent single quotations marks within the native syntax with a single quotation mark. A single quotation mark terminates the native command text.

Drivers advertise support for this escape clause through the new `SQL_NATIVE_ESCAPE_CLAUSE` *InfoType* whose value is the character string “`Y`” if the escape clause is supported; “`N`” otherwise.

#### 3.3.5.5 Refresh Schema

The *ODBC-refresh-schema-escape* clause to enable clients to request that schema be refreshed, defined as follows:

*ODBC-refresh-schema-escape* ::=
     *ODBC-esc-initiator* refresh \[*sql-identifier*\[, *sql-identifier*\]\*\] *ODBC-esc-terminator*

The optional list of sql-identifiers provide a hint to the driver as to the specific objects from the information schema objects (tables, udts, etc.) that should be refreshed. The driver is always allowed to refresh additional schema objects, particularly in the case where the tables/udts/etc are virtualized based on a non-relational view.

#### 3.3.6 SAFECONVERT

The SAFECONVERT function, which can be invoked using the function escape clause, safely converts an expression to a given type.

SAFECONVERT ( *value-exp*, *data-type* )

The result of the SAFECONVERT function is the value of *value-exp* converted to *data-type*, where *data-type* is one of the keywords listed for the CONVERT function.  If the value of *value-exp* cannot be converted to *data-type*, the result is null.

The SAFECONVERT function supports the same set of conversions as the CONVERT function (as reported by SQLGetInfo).

#### 3.3.7 TYPEOF

The TYPEOF function, which can be invoked using the function escape clause, returns the type name of an expression.

TYPEOF ( *value-exp* )

#### 3.3.8 SQLTYPEOF

The SQLTYPEOF function, which can be invoked using the function escape clause, returns the canonical SQLTYPE name of an expression, which can be used in [SAFECONVERT](#336-SAFECONVERT).

SQLTYPEOF ( *value-exp* )

#### 3.3.9 SAFECAST

The SAFECAST function, which can be invoked using the function escape clause, allows an expression to be treated as the specified user-defined or datasource-specific type.

SAFECAST ( *value-exp*, *data-type-name* )

The result of the SAFECAST function is the value of *value-exp* converted to *data-type-name*, where *data-type-name* is datasource-specific type, or the fully qualified name of a user defined or structural type. If the value of *value-exp* cannot be converted to *data-type-name*, the result is null.

### 3.3.10 Added Reserved Words

The following are added to the list of ODBC reserved words:

* ARRAY
* MULTISET
* ROW

## 3.4 String Format

A new SQLCHAR\*-valued descriptor field, `SQL_DESC_MIME_TYPE`, is added to \[ALL\] descriptors to specify the format of a field or parameter whose `SQL_DESC_TYPE` specifies a string or binary type. The value of this descriptor field is the mime type of the data (i.e., application/json, application/xml,…), or an empty string if the field or parameter is not a string or binary type, or if the format is not known.

## 3.5 Schema Inference

Schema can be inferred over a schema-less source, such as a JSON, XML, or CSV document, to expose a standard relational view over the data.

### 3.5.1 `SQL_SCHEMA_INFERENCE` GetInfo

Drivers that support schema inference return `SQL_TRUE` when an application calls SQLGetInfo with the new `SQL_SCHEMA_INFERENCE` *InfoType*.

### 3.5.2 `SQL_ATTR_INFERENCE_ACCURACY` Connection Attribute

A new connection attribute, `SQL_ATTR_INFERENCE_ACCURACY`, whose value is an integer between 1-100, is added to support schema inference by the driver.

Setting this attribute to 100 means that all rows should be read in order to ensure that the inference is accurate for every row in the result.

Setting this attribute to zero specifies that the driver should not infer schema. In this case, a driver that does not have an explicit way to determine schema returns an empty result when calling SQLColumns, and every column is treated as a [dynamic column](#39-dynamic_columns). Applications can then do their own sampling of the data, and can impose column types in their queries using CONVERT.

A value between 1 and 99 gives an approximate estimation of the quality of the inference. The driver may adjust the number of rows sampled, the method of sampling, or how specific of a type or length to infer based on this setting.

Setting this attribute (including setting to its current value) forces the driver to re-infer schema.

Attempting to get or set this attribute for drivers that do not support schema inference returns `SQL_ERROR` with a diagnostic code of HYC00, Optional feature not implemented.

## 3.6 Getting Data during Fetch

In ODBC 3.x, applications read large data values by placing them at the end of the select list and calling SQLGetData after all bound columns have been populated.

Applications can efficiently retrieve large data values (such as potentially large string or binary values, nested collections, and structured columns) that appear in any order by setting the [`SQL_DESC_DATA_AT_FETCH`](#361-sql_desc_data_at_fetch-descriptor-field) descriptor field to `SQL_TRUE` for columns to be retrieved as they become available during the fetch operation. The application is notified during the fetch operation when data-at-fetch columns are available to be retrieved via SQLGetData, or a nested statement handle for nested collections and structured columns.

In order to be notified when an unbound column is available to be retrieved during the fetch operation:

1. The application binds all columns to be fetched directly into bound buffers. Binding a column automatically sets the `SQL_DESC_DATA_AT_FETCH` value to `SQL_FALSE`.

2. For each column to be retrieved at fetch time, the application calls SQLSetDescField, specifying `SQL_DESC_DATA_AT_FETCH` with a value of `SQL_TRUE`.  Setting `SQL_DESC_DATA_AT_FETCH` to `SQL_TRUE` for a column automatically unbinds the column by setting the `TargetValuePtr` to null.

3. For nullable columns, the application sets `SQL_DESC_INDICATOR_PTR` so that the driver can signal a null value for the column.
 
4. The application calls SQLFetch/SQLFetchScroll
    -   Driver begins populating the bindings
    -   Driver returns `SQL_DATA_AVAILABLE` from SQLFetch/SQLFetchScroll to signal data-at-fetch columns are available.
        - Calling SQLCloseCursor (or SQLFreeStmt(SQL_CLOSE)) in the `DATA_AVAILABLE` state cancels the fetch operation and closes the cursor.

5. The application calls SQLNextColumn to find out which column is available. Applications must not assume that the columns are returned in any particular order.
    -   Driver returns the ordinal of the available column
    -   For rowset sizes greater than 1, the row for which the column is available can be determined through the `SQL_ATTR_ROWS_FETCHED_PTR` statement attribute. The driver must populate the rows of a given column in order, but the client cannot assume that any other columns have been populated until the entire fetch sequence has completed and the initial fetch or a subsequent call to SQLNextColumn returns a value other than `SQL_DATA_AVAILABLE`.
	- The following operations are valid in a `SQL_DATA_AVAILABLE` state. For any other operations on a statement handle in the `SQL_DATA_AVAILABLE` case, the DM returns HY010, function sequence error.
		- Changing binding information in the ARD for the current column*
		- Calling SQLGetData for the current column* (if not structured or a nested collection)
		- Calling SQLGetNestedHandle for the current column* (if structured or a nested collection)
		- Attempting to fetch from a nested handle associated with the current column*
		- Calling SQLCloseCursor
		- Calling SQLNextColumn
		- Calling SQLFetch/SQLFetchScroll
		- Calling SQLDescribeCol/SQLGetDescField/SQLGetDescRecord
	
	    *The Driver Manager returns SQLState 07009, Invalid descriptor index, if the application attempts to call SQLBindCol, SQLSetDescField, SQLSetDescRecord in the `DATA_AVAILABLE` case for a column other than the current column, or if SQLGetData or SQLGetNestedHandle is called for a column other than the current column  and the driver does not support SQL_GD_ANY_ORDER.
	- Applications SHOULD NOT change the offset value indicated by SQL_DESC_BIND_OFFSET_PTR while in a `SQL_DATA_AVAILABLE` state. As the driver can populate columns and indicators in any order until the fetch operations completes, changing this value while in a `SQL_DATA_AVAILABLE` state has indeterminate affects. 
	
6.  The app can update the binding information for the current column only, which will used when populating subsequent rows for the given column. Note that changing the binding has no affect on previously populated rows.

7.  App typically retrieves the data for the current column by calling SQLGetData, or by reading data from the nested handle, as appropriate.

8.  App calls [SQLNextColumn](#61-sqlnextcolumn) to continue fetching the current row
    -   Any nested handles on the same statement are closed, unless [`SQL_GD_CONCURRENT`](#361-sql_gd_concurrent-bit-for-sql_getdata_extensions) is specified (in which case any nested handles are closed upon the next call to fetch or close cursor on the parent statement handle.)
    -   Driver returns `SQL_DATA_AVAILABLE`, along with the ordinal of the next column available to be retrieved, if additional columns are available
    -   Driver returns `SQL_SUCCESS` or `SQL_SUCCESS_WITH_INFO` when all bound or data-at-fetch columns have been retrieved. The application must not assume any bindings or diagnostic information associated with the fetch has been populated until SQLNextColumn returns `SQL_SUCCESS` or `SQL_SUCCESS_WITH_INFO`, or `SQL_ERROR`.

9.  App can call SQLGetData, or retrieve nested data, for additional columns beyond the last column retrieved, or subject to the ordering constraints of the driver.

### 3.6.1 `SQL_DESC_DATA_AT_FETCH` descriptor field

A new application row descriptor (ARD) field, `SQL_DESC_DATA_AT_FETCH`, is added to specify that a column is to be retrieved through a `SQL_DATA_AVAILABLE` state at fetch time.

- `SQL_FALSE` (default) specifies that the driver will return data for the column directly to the application's buffer if a binding is specified
- `SQL_TRUE` specifies that the driver will return `SQL_DATA_AVAILABLE` when reading the column. Setting `SQL_DESC_DATA_AT_FETCH` to `SQL_TRUE` for a column automatically unbinds the column by setting the `TargetValuePtr` to null.

Binding a column (through SQLBindCol or by setting the `SQL_DESC_DATA_PTR` to a non-null value) sets the `SQL_DESC_DATA_AT_FETCH` value to `SQL_FALSE`.

For nullable columns, clients should also specify *SQL\_DESC\_INDICATOR\_PTR* for the driver to indicate that the value of the data-at-fetch parameter is null. If *SQL\_DESC\_INDICATOR\_PTR* is not specified for a column whose value is null, the driver returns an error with SQLState `22002`, Indicator variable required but not supplied.

### 3.6.2 `SQL_GD_CONCURRENT` bit for `SQL_GETDATA_EXTENSIONS`

A new SQL GetData Extension, `SQL_GD_CONCURRENT`, is added. 

If the driver reports `SQL_GD_CONCURRENT`, fetching partial data from one column (using SQLGetData) or on a nested statement handle does not affect the position or state of any other column or nested handle. If the driver specifies `SQL_GD_ANY_ORDER` and does not specify `SQL_GD_CONCURRENT`, calling SQLGetData or retrieving data from a nested handle resets the position of any columns previously fetched using SQLGetData to the beginning and closes the cursor (but does not automatically free the hstmt) of any previously fetched nested columns for this row.

Note that SQL_GD_CONCURRENT does not imply that the driver supports multiple concurrent asynchronous requests associated with the same parent statement handle. The root statement handle defines the scope for concurrent asynchronous operations.

## 3.7 Variable Typed Columns
Variable typed columns are columns whose type is unknown or may change on a row-by-row basis.

### 3.7.1 Schema Extensions for Variable Typed Columns

Columns whose type may vary across rows are described to ODBC 4.0 clients using the `SQL_VARIANT_TYPE` value for `DATA_TYPE` and `SQL_DATA_TYPE` in SQLColumns, and for the `SQL_DESC_TYPE` and `SQL_DESC_CONCISE_TYPE` descriptor fields when describing a result prior to fetching the first row. The `SQL_VARIANT_TYPE` data type must not be used in SQLColumns, or returned in a parameter or row descriptor, for ODBC 3.5 and earlier clients.

### 3.7.2 Query Extensions for Variable Typed Columns

Variable typed columns used in expressions should first be cast to an appropriate operand type. The [SAFECONVERT](#336-safeconvert) canonical function can be used to safely cast the variable typed column to the expected type, or null if the column cannot be cast to the expected type.

The [TYPEOF](#337-TYPEOF) canonical function can be used to get the type name returned by a given expression. 

The [SQLTYPEOF](#338-SQLTYPEOF) canonical function can be used to get the name of the SQLTYPE returned by a given expression, which can be used in SAFECONVERT. 

### 3.7.3 Response Extensions for Variable Typed Columns

The application can attempt to set the type information in the IRD to an "expected" type. Drivers SHOULD accept setting the IRD for variable type columns but MAY return an error if the type information for the column is a known, fixed type. 

As data is read for a row, the IRD for each variable typed column is updated to reflect the actual type information for the current row. Columns whose `SQL_DESC_TYPE` or `SQL_DESC_CONCISE_TYPE` descriptor fields indicate `SQL_VARIANT_TYPE` are updated to reflect the actual type of the data to be read.

Depending on the value of [`SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`](#3731-sql_attr_type_exception_behavior), if the descriptor is changed by the driver when reading a column the driver may return `SQL_METADATA_CHANGED`.

Upon receiving `SQL_METADATA_CHANGED`, the application can call SQLNextColumn to determine the index of the column that has changed, get the current information for that column, adjust bindings for the subsequent fetches of the current column only (including setting the IRD to the "expected" type), and can call SQLGetData or SQLGetHandle, as appropriate for the type of the column. Any changes to the descriptor record persist for future uses of that descriptor across rows.

Note that SQLGetData starts at the beginning of the value and reads sequentially until all bytes have been consumed. If a conversion failure occurs after the first call to SQLGetData, the rest of the data for that column will not be retrievable. 

The application calls SQLNextColumn to continue fetching values for any remaining columns in the row. The application must not assume any bindings or diagnostic information associated with the fetch have been populated until SQLNextColumn returns `SQL_SUCCESS` or `SQL_SUCCESS_WITH_INFO`, or `SQL_ERROR`.

Output parameters that are bound with a type of `SQL_VARIANT_TYPE` MUST be bound as data-at-exec parameters. When executing a statement that contains data-at-exec output parameters, the driver returns `SQL_PARAM_DATA_AVAILABLE`. The application calls SQLParamData to find out which parameter is available and then calls SQLGetData or SQLGetNestedHandle for that parameter, as appropriate.

#### 3.7.3.1 `SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`

`SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR` is a `SQLUSMALLINT` value that allows the client to control how type exceptions occur when retrieving bound values.  It does not affect SQLGetData.

* **SQL\_TE\_ERROR** – (Default) Return an error if the data value is incompatible with the bound type.
* **SQL\_TE\_CONTINUE** – Set the `str_type_or_ind_ptr` value (if not null) to `SQL_TYPE_EXCEPTION` if the data value is incompatible with the bound type and continue, or error if `str_type_or_ind_ptr` is a null pointer.
* **SQL\_TE\_REPORT\_EXCEPTION** – Set the `str_type_or_ind_ptr` value (if not null) to `SQL_TYPE_EXCEPTION` if the data value is incompatible with the bound type and return `SQL_METADATA_CHANGED`. The application can retrieve the value for the column using SQLGetData or SQLGetNestedHandle, as appropriate. SQLGetNestedHandle will always return a fresh statement handle for a structured- or collection-valued column whose IRD has changed. The application can update the binding for this column which will used when populating subsequent rows for the given column. Changing binding information has no affect on previously fetched values.
* **SQL\_TE\_REPORT\_ALL** – Set the `str_type_or_ind_ptr` value (if not null) to `SQL_TYPE_EXCEPTION` and return `SQL_METADATA_CHANGED` if the data value is incompatible with the bound type, or if the type information in the IRD (for rows) or IPD (for parameters) has changed. The application can retrieve the value for the column using SQLGetData or SQLGetNestedHandle, as appropriate. The application can update the binding for this column which will used when populating subsequent rows for the given column. Changing binding information has no affect on previously fetched values.

### 3.7.4 Write Extensions for Variable Typed Columns

Applications can use the CONVERT scalar function to specify a SQL_TYPE to be used when setting the value for a variable-typed column in an INSERT or UPDATE statement. In the absence of a CONVERT, the driver will store the value in an appropriate type given the value specified in the INSERT or UPDATE statement for the variable-typed column.

Applications can pass variable-typed parameters at execute time by setting the ParameterType in SQLBindParameter, or `SQL_DESC_TYPE` in the IPD, to `SQL_VARIANT_TYPE`, and `StrLen_or_IndPtr` to indicate `SQL_DATA_AT_EXEC`. 

After calling SQLExecute or SQLExecDirect, the driver returns `SQL_NEED_DATA` for any data-at-exec input parameters. The application calls SQLParamData to discover which parameter the driver is ready to process. 

For input data-at-exec parameters, the application fills in the IPD to specify the type of the parameter (including the SQL_DESC_TYPE_NAME for UDTs) and calls SQLPutData or SQLGetNestedHandle, as appropriate, to send the information for the current parameter.

The application calls SQLParamData to continue processing any remaining parameters, using the information in the IPD but ignoring any bound value if SQPutData or SQLGetNestedHandle has been called for this parameter.

### 3.7.5 Data Definition Extensions for Variable Typed Columns

Clients can call SQLGetTypeInfo with a *DataType* value of `SQL_VARIANT_TYPE` to determine the implementation-specific name of any types that can be used to hold variable-typed data.

## 3.8 Variable Length Columns

ODBC 4.0 drivers report the expected size of string and binary columns in SQLColumns. Applications generally base the allocation of bound buffers on this size.

In some cases, the size of the data may exceed the length of the bound buffer. In this case, the application has the option of ignoring the extra data as in ODBC 3.x (the driver returns 01004, String data right truncated) or having the driver return `SQL_MORE_DATA` at fetch time, based on the value of the [`SQL_ATTR_LENGTH_EXCEPTION_BEHAVIOR`](#381-sql_attr_length_exception_behavior) statement attribute.

When the length of data exceeds the bound buffer for string or binary data, and the application has set `SQL_ATTR_LENGTH_EXCEPTION_BEHAVIOR` to `SQL_LE_REPORT`, the driver populates as much data as fits in the allocated buffer, sets `str_len_or_ind_ptr` to indicate the total amount of data available, if known, or `SQL_NO_TOTAL` if the driver does not know how much additional data is to be written, and returns `SQL_MORE_DATA`. The application then calls SQLNextColumn in order to determine the column of partially fetched data. At this point the application can call SQLGetData in order to retrieve the remainder of the string or binary data. The application calls SQLNextColumn in order to continue processing additional columns.

### 3.8.1 `SQL_ATTR_LENGTH_EXCEPTION_BEHAVIOR`

The new `SQL_ATTR_LENGTH_EXCEPTION_BEHAVIOR` is a `SQL_USMALLINT` value that allows the client to control how string and binary overflows are handled when retrieving bound values.

* **SQL\_LE\_CONTINUE** – (Default) For string or binary values larger than the bound buffer length, set the `str_type_or_ind_ptr` value to the total length available, if known, otherwise `SQL_NO_TOTAL`. When the driver completes fetching the row/executing the statement it will return `SQL_SUCCESS_WITH_INFO` with a `SQLSTATE` of `01004`, String data, right truncated.
* **SQL\_LE\_REPORT** – Return `SQL_MORE_DATA` when the string or binary value of a row or parameter does not fit the bound buffer. The application can use SQLNextColumn to determine the column with additional data available, and can call SQLGetData to retrieve the remainder of the column/output parameter using the `SQL_ARD_TYPE` or `SQL_APD_TYPE` (or the equivalent C type), as appropriate. The application then calls SQLNextColumn or SQLParamData, as appropriate, to continue processing.

## 3.9 Dynamic Columns

Columns that are known and consistent across rows are exposed in SQLColumns. Applications can bind to these columns in the regular way.

ODBC is extended to support dynamic columns (columns not returned in SQLColumns). The set of such “Dynamic” columns may vary in order, type, or existence from row to row.

Dynamic columns are discovered by the application when retrieving results.

A table may be entirely unschematized (in which case SQLColumns returns an empty result set) or partially schematized (in which case SQLColumns returns the columns that are known and consistent across rows).

### 3.9.1 `SQL_ATTR_DYNAMIC_COLUMNS` Statement Attribute

The new `SQL_ATTR_DYNAMIC_COLUMNS` statement attribute controls whether or not additional, dynamic columns may be returned for unbounded select results.
 
The default for this statement attribute is *False*.

Data sources with fixed schemas always return *False* for this value. Attempting to set this value for such a data source returns `SQL_SUCCESS_WITH_INFO` with a diagnostic code of 01S02, Option Value Changed, and reading this value will continue to return *False*.

If set to *True*, SQLFetch, SQLFetchScroll and SQL_NextColumn return `SQL_DATA_AVAILABLE` if new columns are discovered/added to the IRD while retrieving results.

If `SQL_ATTR_DYNAMIC_COLUMNS` is *True*, then bound columns in the ARD that don’t apply to the current row have their `len_or_ind_ptr` set to `SQL_DATA_UNAVAILABLE`. If `SQL_ATTR_DYNAMIC_COLUMNS` is *False*, then only known columns are allowed in a select list and only known columns are returned when \* is specified.

Applications may change `SQL_ATTR_DYNAMIC_COLUMNS` from to *True* to *False* at any time to ignore dynamic columns. Attempting to change `SQL_ATTR_DYNAMIC_COLUMNS` from *False* to *True* on a statement handle with an open cursor results in HY010, Function Sequence Error (raised by the Driver Manager).

### 3.9.2 Schema Extensions for Dynamic Columns

If [`SQL_ATTR_DYNAMIC_COLUMNS`](#391-SQL_ATTR_DYNAMIC_COLUMNS) is *True*, tables that support properties not defined in SQLColumns are returned from SQLTables using the new `“OPEN TABLE”` table type. If `SQL_ATTR_DYNAMIC_COLUMNS` is *False*, such tables are returned with a table type of `"TABLE"`.

### 3.9.3 Query Extensions for Dynamic Columns

A fully explicit select-list (i.e., anything that doesn't contain \*) imposes a structure on the results. Such results only contain the specified columns. When `SQL_ATTR_DYNAMIC_COLUMNS` is *True*, a select list that includes \* also includes any dynamic (unschematized) columns. Dynamic columns must be explicitly included in the select-list in order to be referenced by name or ordinal in the order-by clause.

Structured columns specified in a select list implicitly select all columns (declared and dynamic) of the structured type.

For OPEN TABLEs, any valid identifier for the driver is allowed as a column reference in a SQL Statement (select-list, expression in a where-clause, etc.). If more than one column-source is in scope, any references to unschematized columns MUST be qualified with the row source. Columns that aren’t defined for a particular row are considered null in evaluating the expression. Such columns in a select list or an expression are treated as [variable typed columns](#37-Variable-Typed-Columns).

For tables not reported as open (including all tables for pre-ODBC 4.0 clients), it remains an error to reference an undefined column.

### 3.9.4 Response Extensions for Dynamic Columns

Result descriptor functions (SQLNumResultCols, SQLDescribeCol, SQLColAttribute(s), SqlGetDescField/Rec) called before the first call to SQLFetch/SQLFetchScroll describe the known/fixed schema columns (i.e., those explicitly selected, or those defined in schema if \* is specified in the select list). After that, these functions reflect the columns discovered so far. The driver appends records to the IRD as the application reads data to describe dynamic columns not explicitly selected or defined in schema. Once discovered, a column is added to the IRD for the lifetime of the IRD. 

#### 3.9.4.1 Retrieving Dynamic Columns

If `SQL_ATTR_DYNAMIC_COLUMNS` is set to *True*, the driver can return dynamic columns as data-at-fetch columns when fetching a row.

Dynamic (unschematized) columns are identified by name. If the driver encounters a dynamic column that does not match the name of an existing unschematized IRD record not previously used for this row, the driver:

1.  Creates a new IRD record for the column; this becomes a new, unbound column in the result. The application must bind the column, or set the [`SQL_DESC_DATA_AT_FETCH`](#361-sql_desc_data_at_fetch-descriptor-field) descriptor field to `SQL_TRUE`, in order to retrieve the value in subsequent calls to SQLFetch. 
Once added to the IRD, dynamic columns become "known columns". If the type and length information for a known dynamic column changes on subsequent rows, the driver should use the same IRD record and return SQL_METADATA_CHANGED based on the [`SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`](#3731-SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR). Note, however, that applications must continue to be prepared to deal with multiple IRD records with the same name, and the driver must create a new IRD record if a dynamic column with the same name has already been returned for the current row.
Once added, the descriptor fields are a permanent part of the descriptor until it is freed.

2.  Returns `SQL_DATA_AVAILABLE` from SQLFetch or SQLFetchScroll

    1.  Application calls [SQLNextColumn](#61-sqlnextcolumn) to retrieve the ordinal of the descriptor record that describes the column.

    2.  The application optionally retrieves the data using SQLGetData or a nested handle, as appropriate.
  
    3.  The application can optionally bind the column which will used when populating subsequent rows for the given column. Note that changing binding information has no affect on previously fetched values.

    3.  Application calls SQLNextColumn to continue fetching the current row.

Once discovered, dynamic columns can be bound like any other column. If a bound dynamic column does not exist for a row, the driver sets its `str_len_or_indicator_ptr` to `SQL_DATA_UNAVAILABLE`. If `str_len_or_indicator_ptr` is null for such a bound dynamic column that doesn't exist for the current row, SQLFetch, SQLFetchScroll, or SQLNextColumn returns an error (22002, Indicator variable required but not supplied).

If the driver returns `SQL_DATA_AVAILABLE`, the application must not assume that any bound columns have been populated, regardless of their ordinal, or that any diagnostic information associated with the fetch has been populated, until the entire fetch operation completes with `SQL_SUCCESS` or `SQL_SUCCESS_WITH_INFO`, or `SQL_ERROR`.

### 3.9.5 Write Extensions for Dynamic Columns

Open Tables may support inserting or updating dynamic columns.

To create/update dynamic columns, clients can specify column names outside of those returned in SQLColumns in the column list for an insert/update statement. An Insert or Update statement that specifies an unschematized column adds a dynamic column for that row in the table. Setting the value of a dynamic column to null removes that column from that row.

## 3.10 Structured Columns

ODBC 4.0 adds support for structured columns.

The ability to report structured columns leverages the extensible type facility supported in ODBC 3.8 and greater. If a client has not called SQLSetEnvAttr with `SQL_ATTR_ODBC_VERSION` attribute set to `ODBC_OV_ODBC3_80` or greater, then the server must not return the `SQL_ROW` or `SQL_UDT` values in the `DATA_TYPE` or `SQL_DATA_TYPE` column descriptions, or `SQL_DESC_TYPE` or `SQL_DESC_CONCISE_TYPE` in any row or parameter descriptor. In this case the server may either flatten the structured value into multiple columns, create a join table containing the nested results, return the nested results as a string, or omit the nested results, as appropriate for the data source.

### 3.10.1 Schema Extensions for Structured Columns

Structured columns may have a named type, whose structure can be described in [SQLStructuredTypeColumns](#64-sqlstructuredtypecolumns).

Structured columns with no backing type can be described by passing a path as the `TABLE_NAME` to SQLColumns, or the `UDT_NAME` to SQLStructuredTypeColumns. The dot-separated path starts with the table or named structural type name in which the anonymous type is used, can traverse an arbitrary number of `SQL_ROW` columns, and must terminate on a column whose type is `SQL_ROW`. Any identifiers in the path containing the dot character (.) must be quoted using the appropriate identifier quote character. Clients must assume that instances of structured columns with no backing type may include undeclared ([dynamic](#39-dynamic-columns)) columns.

The following columns are added, in order, to the set of columns returned by SQLColumns when requested by ODBC 4.0 or greater applications:

| **Column name**  | **Column number** | **Data type** | **Comments**                                           |
|---------------------|---------------|---------------|---------------------------------------------------------|
| `CHAR_SET_CAT`    | 19                | Varchar       | Catalog of the Character set for the column, or null.                  |
| `CHAR_SET_SCHEM`  | 20                | Varchar       | Schema of the Character set for the column, or null.                   |
| `CHAR_SET_NAME`   | 21                | Varchar       | Character set name for the column, or null.                            |
| `COLLATION_CAT`   | 22                | Varchar       | Catalog of the Collation for the column, or null.                      |
| `COLLATION_SCHEM` | 23                | Varchar       | Schema of the Collation for the column, or null.                       |
| `COLLATION_NAME`  | 24                | Varchar       | Name of the Collation for the column, or null.                         |
| `UDT_CAT`         | 25                | Varchar       | Catalog of the structured type if the column is a UDT, otherwise null. |
| `UDT_SCHEM`       | 26                | Varchar       | Schema of the structured type if the column is a UDT, otherwise null.  |
| `UDT_NAME`        | 27                | Varchar       | Name of the structured type if the column is a UDT, otherwise null.    |

The first six are set to null if not supported by the driver. The last three can be passed to SQLStructuredTypeColumns to describe the columns of a UDT, and are null for `DATA_TYPE`/`SQL_DATA_TYPE` values other than `SQL_UDT`.

### 3.10.2 Query Extensions for Structured Columns

Referencing a structured column by name references the entire structured column.

For example:

> Select EmpID, Address from Employee

would return a result with two columns; one named “EmpID” containing the employee ID, and one named “Address” containing the address as a structured column.

A subset of the properties for a structural type can be selected by following the reference to the structural column with a comma-separate list of properties, enclosed in parenthesis.

For example:

> Select EmpID, Address(Region,Country) from Employee

would return a result with two columns; one named “EmpID” containing the employee ID, and one named “Address” containing only the Region and Country columns from the Address column. This is equivalent to the following query:

> Select EmpID, ROW(Region, Country) as Address
> from Employee

Two structured values are comparable if, and only if, they have the same structural members. For two structured values *SV1* and *SV2*. *SV1 = SV2* is true if every member *m1* in *SV1* equals the corresponding member *m2* in *SV2*, otherwise *SV1 = SV2* is false. *SV1 < SV2* is true if *SV1 != SV2* and every member *m1* in *SV1* is less than or equal to the corresponding member *m2* in *SV2*, otherwise *SV1 < SV2* is false.

It is generally not possible to order by structured values, although it is possible to order by primitive members of a structural value.

Individual members of a structured column may be referenced as the name of the structured column, followed by a single dot (.), followed by the name of the member. This naming schema is recursive (i.e., when referencing a member of a structured member that is itself a member of a structured member.

For example:

> Select EmpID, Address.Region, Address.Country from Employee

would return a result with three columns; one named “EmpID” containing the employee id, one named “Address.Region” containing the region member of the employee’s address, and one named “Address.Country” containing the country member from the employee’s address.

### 3.10.3 Response Extensions for Structured Columns

To retrieve structured result columns, the client specifies [`SQL_DESC_DATA_AT_FETCH`](#36-getting-data-during-fetch) for the column before calling SQLFetch or SQLFetchScroll.

For non-null values, the driver returns `SQL_DATA_AVAILABLE` for the structured column, at which time the can client call [SQLGetNestedHandle](#62-sqlgetnestedhandle) to obtain a statement handle (HSTMT) on which the nested results can be described, bound, and fetched using the appropriate standard ODBC function calls. Structural columns not specified as `SQL_DATA_AT_FETCH` are not returned by the driver. Attempting to return a structural value for a bound column results in a type exception, which may be handled according to the value of [`SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`](#3731-sql_attr_type_exception_behavior).

For null values, the driver sets `SQL_DESC_INDICATOR_PTR` to `SQL_NULL_VALUE`. If `SQL_DESC_INDICATOR_PTR `is not specified for a structured column whose value is null, the driver returns an error with SQLState `22002`, Indicator variable required but not supplied.

#### 3.10.3.1 Result Descriptors for structured columns

For structured columns with named types, the value of the `SQL_DESC_TYPE` or `SQL_DESC_CONCISE_TYPE` descriptor fields is `SQL_UDT` and the `TYPE_NAME` attribute returns the qualified name of the type.

Additional descriptor fields can be used to describe UDTs in results:

* `SQL_DESC_USER_DEFINED_TYPE_CATALOG`
* `SQL_DESC_USER_DEFINED_TYPE_SCHEMA`
* `SQL_DESC_USER_DEFINED_TYPE_NAME`

Note: ANSI also defines `USER_DEFINED_TYPE_CODE` (1045) with a value of “`DISTINCT`” (1) or “`STRUCTURED`” (2).

For structured columns not described by a named type, the value of the `SQL_DESC_TYPE` or `SQL_DESC_CONCISE_TYPE` descriptor fields is `SQL_ROW`.

Calling GetNestedHandle for an unnamed structured column returns a statement handle with descriptor fields describing any known columns of the nested column.

### 3.10.4 Write Extensions for Structured Columns

Clients may specify the value for a member of a structured column in an INSERT or UPDATE statement by referencing the member directly. For Example:

> UPDATE Employee SET FullName.FirstName = 'John', FullName.LastName='Smith'

Dynamic columns can similarly be added to structured columns defined as open, for example:

> UPDATE Employee SET FullName.Foo = 'Bar'

Alternatively, clients may specify the value of a structured column in an INSERT or UPDATE statement using the *row-value-constructor* defined as follows:

*row-value-constructor* ::=
ROW *left-paren value-expression* \[*comma* *value-expression*\]… *right-paren*

For example:

> UPDATE Employee SET FullName = ROW('John', 'Smith')

All members must be specified in the order they occur in the definition of the structured type being updated. To update a subset of properties, a column-list can be added to the structured column:

> UPDATE Employee SET FullName(FirstName, LastName) = ROW('John', 'Smith')

In this case, the order of the members of the row-value-constructor must match the order of columns in the column-list applied to the structured column.

Appending a column-list to the structured column can also be used to add dynamic columns to the structured column, for example, the following statements add a “Foo” member to the FullName column:

> UPDATE Employee SET FullName(Foo) = ROW('Bar')

> UPDATE Employee SET FullName(FirstName, Foo) = ROW('John', 'Bar')

Side note: ANSI has two forms; one that is prefixed with “ROW” and can have one or more elements, and one that is not prefixed and must have at least two elements.

Alternatively, clients can use a typed-constructor.

*typed-value-constructor* ::=
*type-name* *left-paren value-expression* \[*comma* *value-expression*\]… *right-paren*

All members must be specified in the order they occur in the definition of the structured type being updated.

For example:

> UPDATE Employee SET FullName = FULLNAME('John','Smith')

Clients may update an individual member within a structured column by referencing that member in the set list of an update statement.

For example:

> UPDATE Employee SET Address.Country = ‘USA'

#### 3.10.4.1 Passing structured parameters at execute time

Structured parameters can be passed to the driver at execute time.

To pass a structured parameter at execute time:

1.  The application calls SQLBindParameter (or equivalent SQLDescField/SQLDescRec), specifying the following:
  - ValueType is `SQL_C_DEFAULT` (sets `SQL_DESC_TYPE`/`SQL_DESC_CONCISE_TYPE` in APD)
  - ParameterType is `SQL_ROW` or `SQL_UDT` (sets `SQL_DESC_TYPE`/`SQL_DESC_CONCISE_TYPE` in IPD)
  - For SQL_UDT, must set descriptor fields to specify:
     -   `USER_DEFINED_TYPE_CATALOG`
     -   `USER_DEFINED_TYPE_NAME`
     -   `USER_DEFINED_TYPE_SCHEMA`
  - ParameterValuePtr is set to a 32bit value identifying the parameter (passed back to the app from SQLParamData when data is needed for this parameter)
  -   `StrLen_or_IndPtr` is set to a buffer containing `SQL_DATA_AT_EXEC`
2.  The application calls SQLExecute or SQLExecDirect
    -   Driver returns `SQL_NEED_DATA`.
3.  The application calls SQLParamData to find out which param the driver is asking about.
    -   Driver returns `SQL_NEED_DATA` and passes back value from ParameterValuePtr.
4.  To specify a null or a default parameter value, the application calls SQLPutData with `str_len_or_ind` set to `SQL_NULL_DATA` or `SQL_DEFAULT_PARAM`, respectively, followed by SQLParamData to progress to the next parameter.
5.  For a non-null structured parameter, the application calls SQLGetNestedHandle for the specified parameter
    -   Driver returns a nested statement handle
6.  The application calls SQLBindParameter/SQLSetDescField to bind the nested parameters as named parameters.
    -   Individual property names are specified by setting the `SQL_DESC_NAME` attribute in the Implementation Parameter Descriptor (Driver sets `SQL_DESC_UNNAMED` to `SQL_NAMED`)
7.  The application sets the appropriate values in the buffer and calls SQLExecute on the nested handle to send the row
    -   Driver returns `SQL_NEED_DATA` if there are any data-at-execute parameters on the nested handle
8.  The application calls SQLParamData to indicate it has sent all of the data for the parameter
    -   If the parameter is a structured- or collection-valued parameter whose value was set through a nested statement handle, calling SQLParamData implicitly frees that handle
    -   Driver returns `SQL_NEED_DATA` if there are any remaining data-at-execute parameters.

### 3.10.5 Data Definition Extensions for Structured Columns

Structured types may be defined using the CREATE TYPE escape clause, as follows:

{ CREATE \[Open\] TYPE *type\_name* (*column-identifier data-type* \[*,column-identifier data-type*\]...) }

For example:

> {CREATE OPEN TYPE FULLNAME(Firstname varchar(25), LastName varchar (25))}

These types can be enumerated through the [SQLStructuredTypes](#63-sqlstructuredtypes) schema function. Their structure is described through [SQLStructuredTypeColumns](#64-sqlstructuredtypecolumns).

Structured columns can be created within a table using a named structural type or by using the ROW clause:

> CREATE TABLE Employees(FullName ROW(FirstName varchar(25),
LastName varchar(25)), Address ADDRESS)

## 3.11 Collection-valued Columns

ODBC adds support for collection-valued columns.

The ability to report collection-valued columns leverages the extensible type facility supported in ODBC 3.8 and greater. If a client has not called SQLSetEnvAttr with `SQL_ATTR_ODBC_VERSION` attribute set to `ODBC_OV_ODBC3_80` or greater, then the server must not return the `SQL_ARRAY` or `SQL_MULTISET` values in the `DATA_TYPE` columns or descriptor fields. In this case the server may either flatten the array values into multiple columns, create a join table containing the nested results, return the nested results as a string, or omit the nested results, as appropriate for the data source.

### 3.11.1 Schema Extensions for Collection-valued Columns

Collection-valued table columns are described in SQLColumns. Collection-valued result columns are described through SQLColAttribute(s)/SqlDescribeCol/SqlGetDescField.

Collection-valued columns may be ordered or unordered.

#### 3.11.1.1 Describing Collections using SQLColumns

For ordered array-valued columns, the value of the `DATA_TYPE` attributes is `SQL_ARRAY`. For unordered array-valued columns, the value of the `DATA_TYPE` attributes is `SQL_MULTISET`.

The remaining columns, including the `SQL_DATA_TYPE`, describe the type of the element within the collection.

Additionally, clients may get the type of a collection-valued column by passing the name of the collection-valued column appended with “[]” (repeated, for nested collections), as the column name in SQLColumns/SQLStructuredTypeColumns.

### 3.11.2 Query Extensions for Collections

The following query constructs are added in support of collections.

#### 3.11.2.1 Array Element Reference

Individual members of an array may be referenced using the *array-element-reference* expression.

*array-element-reference* ::=
*array-value-expression left-bracket numeric-value-expression right-bracket*

where *numeric-value-expression* yields an integer greater than zero and less than or equal to the number of elements in *array-value-expression*.

For example:

> Select EmpID, PhoneNumbers\[1\] from Employee

returns the employee ids along with their first phone number.

Since multisets are not ordered, their members cannot be individually referenced.

Note that some data sources may use a bracketed string following a field name in the select list as a shortcut for aliasing the field. For fields representing arrays, if the bracketed value is a number the driver must ensure that it is used as an index, not as an alias.

#### 3.11.2.2 UNNEST

The ANSI SQL *unnest-operator* expands the collection into multiple rows.

*unnest-operator* ::=
UNNEST *left-paren collection-value-expression right-paren*

The result of the *unnest-operator* is a derived table that can then be used with a correlation

For example:

> Select P.Name, Child.Name
> From People P, Unnest(P.Children) as Child

would return a row each child for each person containing the person’s name and child’s name.

If the type of the collection is variant, or the Unnest operator references a dynamic column of an open type, then the Unnest operator returns null for rows in which the column is not defined.

### 3.11.3 Response Extensions for Collection-valued Columns

To retrieve collection-valued result columns, the client specifies [`SQL_DESC_DATA_AT_FETCH`](#36-getting-data-during-fetch) for the column before calling SQLFetch or SQLFetchScroll. 

The driver returns `SQL_DATA_AVAILABLE` for non-null collection-valued columns, at which time the client can call [SQLGetNestedHandle](#62-sqlgetnestedhandle) to obtain a statement handle (HSTMT) on which the nested results can be described, bound, and fetched using the appropriate standard ODBC function calls. Collection-valued columns not specified as data-at-fetch are not returned by the driver. Attempting to return a collection-value for a bound column results in a type exception, which may be handled according to the value of [`SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`](#3731-sql_attr_type_exception_behavior).

For null collection values, the driver sets `SQL_DESC_INDICATOR_PTR` to `SQL_NULL_VALUE`. If `SQL_DESC_INDICATOR_PTR `is not specified for a collection typed column whose value is null, the driver returns an error with SQLState `22002`, Indicator variable required but not supplied.

For typed collections of scalar values, the nested statement handle contains a single descriptor record that describes the scalar value.

For typed collections of [structured values](#310-structured-columns), the nested statement handle describes the columns of the structured value.

For typed collections of [collections](#311-collection-valued-columns), the nested statement contains a single descriptor record on which the application calls [SQLGetNestedHandle](#62-sqlgetnestedhandle) in order to get a statement handle for each nested collection.

For collections of [variable typed values](#37-variable-typed-columns), the nested statement contains a single descriptor record describing the variable value. If the type of that value changes from a previous row, a type exception may be raised and handled according to the value of [`SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`](#3731-sql_attr_type_exception_behavior). For structured or collection-valued members within a variable-typed collection, the application calls [SQLGetNestedHandle](#62-sqlgetnestedhandle) in order to retrieve a statement handle on which to describe and retrieve the values of the structured or collection-valued member.

### 3.11.4 Write Extensions for Collection-valued Columns

Clients may specify the value of an array-valued column in an INSERT or UPDATE statement using the *array-value-constructor* defined as follows:

*array-value-constructor* ::=
ARRAY *left-paren value-expression* \[*comma* *value-expression*\]\* *right-paren*

*multiset-value-constructor* ::=
MULTISET *left-paren value-expression* \[*comma* *value-expression*\]\* *right-paren*

Multisets and arrays are treated as atomic values when updating. That is, the array or multiset can be replaced in whole, but individual members cannot be added, removed, or updated.

Clients may update an individual member within an array-valued value through the use of the *array-element-reference* expression.

### 3.11.5 Data Definition Extensions for Collection-valued Columns

Ordered collection-valued columns are defined using the array column type:

*array-column-type* ::= *data-type* ARRAY \[ *left-bracket maximum-cardinality right-bracket* \]

For example:

> CREATE TABLE Employee(FullName FULLNAME, Address ADDRESS, PhoneNumbers varchar(25) ARRAY\[10\] )

Unordered collection-valued columns are defined using the MULTISET column type :

*multiset-column-type* ::= *data-type* MULTISET

For example:

> CREATE TABLE Employee(FullName FULLNAME, Address ADDRESS, PhoneNumbers varchar(25) MULTISET )

#### 3.11.5.1 Passing collection-valued parameters at execute time

Collection-valued parameters can be passed to the driver at execute time.

To pass an array-valued parameter at execute time, the application:

1.  Application binds the parameter, specifying the following:
  - ValueType is `SQL_C_DEFAULT` (sets `SQL_DESC_TYPE`/`SQL_DESC_CONCISE_TYPE` in APD)
  - ParameterType is `SQL_ARRAY` or `SQL_MULTISET` (sets `SQL_DESC_TYPE`/`SQL_DESC_CONCISE_TYPE` in IPD)
     - Sets the `SQL_DESC_SUBTYPE` of the IPD to the type of the array or multiset, if known
     - For SQL_UDT sets descriptor fields to specify:
         - `USER_DEFINED_TYPE_CATALOG`
         - `USER_DEFINED_TYPE_SCHEMA`
         - `USER_DEFINED_TYPE_NAME`
  - ParameterValuePtr is set to a 32bit value identifying the parameter (passed back to the app from SQLParamData when data is needed for this parameter)
  - StrLen_or_IndPtr is set to a buffer containing `SQL_DATA_AT_EXEC`
2.  Application calls SQLPrepare/SQLExecute or SQLExecDirect
    -   Driver returns `SQL_NEED_DATA`.
3.  Application calls SQLParamData to find out which param the driver is asking about.
    -   Driver returns `SQL_NEED_DATA` and passes back value from ParameterValuePtr.
4.  To specify a null value, a default value, or an empty array, the application calls SQLPutData with `str_len_or_ind` set to `SQL_NULL_DATA`, `SQL_DEFAULT_PARAM`, or `SQL_EMPTY_ARRAY`, respectively, followed by SQLParamData to progress to the next parameter.
5.  To specify a non-empty array, the application calls GetNestedHandle for the specified parameter
    -   Driver returns a nested statement handle
6.  Application calls BindParameter/SetDescField to bind the nested columns.
    -   Individual column names are specified by setting the `SQL_DESC_NAME` attribute in the Implementation Parameter Descriptor.
7.  Application sets the appropriate values in the buffer and calls SQLExecute on the nested handle to send the row of data
    -   Driver returns `SQL_NEED_DATA` if there are any data-at-execute parameters on the nested handle
8.  Application repeats step 7 for each nested row
9.  Application calls SQLParamData to indicate it has sent all of the data for the parameter
    -   Driver returns `SQL_NEED_DATA` if there are any remaining data-at-execute parameters.

# 3.12 New ODBC C typedefs
New C-language typedefs are added for binding time values with fractional seconds.

# 3.12.1 New `SQL_C_TYPE_TIME_WITH_FRACTIONAL_SECONDS`
A new `SQL_C_TYPE_TIME_WITH_FRACTIONAL_SECONDS` is added for binding to a c time type that includes fractional seconds.

A corresponding new struct is added for returning time values with fractional seconds:

	typedef struct tagTIME_WITH_FRACTIONAL_SECONDS_STRUCT
	{
        SQLUSMALLINT   hour;
        SQLUSMALLINT   minute;
        SQLUSMALLINT   second;
        SQLUINTEGER    fraction;
	} TIME_WITH_FRACTIONAL_SECONDS_STRUCT;

4.x drivers must support binding to either the `SQL_C_TYPE_TIME` (without fractional seconds) or `SQL_C_TYPE_TIME_WITH_FRACTIONAL_SECONDS` (with fractional seconds).  4.0 clients should not specify 
`SQL_C_TYPE_TIME_WITH_FRACTIONAL_SECONDS` against a 3.x driver.

If an application specifies `SQL_C_DEFAULT` for a time value, then the driver must assume `SQL_C_TYPE_TIME` and not include fractional seconds.

# 3.12.2 New `SQL_C_TIME_WITH_TIMEZONE_WITH_FRACTIONAL_SECONDS`
A new `SQL_C_TYPE_TIME_WITH_TIMEZONE_WITH_FRACTIONAL_SECONDS` is added for binding to a c time with timezone type that includes fractional seconds.

A new struct is added for returning time with timezone values with fractional seconds:

	typedef struct tagTIME_WITH_TIMEZONE_FRACTIONAL_SECONDS_STRUCT
	{
		SQLUSMALLINT   hour;
		SQLUSMALLINT   minute;
		SQLUSMALLINT   second;
        SQLUINTEGER    fraction;
		SQLSMALLINT    timezone_hours;
		SQLUSMALLINT   timezone_minutes;
	} TIME_WITH_TIMEZONE_FRACTIONAL_SECONDS_STRUCT;

4.x drivers must support binding to either the `SQL_C_TYPE_TIME_WITH_TIMEZONE` (without fractional seconds) or `SQL_C_TYPE_TIME_WITH_TIMEZONE_WITH_FRACTIONAL_SECONDS` (with fractional seconds).  4.0 clients should not specify 
`SQL_C_TYPE_TIME_WITH_TIMEZONE_WITH_FRACTIONAL_SECONDS` against a 3.x driver.

If an application specifies `SQL_C_DEFAULT` for a time value, then the driver must assume `SQL_C_TYPE_TIME_WITH_TIMEZONE` and not include fractional seconds.

# 4 Change Tracking

##NOTE: This section of the specification is in a very early form; it is incomplete and subject to change or removal.

A number of data sources support the ability to determine changes from a certain point.

ODBC 4.0 provides common functions that services can use to expose this information.

ODBC adds a new *ODBC-supports-change-tracking-escape* that returns 1 if a given table supports change tracking, otherwise 0:

*ODBC-supports-change-tracking-escape* ::=
     *ODBC-esc-initiator* trackschanges(*tablename*) *ODBC-esc-terminator*

Where *tablename* is the unqualified, partially qualified, or fully qualified name of the table.

ODBC adds a new *ODBC-changeversion-escape* for getting the current change version (timestamp, version, or other high-water mark) defined as follows:

*ODBC-changeversion-escape* ::=
     *ODBC-esc-initiator* changeversion(*tablename*) *ODBC-esc-terminator*

The value returned by the *ODBC-changeversion-escape* is a binary value that can be passed to the *ODBC-changes-escape* to retrieve the latest values of any rows that have changed since the specified time:

*ODBC-changes-escape* ::=
     *ODBC-esc-initiator* changes(*tablename, changeversion*) *ODBC-esc-terminator*

The *ODBC-changes-escape* returns all rows from the specified *tablename* that have changed since the specified *changeversion*. The first column of the result is named “ColumnStatus” and contains the value “I” if the row is new since the specified changeversion, “U” if it has been updated since the specified changeversion, and “D” if it has been deleted since the specified change version. The additional columns of the result are the columns of the base table. Deleted rows must contain the “ColumnStatus” column, as well as any columns identified as `BEST_ROWID` and ROWVER from SQLSpecialColumns.

# 5 Compatibility
ODBC 4.0 drivers support ODBC 3.x clients by defaulting to a relational view for applications that specify an ODBC version of 2.x or 3.x through the `SQL_ATTR_ODBC_VERSION` environment attribute.

Certain ODBC 4.0 behavior may still available to an application that declares an ODBC version of 3.x, but it must be explicitly opted into by the application, for example, by setting new attributes.

## 5.1 ODBC 3.x Clients
Clients that specify a `SQL_ATTR_ODBC_VERSION` environment attribute value representing ODBC 2.x or 3.x can still use escape clauses, connection keywords, Infotypes, and attributes specified in this document, as supported by the driver. Where available, clients should query the corresponding InfoType to ensure support or be prepared for corresponding errors.

The Driver Manager raises `IM001`, Driver does not support this function, if a client attempts to specify data-at-fetch columns against drivers that report an ODBC 2.x or ODBC 3.x `ODBC_DRIVER_VERSION` InfoType. 

## 5.2 ODBC 3.x Drivers
ODBC 3.x drivers can support ODBC 4.0 escape clauses, connection keywords, infotypes, and attributes, as well as added columns to the catalog functions, without reporting ODBC 4.0 support, but must not utilize `SQL_VARIANT`, `SQL_ROW`, `SQL_UDT`, `SQL_ARRAY`, or `SQL_MULTISET` data types in schema functions or descriptor functions.

## 5.3 ODBC 4.0 Clients
ODBC 4.0 Clients must not utilize the new `SQL_C_TYPE_TIME_WITH_FRACTIONAL_SECONDS` or `SQL_C_TYPE_TIME_WITH_TIMEZONE_WITH_FRACTIONAL_SECONDS` types.

## 5.4 ODBC 4.0 Drivers
In order to report support for ODBC 4.0, a driver:

1. Must return `SQL_OV_ODBC4` from SQLGetInfo with `SQL_ODBC_DRIVER_VERSION` fInfoType
2. Must support data-at-fetch columns, and SQLNextColumn for retrieving the currently available column
3. Must support `SQL_ATTR_LENGTH_EXCEPTION_BEHAVIOR` to control how binary and string overflows are handled
4. Must support `SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR` to control how type exceptions are handled
5. Must support a `SQL_ATTR_DYNAMIC_COLUMNS` value of *False* to ignore dynamic columns
6. Must support binding `SQL_TIME` values to either the `SQL_C_TYPE_TIME` (without fractional seconds) or `SQL_C_TYPE_TIME_WITH_FRACTIONAL_SECONDS` (with fractional seconds)
7. Must support binding `SQL_TIME_WITH_TIMEZONE` values to either the `SQL_C_TYPE_TIME_WITH_TIMEZONE` (without fractional seconds) or `SQL_C_TYPE_TIME_WITH_TIMEZONE_WITH_FRACTIONAL_SECONDS` (with fractional seconds).
7. If `SQL_ATTR_ODBC_VERSION` environment attribute indicates an ODBC 2.x or 3.x version: 
	1. Must not utilize `SQL_VARIANT`, `SQL_ROW`, `SQL_UDT`, `SQL_ARRAY`, or `SQL_MULTISET` in schema or descriptor functions

In addition, if the driver supports `SQL_ROW`, `SQL_UDT`, `SQL_ARRAY`, or `SQL_MULTISET` columns, it must:

8. Support SQLGetNestedHandle, for retrieving a handle to read or write the nested value

In addition, if the driver supports `SQL_UDT`, it must:

9. Support SQLStructuredTypes
10. Support SQLStructuredTypeColumns

## 5.5 ODBC 4.0 Driver Manager
The ODBC 4.0 Driver Manager will map the following InfoTypes and attributes for 2.x and 3.x drivers:

| **InfoType/Attribute**               | **Behavior**                                               |
|--------------------------------------|------------------------------------------------------------|
| `SQL_SCHEMA_INFERENCE`               | If not supported by the driver, SQLGetInfo returns `FALSE` |
| `SQL_ATTR_DYNAMIC_COLUMNS`           | If not supported by the driver, SQLGetStmtAttr returns *False* and SQLSetStmtAttr  returns `SQL_SUCCESS_WITH_INFO` with a diagnostic code of 01S02.  |
| `SQL_ATTR_LENGTH_EXCEPTION_BEHAVIOR` | If not supported by the driver, SQLGetStmtAttr returns `SQL_LE_CONTINUE` and SQLSetStmtAttr with a value other than `SQL_LE_CONTINUE` returns `SQL_ERROR` with a diagnostic code of `HYC00`, Optional feature not implemented |
| `SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`   | If not supported by the driver, SQLGetStmtAttr returns SQL_TE_ERROR and SQLSetStmtAttr with a value other than `SQL_TE_ERROR` returns `SQL_ERROR` with a diagnostic code of `HYC00`, Optional feature not implemented  |

# 6 New Functions
The following functions are added in ODBC 4.0.

## 6.1 SQLNextColumn

The application calls SQLNextColumn in order to retrieve the column of data currently available to be read.

    SQLRETURN SQLNextColumn( 
      SQLHSTMT StatementHandle,
      SQLUSMALLINT* Col_or_Param_Num);

SQLNextColumn can first be called only after SQLFetch or SQLFetchScroll returns `SQL_DATA_AVAILABLE`, `SQL_METADATA_CHANGED`, or `SQL_MORE_DATA`, and as long as SQLNextColumn continues to return one of these values.

The Driver Manager returns `HY010`, Function Sequence Error under the following conditions:

1.  The specified StatementHandle was not in the executed state.

2.  SQLFetch or SQLFetschScroll have not been called on the executed statement.

3.  The most recent call to SQLFetch, SQLFetchScroll, or SQLNextColumn on this statement handle did not return `SQL_DATA_AVAILABLE`, `SQL_METADATA_CHANGED`, or `SQL_MORE_DATA`

4.  There is an asynchronously executing function called on this statement handle, or the connection associated with this statement handle, that has not completed.

The driver manager returns `HY010`, Function sequence error, from SQLSetPos if the most recent call to SQLFetch, SQLFetchScroll, or SQLNextColumn returned `SQL_DATA_AVAILABLE`.

### 6.1.1 Usage

While fetching a row that contains a column whose `SQL_DESC_DATA_AT_FETCH` descriptor is set to `SQL_TRUE`, when reading a dynamic column while [`SQL_ATTR_DYNAMIC_COLUMNS`](#391-SQL_ATTR_DYNAMIC_COLUMNS) is *True*, or when encountering a length or type exception, depending on the value of [`SQL_ATTR_LENGTH_EXCEPTION_BEHAVIOR`](#381-SQL_ATTR_LENGTH_EXCEPTION_BEHAVIOR) and [`SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`](#3731-SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR), respectively, the driver returns `SQL_DATA_AVAILABLE`, `SQL_METADATA_CHANGED`, or `SQL_MORE_DATA`, and the application calls SQLNextColumn in order to determine the ordinal of the next available column to be read.

The application can use the returned `Col_or_Param_Num` to retrieve information about the available column or parameter, but must not change descriptor information relative to the descriptor header record or any descriptor records not associated with the returned `Col_or_Param_Num`.

Once all columns have been processed, SQLNextColumn returns `SQL_SUCCESS` or `SQL_SUCCESS_WITH_INFO` with any valid SQLState from SQLFetch/SQLFetchScroll.

Calling SQLNextColumn with a null pointer for `Col_or_Param_Num` populates any remaining bound columns, skipping any data-at-fetch columns within the rowset. Dynamic columns are added to the descriptor as unbound columns and do not result in `SQL_DATA_AVAILABLE`. Any length exceptions are treated as `SQL_LE_CONTINUE`. If [`SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR`](#3731-SQL_ATTR_TYPE_EXCEPTION_BEHAVIOR) is `SQL_TE_ERROR` then an error is returned if any data values within the rowset are incompatible with their bound types, otherwise any type exceptions are treated as `SQL_TE_CONTINUE`.

Calling SQLFetch or SQLFetchScroll continues to the next rowset, adding any dynamic columns to the descriptor as unbound columns but otherwise skipping any data-at-fetch columns and ignoring type or length exceptions within the current rowset.
 
## 6.2 SQLGetNestedHandle

SQLGetNestedHandle is called in to retrieve a handle for reading or writing structured or collection-valued columns and parameters.

    SQLRETURN SQLGetNestedHandle(
      SQLHSTMT ParentStatementHandle,
      SQLUSMALLINT Col_or_Param_Num,
      SQLHSTMT* OutputChildStatementHandle);

SQLGetNestedHandle can be called for columns containing nested collections or structured columns once the ParentStatementHandle is in a prepared or executed state. The nested handle can be used to fetch data once the ParentStatementHandle is in the executed state and the driver returns `SQL_DATA_AVAILABLE` for the particular column, and may allow fetching data at other times depending upon `SQL_GETDATA_EXTENSIONS`.

The Driver Manager returns HY001 under the following condition:

1.  The Driver Manager was unable to allocate memory for the specified handle

The Driver Manager returns HY003, Program type out of range, under the following condition:

1.  The argument `Col_or_Param_Num` was 0.

The Driver Manager returns HY009, Invalid use of a null pointer under the following condition:

1.  The argument OutputChildStatementHandle was a null pointer.

The Driver Manager returns HY010, Function Sequence Error under the following conditions:

1.  The specified StatementHandle was not in a prepared or executed state.

2.  There is an asynchronously executing function called on this statement handle, or the connection associated with this statement handle, that has not completed.

The Driver Manager returns HY117, Connection is suspended due to unknown transaction state if the connection is in a suspended state.

The driver manager returns IM001, Driver does not support this function, under the following condition:

1.  The driver associated with StatementHandle does not support the function.

### 6.2.1 Usage

Applications call SQLGetNestedHandle in order obtain a nested statement handle representing rows or output parameters whose `SQL_DESC_TYPE` and `SQL_DESC_CONCISE_TYPE` descriptor fields indicate "SQL_UDT", "SQL_ROW", "SQL_ARRAY", or "SQL_MULTISET".

Nested statement handles can be used to describe results once the parent statement handle is in the prepared or executed state. Applications can fetch data from the nested statement handle once the driver returns `SQL_DATA_AVAILABLE` for the particular column, and may allow fetching data at other times depending upon `SQL_GETDATA_EXTENSIONS`.

Fetching results from a handle returned by SQLGetNestedHandle has the same sequencing restrictions as SQLGetData. Drivers report relaxed sequencing rules for both SQLGetData and fetching results from a nested statement handle through `SQL_GET_DATA_EXTENSIONS`. Unless `SQL_GET_DATA_EXTENSIONS` specifies `SQL_GD_ANY_ORDER`, Fetch/GetData must only be called on the handle associated with the most recent call to SQLNextColumn.

After calling SQLGetNestedHandle, the application can use the returned ChildStatementHandle to describe and fetch results for that nested content.

Calling SQLCloseCursor on the nested handle closes the child cursor for the current parent row.

Multiple calls to SQLGetNestedHandle for the same `ParentStatementHandle`, `Col_or_Param_Num`, and `SQL_DESC_CONCISE_TYPE` return the same ChildStatementHandle (including any added/modified descriptor fields) unless that handle has been freed, in which case the next call to SQLGetNestedHandle for that `ParentStatementHandle`, `Col_or_Param_Num`, and `SQL_DESC_CONCISE_TYPE` combination returns a fresh statement handle with default values appropriate for the current row.

Unless the [SQL_GD_CONCURRENT](#361-SQL_GD_CONCURRENT-bit-for-sql_getdata_extensions) bit within SQL_GETDATA_EXTENSIONS is specified, a subsequent call to retrieve data using SQLGetData, SQLFetch, SQLFetchScroll, or SQLNextColumn on the parent handle or a different nested statement handle implicitly closes the statement handle. If `SQL_GD_CONCURRENT` is specified, a subsequent call to retrieve data on the child statement handle does not affect any other statement handle.

Closing a statement handle implicitly closes and frees all child statement handles.

Calling SQLGetNestedHandle for a non-structured or collection valued column, or calling SQLGetData on a structured or collection-valued column, returns `SQL_ERROR` with SQLState `07009`, Invalid descriptor index.

TODO: describe output parameters. Do we need a new InputOutputType, or do we use the existing `SQL_PARAM_[INPUT_]OUTPUT_STREAM`?

## 6.3 SQLStructuredTypes

SQLStructuredTypes enumerates named structural types.

    SQLRETURN SQLStructuredTypes(
      SQLHSTMT StatementHandle,
      SQLCHAR * CatalogName,
      SQLSMALLINT NameLength1,
      SQLCHAR * SchemaName,
      SQLSMALLINT NameLength2,
      SQLCHAR * TypeName,
      SQLSMALLINT NameLength3);

The driver manager enforces the same sequencing and validity checks as in SQLTables. In particular:

The Driver Manager returns `HY010`, Function Sequence Error under the following conditions:

1.  There is an asynchronously executing function called on this statement handle, or the connection associated with this statement handle, that has not completed.

2.  **SQLExecute**, **SQLExecDirect**, or **SQLMoreResults** was called for the *StatementHandle* and returned `SQL_PARAM_DATA_AVAILABLE`. This function was called before data was retrieved for all streamed parameters

3.  **SQLExecute**, **SQLExecDirect**, **SQLBulkOperations**, or **SQLSetPos** was called for the *StatementHandle* and returned `SQL_NEED_DATA`. This function was called before data was sent for all data-at-execution parameters or columns.

The Driver Manager returns `HY090` under the following conditions:

1.  The value of one of the length arguments was less than 0 but not equal to `SQL_NTS`.

2.  The value of one of the name length arguments exceeded the maximum length value for the corresponding name.

The Driver Manager returns `HY117`, Connection is suspended due to unknown transaction state, if the connection is in a suspended state.

The driver manager returns `IM001`, Driver does not support this function, under the following condition:

1.  The driver associated with StatementHandle does not support the function.

### 6.3.1 Usage

The result of SQLStructuredTypes mirrors the result of SQLTables:

| **Column name** | **Column number** | **Data type** | **Comments**                                                              |
|-----------------|-------------------|---------------|---------------------------------------------------------------------------|
| `UDT_CAT`        | 1                 | Varchar       | Catalog name. See `TABLE_CAT` in SQLTables.                                |
| `UDT_SCHEM`      | 2                 | Varchar       | Schema name See `TABLE_SCHEM` in SQLTables.                                |
| `UDT_NAME`       | 3                 | Varchar       | Structured Type name.                                                     |
| `UDT_TYPE`       | 4                 | Varchar       | UDT Type name; "TYPE", "`OPEN TYPE`", or a data source–specific type name. |
| REMARKS         | 5                 | Varchar       | A description of the Structured Type.                                     |

# 6.4 SQLStructuredTypeColumns

SQLStructuredTypeColumns describes the columns of a named structural type.

    SQLRETURN SQLStructuredTypeColumns(
      SQLHSTMT StatementHandle,
      SQLCHAR * CatalogName,
      SQLSMALLINT NameLength1,
      SQLCHAR * SchemaName,
      SQLSMALLINT NameLength2,
      SQLCHAR * TypeName,
      SQLSMALLINT NameLength3,
      SQLCHAR * ColumnName,
      SQLSMALLINT NameLength4);

The result of SQLStructuredTypeColumns mirrors the result of SQLColumns:

| **Column name**     | **Column number** | **Data type**     | **Comments**        |
|---------------------|---------------|---------------|---------------------------------------------------------|
| `UDT_CAT`            | 1                 | Varchar           | Catalog name; NULL if not applicable to the data source.                                                                         |
| `UDT_SCHEM`          | 2                 | Varchar           | Schema name; NULL if not applicable to the data source.                                                                          |
| `UDT_NAME`           | 3                 | Varchar not NULL  | Structured Type name.                                                                                                            |
| `COLUMN_NAME`        | 4                 | Varchar not NULL  | Column name. The driver returns an empty string for a column that does not have a name.                                          |
| `DATA_TYPE`          | 5                 | Smallint not NULL | SQL data type. See `DATA_TYPE` column in SQLColumns.                                                                              |
| `TYPE_NAME`          | 6                 | Varchar not NULL  | Data source–dependent data type name                                                                                             |
| `COLUMN_SIZE`        | 7                 | Integer           | Size of the column. See `COLUMN_SIZE` column in SQLColumns.                                                                       |
| `BUFFER_LENGTH`      | 8                 | Integer           | The length in bytes of data transferred on an SQLGetData, SQLFetch, or SQLFetchScroll operation if `SQL_C_DEFAULT` is specified. |
| `DECIMAL_DIGITS`     | 9                 | Smallint          | See `DECIMAL_DIGITS` in SQLColumns.                                                                                               |
| `NUM_PREC_RADIX`     | 10                | Smallint          | See NUM_PREC_RADIX in SQLColumns.                                                                                              |
| `NULLABLE`           | 11                | Smallint not NULL | See `NULLABLE` in SQLColumns.                                                                                                      |
| `REMARKS`            | 12                | Varchar           | A description of the column.                                                                                                     |
| `COLUMN_DEF`         | 13                | Varchar           | The default value of the column. See the `COLUMN_DEF` column in SQLColumns.                                                       |
| `SQL_DATA_TYPE`      | 14                | Smallint not NULL | See `SQL_DATA_TYPE` in SQLColumns.                                                                                               |
| `SQL_DATETIME_SUB`   | 15                | Smallint          | See `SQL_DATETIME_SUB` in SQLColumns.                                                                                            |
| `CHAR_OCTET_LENGTH`  | 16                | Integer           | The maximum length in bytes of a character or binary data type column.                                                           |
| `ORDINAL_POSITION`   | 17                | Integer not NULL  | The ordinal position of the column in the structured type. The first column in the UDT is number 1.                              |
| `IS_NULLABLE`        | 18                | Varchar           | See `IS_NULLABLE` in SQLColumns.                                                                                                  |
| `CHAR_SET_CAT`       | 19                | Varchar           | Catalog of the Character set for the column, or null.                                                                            |
| `CHAR_SET_SCHEM`     | 20                | Varchar           | Schema of the Character set for the column, or null.                                                                             |
| `CHAR_SET_NAME`      | 21                | Varchar           | Character set name for the column, or null.                                                                                      |
| `COLLATION_CAT`      | 22                | Varchar           | Catalog of the Collation for the column, or null.                                                                                |
| `COLLATION_SCHEM`    | 23                | Varchar           | Schema of the Collation for the column, or null.                                                                                 |
| `COLLATION_NAME`     | 24                | Varchar           | Name of the Collation for the column, or null.                                                                                   |
| `UDT_CAT`            | 25                | Varchar           | Catalog of the structured type if the column is a UDT, otherwise null.                                                           |
| `UDT_SCHEM`          | 26                | Varchar           | Schema of the structured type if the column is a UDT, otherwise null.                                                            |
| `UDT_NAME`           | 27                | Varchar           | Name of the structured type if the column is a UDT, otherwise null.                                                              |
