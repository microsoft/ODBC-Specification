ODBC 4.0
=

#1 Overview

ODBC is a widely adopted, standard, client-side API that works well for relational (i.e., tabular) data. However, many types of interesting data may not fit well into a relational model. In particular:

1.  Semi-structured data – Tables whose schema may not be defined or may change on a row-by-row basis

2.  Hierarchical Data – Data with nested structure (structured fields, lists)

3.  Multi-dimensional data – Data representing aggregated measures across dimensions

ODBC is extended to support non-relational concepts such as:

1.  Dynamic columns (columns not defined in schema)

2.  Structured columns

3.  Collection-valued columns

4.  Untyped/Varying typed columns

In addition, general enhancements are added in order to improve interoperability of generic ODBC clients across diverse data sources, including enhanced discovery, authentication, syntax, and capability reporting.

There are four aspects to consider in order to support non-relational data:

1.  **Describing** data that doesn’t fit into flat, rectangular relational tables

2.  **Querying** non-relational data

3.  **Representing** (binding) non-relational results

4.  **Updating** non-relational data

#2 Compatibility with ODBC 3.x

ODBC 3.x is a relational API. Existing clients expect data to be modelled, queried, represented and updated relationally.

In order to be advertised as an ODBC 3.x driver, and returned from the control panel/driver manager, drivers must be 100% compatible with ODBC 3.x. Such drivers must expose a relational view of the data through the standard schema functions (SQLTables, SQLColumns, etc.) and continue to support standard Entry SQL-92 operations against this flattened view.

The flattened view may not be comprehensive (there may be data not available through the relational view, data modeled as strings representing structured content of a particular format, the data may not be updatable through that view, etc.) but existing ODBC 3.x clients must be able to meaningfully interact with the data.

The convenience to existing clients of representing hierarchical data through a relational lens sacrifices fidelity. Applications that specify a value for the SQL\_ATTR\_ODBC\_VERSION environment attribute of 4.x or greater may see a hierarchical structure containing row, collection, and untyped data values.

#3 Design

ODBC 4.0 is extended to support common features across all types of data sources, as well as specific new features to support new schema concepts.

#3.1 Private Drivers
---------------

Drivers registered through the ODBC setup utility are available to all applications. All applications using the driver use the same version of the driver.

ODBC 4.0 enables applications to use their own private drivers. Such drivers are not visible to other applications and may support different versions from applications using the same driver.

Private drivers are not registered using the ODBC setup utility. The Driver Manager loads private drivers by looking in the application’s install directory for a folder named “ODBCDrivers”. Each private driver is represented by a subfolder with the same name as *driver-name*. This folder must contain a .ini file whose name is the *driver-name* with the “.ini” suffix.

The .ini file follows the [windows .ini file format](https://en.wikipedia.org/wiki/INI_file). It is a text file whose first line is a section header containing the name of the driver, enclosed in square brackets, and followed by keyword=value pairs, one per line. Valid keyword-value pairs are the [subkeys](https://msdn.microsoft.com/en-us/library/ms709431(v=vs.85).aspx) allowed for the driver specific section under ODBCINST.INI, along with a new “ApplicationKey” key-value pair which, if specified, is passed by the driver manager to the driver in a call to SetEnvAttr with the new SQL\_ATTR\_APPLICATION\_KEY attribute. The driver developer can use this to validate that the calling application is permitted to use the driver, for example, by using a signed hash of the application’s publisher validated against the calling application at runtime.

SQLDrivers enumerates the drivers registered under the ODBCINST.INI registry key as well as the drivers bin-deployed within the current application’s install directory as described above. The *DriverAttributes* returned by SQLDrivers for a bin-deployed driver is the set of key-value pairs read from the corresponding .ini file.

This expanded set of drivers is also available in the data source configuration dialog for creating new DSNs.

SQLDataSources filters the set of DSNs based on the underlying driver, enumerating only those data sources whose driver is available to the application. This also restricts the list of available data sources that are enumerated in the data source connection dialog.

When resolving a driver for a registered DSN, the Driver Manager first looks to the Driver keyword within the named DSN that is a direct child of the ODBC.INI subkey. If this string matches the name of a directory under the current “ODBC Drivers” directory, then the driver is loaded according to the .ini file defined in that directory. Otherwise, the Driver Manager looks for a registry entry under the “Drivers” section within ODBCINST.INI. Failing that, the Driver Manager treats the string as a file path and attempts to load the driver using that path.

This allows bin-deployed drivers to work like any other driver from the application’s perspective, including the creation of DSNs, and allows DSNs to be shared across applications that each have their own copy of the driver.

##3.2 Web Authorization

Applications can connect to drivers that require web-based authentication through SQLDriverConnect or SQLBrowseConnect.

###3.2.1 Web Authorization Flow with SQLDriverConnect

Applications can connect to services requiring web authentication by calling SQLDriverConnect.

If the input connection string contains the required information (for example, access token, scope, refresh token, and expiration, as appropriate) in addition to any driver-specific properties, and the access token has not expired, no additional information should be required. If a provided access token has expired and the connection string contains a refresh token, the driver attempts to refresh the connection using the supplied refresh token. If the access token has expired and the connection string does not contain a refresh token, SQLDriverConnect fails as described in [Token Expiration](#new-connection-keywords).

Applications that don’t have a complete connection string, but are willing to allow drivers to pop up dialogs to obtain necessary information from the user, can call SQLDriverConnect, specifying a DriverCompletion value of SQL\_DRIVER\_PROMPT, SQL\_DRIVER\_COMPLETE, or SQL\_DRIVER\_COMPLETE\_REQUIRED.

When SQLDriverConnect is called with a DriverCompletion value other than SQL\_DRIVER\_NOPROMPT, drivers that require web authentication not present in the input connection string conduct the necessary authentication flow, including presenting browser windows as necessary, and return to the application a completed connection string containing driver-specific properties, an access token, scope (as appropriate) and, if the access token has an expiration, the expiration (as an ISO 8601 datetime with timezone) and refresh token:

> "Auth\_AccessToken=xxx;Auth\_Expires=xxx;Auth\_RefreshToken=xxx;
> Auth\_Scope=ZZZ;Prop1=xxx; "

This connection string contains all the information necessary to be used in a subsequent call to SQLDriverConnect or SQLBrowseConnect to reconnect to the service. Note that this connection string may contain sensitive information and should be protected by the application.

###3.2.2 Web-based Authentication Flow with SQLBrowseConnect

Applications that are unable to allow drivers to pop up dialogs can call SQLBrowseConnect to connect to the service.

SQLBrowseConnect provides an iterative dialog between the driver and the application where the application passes in an initial input connection string. If the connection string contains sufficient information to connect, the driver responds with SQL\_SUCCESS and an output connection string containing the complete set of connection attributes used.

If the initial input connection string does not contain sufficient information to connect to the source, the driver responds with SQL\_NEED\_DATA and an output connection string specifying informational attributes for the application (such as the authorization url) as well as required attributes to be specified in a subsequent call to SQLBrowseConnect. Attribute names returned by SQLBrowseConnect may include a colon followed by a localized identifier, and the value of the requested attribute is either a single question mark or a comma-separated list of valid values (optionally including localized identifiers) enclosed in curly braces. Optional attributes are returned preceded with an asterix (\*).

In a Web-based authentication scenario, if SQLBrowseConnect is called with an input connection string containing an access token that has not expired, along with any other required properties, no additional information should be required. If the access token has expired and the connection string contains a refresh token, the driver attempts to refresh the connection using the supplied refresh token.

If SQLBrowseConnect is called without an access token, with an expired access token and the driver is unable to refresh the token, or with other required information missing, SQLBrowseConnect returns with SQL\_NEED\_DATA.

In cases where different authentication methods may be supported, the driver can initially return an OutConnectionString allowing the application to select from an enumerated set of authentication methods.

> "Auth\_Type={OAuth\_1.0, OAuth\_2.0}"

Common keywords for Auth\_Type include “Basic”, “Integrated”, “OAuth\_1.0”, “OAuth\_2.0”, and “None”. Drivers should use these keywords for the corresponding authentication types, where supported, and may specify other driver-specific authentication types.

In order to specify an authentication method, the application calls SQLBrowseConnect with a collection string specifying one of those allowed authentication methods, and SQLBrowseConnect returns an output connection string requesting the connection attributes required for that particular choice.

For OAuth 2.0, SQLBrowseConnect returns an OutConnectionString requesting a redirect uri, along with a client ID, a client secret, and a scope, as required, along with any required provider-specific properties.

> "Auth\_BaseRedirectUri:Redirect Uri:?;Auth\_Client\_ID:ClientID=?;
> Auth\_Client\_Secret:Client Secret=?;\*Auth\_Scope:Scope={XXX:xxx,YYY:yyy,ZZZ:zzz}"

If the service supports scopes but the driver is unable to enumerate those scopes, it simply returns a question mark (?) as the value for Auth\_Scope in the output connection string:

> "Auth\_BaseRedirectUri:Redirect Uri:?;Auth\_Client\_ID:ClientID=?;
> Auth\_Client\_Secret:Client Secret=?;\*Auth\_Scope:Scope=?"

For an OAuth 1.0, the output connection string would also ask for Auth\_Realm and Auth\_Token\_Secret.

> "Auth\_BaseRedirectUri:Redirect Uri:?;Auth\_Client\_ID:ClientID=?;Auth\_Client\_Secret:Client Secret=?;\*Auth\_Scope:Scope=?;Auth\_Realm:Realm=?;Auth\_Token\_Secret:Token Secret=?"

The application next calls SQLBrowseConnect with a connection string containing the redirect uri, the client ID, and client secret, along with scope, realm, and token secret, as appropriate:

> "Auth\_BaseRedirectUri=https://abc.com/auth;Auth\_Client\_ID=myapp;Auth\_Client\_Secret=xyz;
> Auth\_Scope=ZZZ"

The driver responds with a connection string containing the authorization url and requesting the final redirect uri. For the best client UI experience, the output connection string should also include suggested window height and width (in pixels) for the browser window:

> “AuthorizationUrl=xxx; Auth\_WindowHeight=xxx;Auth\_WindowWidth=xxx;
> Auth\_CompletedRedirectUri=?”

The application navigates the browser to the Url specified by the authorization url. Auth flow is complete once the browser reaches the redirect uri.

At that point, the application calls SQLBrowseConnect again, supplying the completed redirect uri (including any query string parameters). The driver uses the query string parameters returned in this uri to complete any additional flow required and returns SQL\_SUCCESS with an OutConnectionString containing driver-specific properties, an access token, and scope, expiration (as an ISO 8601 datetime with timezone), and refresh token as appropriate:

> "Auth\_AccessToken=xxx;Auth\_Expires=xxx;Auth\_RefreshToken=xxx;
> Auth\_Scope=ZZZ;Prop1=xxx; "

This connection string contains all the information necessary to be used in a subsequent call to SQLDriverConnect or SQLBrowseConnect to reconnect to the service. Note that this connection string may contain sensitive information and should be protected by the application.

###3.2.3 New Connection Keywords

ODBC 4.0 defines the following new Connection string key/value pairs:

####3.2.3.1 Auth\_Type

Auth\_Type defines common types of authentication. Drivers that support multiple types of authentication can request Auth\_Type in the output connection string returned by BrowseConnect in order to allow the client to select from an enumerated list of authentication mechanisms.

While the driver can return custom values, the following common values should be used where applicable:


 **Keyword** | **Description** 
-------------|---------------
 None        | No authentication is required; the driver supports anonymous access
 Basic       | Basic authentication using User ID (UID) and Password (PWD) is supported by the driver
 Trusted     | The driver supports connecting using trusted (i.e., Kerberos or NTLM) Authentication.
 OAuth\_1.0  | The driver supports an OAuth 1.0 authentication flow, as described in [Web Authorization](#web-authorization).
 OAuth\_2.0  | The driver supports an OAuth 2.0 authentication flow, as described in [Web Authorization](#web-authorization)
 LDAP        | The driver supports authentication through LDAP 

####3.2.3.2 Auth\_BaseRedirectUri

In a [Web Authorization Flow](#web-authorization), the driver may request the application to provide a Uri to be called when the authentication completes. This is typically a URI to an endpoint supported by the application. The driver will construct an [AuthorizationUrl](#authorizationurl) using this callback URL.

####3.2.3.3 AuthorizationUrl

In a [Web Authorization Flow](#web-authorization), the driver may provide an AuthorizationUrl that the application must invoke in order to initiate the authorization.

####3.2.3.4 Auth\_CompletedRedirectUri 

In a [Web Authorization Flow](#web-authorization), the driver may ask for a CompletedRedirectUri that is the result of invoking the AuthorizationUrl. This CompletedRedirectUri will generally contain information, i.e. through query string parameters, used by the driver to complete the authorization flow.

####3.2.3.5 Auth\_Client\_ID, Auth\_Client\_Secret

In a [Web Authorization Flow](#web-authorization), the driver may request a Client ID and Client Secret from the application. This information is typically used to construct the [AuthorizationUrl](#authorizationurl) used to initiate authorization.

####3.2.3.6 Auth\_Scope

In a [Web Authorization Flow](#web-authorization), the driver may request a Scope from the application. The driver may provide an enumerated list of known scopes that the application can pick from, or may allow the application to provide an arbitrary scope. This information is typically used to construct the [AuthorizationUrl](#authorizationurl) used to initiate authorization.

####3.2.3.7 Auth\_Realm

In a [OAuth 1.0 Authorization Flow](#web-authorization), the driver may request a Realm from the application. This information is typically used to construct the [AuthorizationUrl](#authorizationurl) used to initiate authorization.

####3.2.3.8 Auth\_Token\_Secret.

In a [OAuth 1.0 Authorization Flow](#web-authorization), the driver may request a Secret from the application. This information is typically used to construct the [AuthorizationUrl](#authorizationurl) used to initiate authorization.

####3.2.3.9 Auth\_WindowHeight, Auth\_WindowWidth

In a [Web Authorization Flow](#web-authorization), the application may have to provide a window (i.e., through a browser) to collect input from the user. The driver may provide WindowHeight and WindowWidth keywords in the output query string, for example along with the [AuthorizationUrl](#authorizationurl), as hints to the application as to the size of the window to present to the user.

###3.2.4 Token Expiration

Applications can determine whether or not the access token has expired by calling SQLGetConnectAttr with the SQL\_ATTR\_CONNECTION\_DEAD attribute. As this connection attribute is also used by the Driver Manager in connection pooling, getting this attribute is performance critical and the driver SHOULD NOT make a round trip to determine the status of the connection, but should return SQL\_CD\_TRUE if it has previously determined that the connection is dead.

If [SQL\_ATTR\_REFRESH\_CONNECTION](#sql_attr_-credentials) is SQL\_REFRESH\_MANUAL, or the driver is unable to refresh the token, attempting to call an operation that requires a connection after the access token has expired returns SQL\_ERROR with a SQLState value of 08006 (Connection Failure). Applications can attempt to refresh the connection by calling SQLSetConnectAttr with [SQL\_ATTR\_REFRESH\_CONNECTION](#sql-syntax-support) and, if successful, retry the failed operation. If refreshing the connection fails, the application must close the connection (resulting in any dependent statement or descriptor handles being implicitly freed) and start over.

Issue: should we define more specific subcodes/diag fields for expired token? Note that, in many cases, the service won’t know that the connection issue was an expired token versus some other connection failure, so we’d still have to allow fall-back to the general ConnectionFailure.

###3.2.5 Token Refresh

Access tokens can be refreshed automatically by the driver, or manually by the application.

####3.2.5.1 SQL\_ATTR\_ CREDENTIALS

Applications can retrieve the current connection information, including (for web authorization scenarios) the current access token, along with refresh token and expiration, as appropriate, by calling SQLGetConnectAttr for the new SQL\_ATTR\_ CREDENTIALS connection attribute.

Applications can update credentials by calling SQLSetConnectAttr for SQL\_ATTR\_CREDENTIALS and specifying the value retrieved in a previous call to GetConnectionAttr for SQL\_ATTR\_ CREDENTIALS to the same data source.

This information returned by SQL\_ATTR\_CREDENTIALS is an encoding of the necessary connection information sufficient to create a new connection to the same source by passing to SQLDriverConnect or SQLBrowseConnect, or enable refreshing credentials for the same source by calling SQLSetConnectAttr with the new SQL\_ATTR\_ CREDENTIALS connection attribute.

Note that, while drivers should encode sensitive information within the string, applications should treat the credentials as sensitive information.

####3.2.5.2 SQL\_ATTR\_REFRESH\_CONNECTION

A new connection attribute, SQL\_ATTR\_ REFRESH\_CONNECTION, is added to allow applications to control how and when drivers refresh a connection.

If SQL\_ATTR\_ REFRESH\_CONNECTION is SQL\_REFRESH\_AUTO (the default), the driver attempts to automatically refresh connections (i.e., access token) as required, using the current value of SQL\_ATTR\_ CREDENTIALS. Upon successful refresh, the value of SQL\_ATTR\_ CREDENTIALS is updated with the new credentials (i.e., access token, and refresh token and expiration, as appropriate.)

If SQL\_ATTR\_CONNECTION is set to SQL\_REFRESH\_NOW, the driver attempts to refresh the connection (i.e., access token) using the current value of SQL\_ATTR\_ CREDENTIALS. Upon successful refresh, the value of SQL\_ATTR\_ CREDENTIALS is updated with the new credentials (i.e., access token, and refresh token and expiration, as appropriate.) Setting SQL\_ATTR\_CONNECTION to SQL\_REFRESH\_NOW does an immediate refresh of the connection but does not change the persisted value of connection option. Drivers may, but are not required to, return SQL\_SUCCESS\_WITH\_INFO and 01S02, Optional Value Changed, since the value of SQL\_ATTR\_REFRESCH\_CONNECTION is not actually changed to the specified value.

If SQL\_ATTR\_ REFRESH\_CONNECTION is set to SQL\_REFRESH\_MANUAL, the driver should not attempt to automatically refresh access tokens.

##3.3 SQL Syntax Support

ODBC is designed as an API to provide a common way to connect to, describe, execute a command, and retrieve results from a relational store. ODBC relies on ANSI/ISO SQL to provide a common grammar across relational stores, and supplies a way to report levels of conformance and support for optional syntax elements.

###3.3.1 SQL Compliance

Drivers that support ANSI SQL syntax must follow SQL semantics for that syntax or fail the request.

###3.3.2 String escaping

> As per ANSI, a single-quote character within a string literal is escaped by a single-quote character. Drivers must translate double-single quotes to whatever is the native syntax for an escaped single quote.

###3.3.3 Nesting Escape Clauses

Drivers should support nesting of escape clauses, as appropriate (for example, using an escape clause within an expression passed to a scalar function).

###3.3.4 SQL Syntax Capabilities

ODBC Drivers report supported SQL syntax through SQLGetInfo. The following SQLGetInfo information types and values are added to help drivers describe support for additional SQL constructs

####3.3.4.1 Binary Functions

ODBC 4.0 drivers may support the [canonical string functions](https://msdn.microsoft.com/en-us/library/ms710249(v=vs.85).aspx) against binary strings.

#####3.3.4.1.1 SQL\_BINARY\_FUNCTIONS

A new SQLGetInfo *information type,* SQL\_BINARY\_FUNCTIONS, is added to allow a server to report which of the string functions can be used with binary strings.

The following new bitmask values may be returned from SQL\_BINARY\_FUNCTIONS:

SQL\_FN\_BIN\_BIT\_LENGTH, SQL\_FN\_BIN\_CONCAT, SQL\_FN\_BIN\_INSERT, SQL\_FN\_BIN\_LTRIM, SQL\_FN\_BIN\_OCTET\_LENGTH, SQL\_FN\_BIN\_POSITION, SQL\_FN\_BIN\_RTRIM, SQL\_FN\_BIN\_SUBSTRING

Note: Each of these is defined with the same value as their existing SQL\_FN\_STR counterpart.

#####3.3.4.1.2 SQL\_ISO\_BINARY\_FUNCTIONS

A new GetInfo *information type*, SQL\_ISO\_BINARY\_FUNCTIONS, is added to allow a server to report which of the ANSI SQL Binary functions can be used.

The following bitmasks are used to determine which string functions are supported. Their values are the same as their SQL\_SFF counterparts for SQL\_SQLSTRING\_FUNCTIONS.

SQL\_SBF\_CONVERT, SQL\_SBF\_SUBSTRING, SQL\_SBF\_TRIM\_BOTH, SQL\_SBF\_TRIM\_LEADING, SQL\_SBF\_TRIM\_TRAILING, SQL\_SBF\_OVERLAY, SQL\_SBF\_LENGTH, SQL\_SBF\_POSITION, SQL\_SBF\_CONCAT

####3.3.4.2 SQL\_ISO\_STRING\_FUNCTIONS

A new SQLGetInfo *information type*, SQL\_ISO\_STRING\_FUNCTIONS, is added to allow a server to report which of the ANSI SQL String functions can be used. Its value is equal to the existing SQL\_SQL92\_STRING\_FUNCTIONS. Values are the existing SQL\_SSF values from SQL\_SQL92\_STRING\_FUNCTIONS, plus the following values:

SQL\_SSF\_OVERLAY, SQL\_SSF\_LENGTH, SQL\_SSF\_POSITION, SQL\_SSF\_CONCAT

####3.3.4.3 SQL\_SQL92 Equivalents

The following SQLGetInfo *information type*s are defined equivalent to existing SQL\_SQL92 defines, but not specific to SQL92:

> SQL\_ISO\_DATETIME\_FUNCTIONS = SQL\_SQL92\_DATETIME\_FUNCTIONS
>
> SQL\_ISO\_FOREIGN\_KEY\_DELETE\_RULE = SQL\_SQL92\_FOREIGN\_KEY\_DELETE\_RULE
>
> SQL\_ISO\_FOREIGN\_KEY\_UPDATE\_RULE = SQL\_SQL92\_FOREIGN\_KEY\_UPDATE\_RULE
>
> SQL\_ISO\_GRANT = SQL\_SQL92\_GRANT
>
> SQL\_ISO\_NUMERIC\_VALUE\_FUNCTIONS = SQL\_SQL92\_NUMERIC\_VALUE\_FUNCTIONS
>
> SQL\_ISO\_PREDICATES = SQL\_SQL92\_PREDICATES
>
> SQL\_ISO\_RELATIONAL\_JOIN\_OPERATORS = SQL\_SQL92\_RELATIONAL\_JOIN\_OPERATORS
>
> SQL\_ISO\_REVOKE = SQL\_SQL92\_REVOKE
>
> SQL\_ISO\_ROW\_VALUE\_CONSTRUCTOR = SQL\_SQL92\_ROW\_VALUE\_CONSTRUCTOR
>
> SQL\_ISO\_VALUE\_EXPRESSIONS = SQL\_SQL92\_VALUE\_EXPRESSIONS

####3.3.4.4  Added SQL\_AGGREGATION\_FUNCTIONS bitmasks

The following bitmasks are added to SQL\_AGGREGATION\_FUNCTIONS to advertise support for additional ANSI SQL-defined aggregation functions:

| **BitMask**           | **ANSI SQL Function** |
|-----------------------|-----------------------|
| SQL\_AF\_EVERY        | EVERY                 |
| SQL\_AF\_ANY          | ANY                   |
| SQL\_AF\_STDEV\_OP    | STDEV\_OP             |
| SQL\_AF\_STDEV\_SAMP  | STDEV\_SAMP           |
| SQL\_AF\_VAR\_SAMP    | VAR\_SAMP             |
| SQL\_AF\_VAR\_POP     | VAR\_POP              |
| SQL\_AF\_ARRAY\_AGG   | ARRAY\_AGG            |
| SQL\_AF\_COLLECT      | COLLECT               |
| SQL\_AF\_FUSION       | FUSION                |
| SQL\_AF\_INTERSECTION | INTERSECTION          |

The value of SQL\_AF\_EVERY is a synonym for the current SQL\_AF\_ALL.

####3.3.4.5 Added Scalar Function bitmasks

Todo…

###3.3.5 Language Extensions

For common constructs that are not part of SQL-92 (i.e., outer joins, function execution, datetime literals), ODBC defines escape clauses as a way to embed functionality within native commands that can easily be scanned, tweaked, and then passed-through to the back-end. These escape clauses provide a level of interoperability for basic operations while allowing clients familiar with a native syntax to leverage constructs within that syntax.

The following new ODBC escape clauses are added to the ODBC grammar. These clauses are not dependent upon the version of ODBC; applications determine support for the clauses through SQLGetInfo.

####3.3.5.1 Limit Clause

Interactive applications typically need a way to “preview” results, for example by showing the first n rows. Applications may also “page” by selecting the next n rows after skipping m.

Different databases support this through different syntax (top, first, skip, start, fetch first, limit, etc.) ANSI SQL 2003 defines window functions for defining a ROW\_NUMBER that can then be filtered on and then selected out.

ODBC adds a new *ODBC-limit-escape*, defined as follows:

*ODBC-limit-escape* ::=
     *ODBC-esc-initiator* limit \[*skip-value comma*\] *take-value ODBC-esc-terminator *

Where *skip-value* is an integer specifying the number of rows to skip and *take-value* is an integer specifying the number of rows to return.

The optional *ODBC-limit-escape* clause immediately follows the *order-by-clause*, if present.

For example:

Select \* from table {limit 10}

would fetch the first ten rows of the table, while

Select \* from table {limit 20,10}

would skip twenty rows and then fetch the next ten.

Drivers advertise support for this escape clause through the new SQL\_LIMIT\_ESCAPE\_CLAUSE *InfoType*. The value of the SQL\_LIMIT\_ESCAPE\_CLAUSE is a bitmask made up of the following values:

> SQL\_LC\_NONE = 0 – the driver has no support for the limit escape clause
> SQL\_LC\_TAKE = 1 – the driver supports only the take portion of the limit escape clause
> SQL\_LC\_TAKE\_AND\_SKIP = 3 – the driver supports both take and skip in the limit escape clause

####3.3.5.2 Selecting Inserted/Updated/Deleted Values

The *ODBC-return-escape* clause returns a table containing information from inserted, updated, and deleted records:

*ODBC-return-escape* ::=
     *ODBC-esc-initiator* return *select-list* from (*vendor-dml-statement*) *ODBC-esc-terminator *

For example, the following statement returns a table containing the id and total of a newly inserted row:

{return id from (INSERT INTO table(amount,total) VALUES (2,3))}

The following statement retrieves the new total of all rows with ids above 10 that were updated:

{return total from (UPDATE table SET {amount=amount\*10} WHERE id &gt; 10)}

The following statement retrieves the ids of all rows with totals above 10 that were deleted:

{return id from (DELETE FROM table WHERE total &gt; 10)}

Drivers advertise support for this escape clause through the new SQL\_RETURN\_ESCAPE\_CLAUSE *InfoType* whose value is a bitmask made up of the following values. Note that supporting an arbitrary column for an expression implies supporting the id for that expression.

| Value                | Description                                                                         |
|----------------------|-------------------------------------------------------------------------------------|
| SQL\_RC\_NONE = 0    | The driver has no support for the return escape clause                              |
| SQL\_RC\_INSERT\_ID  | The driver supports getting the ID of inserted rows                                 |
| SQL\_RC\_INSERT\_ANY | The driver supports getting arbitrary columns from inserted rows                    |
| SQL\_RC\_UPDATE\_ID  | The driver supports getting the ID of inserted updated rows                         |
| SQL\_RC\_UPDATE\_ANY | The driver supports the driver supports getting arbitrary columns from updated rows |
| SQL\_RC\_DELETE\_ID  | The driver supports getting the ID of inserted rows                                 |
| SQL\_RC\_DELETE\_ANY | The driver supports the driver supports getting arbitrary columns from deleted rows |

####3.3.5.3 Native Syntax

ODBC clients use the SQL\_NOSCAN statement attribute to specify that the command text is in a native syntax and should not be parsed by the driver. In this case, the entire statement must be in a native syntax and escape clauses must not be used as the driver will not scan the statement but instead pass it directly to the server.

The *ODBC-native-escape* clause enables clients to embed native syntax within a SQL92 statement:

<span id="_MailEndCompose" class="anchor"></span>*ODBC-native-escape* ::=
     *ODBC-esc-initiator* native (*command-text*) \[*returning-clause*\] *ODBC-esc-terminator *

*returning-clause* ::= RETURNING (*type* \[, *type*\]…) \[*json-format-clause*\]

*json-format-clause* ::= FORMAT JSON \[ENCODING {UTF8 | UTF16 | UTF32}\] * *

*type* ::= {*data-type* | ROW ( field-definition \[, field-definition\]… ) } \[ARRAY | MULTISET\]

*field-definition* ::= *field-name type*

Where *command-text* is a textual query in the native language of the service. Any parenthesis not within single-quotation marks within command-text must be balanced.

The *returning-clause* is required for retrieving results from the native syntax and when embedding the native syntax in place of a query expression within a standard SQL statement.

Single question marks within the native command not within single-quotation marks are interpreted as parameter markers. In order to pass an unquoted question mark as part of the native syntax, the application must double the question mark. The driver will convert unquoted doubled question marks to single question marks when evaluating the native command.

Drivers advertise support for this escape clause through the new SQL\_NATIVE\_ESCAPE\_CLAUSE *InfoType* whose value is the character string “Y” if the escape clause is supported; “N” otherwise.

####3.3.5.4 Refresh Schema

ODBC adds a new *ODBC-refresh-schema-escape* clause to enable clients to request that schema be refreshed, defined as follows:

*ODBC-refresh-schema-escape* ::=
     *ODBC-esc-initiator* refresh \[*sql-identifier*\[, *sql-identifier*\]\*\] *ODBC-esc-terminator *

The optional list of sql-identifiers provide a hint to the driver as to the specific objects from the information schema objects (tables, udts, etc.) that should be refreshed. The driver is always allowed to refresh additional schema objects, particularly in the case where the tables/udts/etc are virtualized based on a non-relational view.

###3.3.6 Added Reserved Words

The following are added to the list of ODBC reserved words:

ARRAY

MULTISET

ROW

String Format
-------------

A new SQLCHAR\*-valued descriptor field, SQL\_DESC\_MIME\_TYPE, is added to \[ALL\] descriptors to specify the format of a field or parameter whose SQL\_DESC\_TYPE specifies a string or binary type. The value of this descriptor field is the mime type of the data (i.e., application/json, application/xml,…).

##3.5 Schema Inference

Schema can be inferred over a schema-less source, such as a JSON, XML, or CSV document, to expose a standard relational view over the data.

###3.5.1 SQL\_SCHEMA\_INFERENCE GetInfo

Drivers that support schema inference return SQL\_TRUE when an application calls SQLGetInfo with the new SQL\_SCHEMA\_INFERENCE *InfoType*.

###3.5.2 SQL\_ATTR\_INFERENCE\_ACCURACY Connection Attribute

A value between 1-100 specifying the accuracy of the inferred schema.

Setting this attribute to 100 means that all rows should be read in order to ensure that the inference is accurate for every row in the result.

Setting this attribute to zero specifies that the driver should not infer schema. In this case, a driver that does not have an explicit way to determine schema returns an empty result when calling SQLColumns, and every column is treated as an [dynamic column](#_Unschematized_Columns_1). Applications can then do their own sampling of the data, and can impose column types in their queries using CONVERT.

A value between 1 and 99 gives an approximate estimation of the quality of the inference. The driver may adjust the number of rows sampled, the method of sampling, or how specific of a type or length to infer based on this setting.

Setting this attribute (including setting to its current value) forces the driver to re-infer schema.

Attempting to set this attribute to a value not supported by the driver may returns SQL\_SUCCESS\_WITH\_INFO with a diagnostic code of 01S02, Option Value Changed. In this case, SQLGetStmtAttr can be called to retrieve the value being used.

Attempting to get or set this attribute for drivers that do not support schema inference returns SQL\_ERROR with a diagnostic code of HYC00, Optional feature not implemented.

<span id="_Unschematized_Columns_1" class="anchor"><span id="_Getting_Data_during" class="anchor"></span></span>

##3.6 Getting Data during Fetch

In ODBC 3.x, applications read large data values by placing them at the end of the select list and calling SQLGetData after all bound columns have been populated.

Applications can specify that certain columns are to be retrieved using SQLGetData or SQLGetNestedData as the bound columns are being populated during fetch by setting the StrLen\_or\_IndPtr to a buffer containing SQL\_DATA\_AT\_FETCH.

In order to specify that a column is to be retrieved using SQLGetData or [SQLGetNestedHandle](#sqlgetnestedhandle):

1.  The application calls SQLBindCol (or equivalent SQLDescField/SQLDescRec), specifying the following:

    -   ColumnNumber is the number of the column to be retrieved using SQLGetData/SQLGetNestedData

    -   TargetType is the type of the column

    -   TargetValuePtr and BufferLength are ignored

    -   StrLen\_or\_IndPtr is set to a buffer containing SQL\_DATA\_AT\_FETCH

2.  App calls SQLFetch/SQLFetchScroll

    -   Driver begins populating the bindings

    -   If the driver comes to a column for which *str\_len\_or\_indicator\_ptr* values is set to SQL\_DATA\_AT\_FETCH, it sets *str\_len\_or\_indicator\_ptr* for all columns not yet retrieved, and whose values are not SQL\_DATA\_AT\_FETCH, to SQL\_DATA\_UNAVAILABLE.

    -   Driver returns SQL\_DATA\_AVAILABLE return code from SQLFetch/SQLFetchScroll

        1.  Calling SQLFetch/SQLFetchScroll after SQLFetch, SQLFetchScroll, or SQLNextColumn returns SQL\_DATA\_AVAILABLE skips the remaining columns and begins fetching the next row.

3.  App calls SQLNextColumn to find out which column is available. Applications must not assume that the columns are returned in any particular order.

    -   Driver returns the ordinal of the available column

    -   For row array sizes greater than 1, the row for which the column is available can be determined through the SQL\_ATTR\_ROWS\_FETCHED\_PTR statement attribute. The client cannot assume that rows prior to the current value of SQL\_ATTR\_ROWS\_FETCHED\_PTR are complete until the entire fetch sequence has completed and SQLFetch or SQLNextColumn returns a value other than SQL\_DATA\_AVAILABLE.

4.  App calls SQLGetData/SQLGetNestedHandle, as appropriate, to read the specified column

    -   Or just ignores it, or binds it

5.  App calls [SQLNextColumn](#sqlnextcolumn) to continue fetching the current row

    -   Driver returns SQL\_DATA\_AVAILABLE, along with the ordinal of the next column available to be retrieved, if additional columns are available

    -   Driver returns SQL\_SUCCESS or SQL\_SUCCESS\_WITH\_INFO when all columns for which a binding has been specified have been retrieved.

6.  App can call SQLGetData/SQLGetNestedData for additional columns beyond the last bound column, or subject to the ordering constraints of the driver.

<span id="_SQLNextColumn" class="anchor"><span id="_Untyped/Variable_Typed_Columns" class="anchor"><span id="_New_SQL_GetData" class="anchor"></span></span></span>
###3.6.1 New SQL\_GD\_CONCURRENT bit for SQL\_GETDATA\_EXTENSIONS

A new SQL GetData Extension, SQL\_GD\_CONCURRENT, is added. If the driver reports SQL\_GD\_CONCURRENT, the driver supports concurrent fetching of multiple columns using SQLGetData and/or SQLGetNestedData. If the driver specifies SQL\_GD\_ANY\_ORDER and does not specify SQL\_GD\_CONCURRENT, calling SQLGetData or SQLGetNestedHandle for a column resets the position of any columns previously fetched using SQLGetData to the beginning and closes the cursor of any previously fetched nested columns for this row.

##3.7 Variable Typed Columns

###3.7.1 Schema Extensions for Variable Typed Columns

Columns whose type may vary across rows are described to ODBC 4.0 clients using the SQL\_VARIANT\_TYPE value for DATA\_TYPE and SQL\_DATA\_TYPE in SQLColumns and when describing a result prior to fetching the first row. This value must not be used in SQLColumns for ODBC 3.5 and earlier clients.

###3.7.2 Query Extensions for Variable Typed Columns

Expressions involving variable typed columns whose type is not compatible with the expression result in the unchanged value of the column. So if the value of a variable typed column Age is “unknown”, then Age+5 results in “unknown”. Boolean expressions comparing instances of variable typed columns whose type is not compatible evaluate to false, so Age&gt;5, Age&lt;5, and Age=5 all evaluate to false for Age with a value of “unknown”.

###3.7.3 Response Extensions for Variable Typed Columns

Upon fetching a row containing a column defined as SQL\_VARIANT\_TYPE, or if a column is retrieved that does not match the data type of the current IRD, the driver updates the IRD with the actual type of the column in the current row, sets the status of any unretrieved columns to SQL\_DATA\_UNAVAILABLE, and returns the new result code SQL\_METADATA\_CHANGED.

Upon receiving SQL\_METADATA\_CHANGED, the application can call SQLNextColumn to determine the index of the column has changed, can get the current type for the changed column, and can either bind the column using an appropriate type, or call SQLGetData or SQLGetHandle as appropriate for the type of the column.

The application then calls SQLNextColumn to continue fetching values, including the changed column if it has been appropriately bound.

Clients can bind parameters using SQL\_VARIANT\_TYPE. Parameters that are bound with a type of SQL\_VARIANT\_TYPE are treated as data-at-exec parameters. When executing a statement that contains parameters whose type is SQL\_VARIANT\_TYPE, the driver returns SQL\_NEED\_DATA. The application calls SQLParamData to find out which column the driver is requesting, and then calls SQLPutData for that parameter.

####3.7.3.1 SQL\_ATTR\_TYPE\_EXCEPTION\_BEHAVIOR

The new SQL\_ATTR\_TYPE\_EXCEPTION\_BEHAVIOR is a SQL\_USMALLINT value that allows the client to control how type exceptions occur when retrieving values.

**SQL\_TE\_ERROR** – (Default) Return an error if the type of the data value is incompatible with the bound type.

**SQL\_TE\_CONTINUE** – Set the *str\_type\_or\_ind\_ptr* value to SQL\_TYPE\_EXCEPTION and continue, or error if *str\_type\_or\_ind\_ptr* is a null pointer.

**SQL\_TE\_REPORT** – Return SQL\_METADATA\_CHANGED and allow the application to rebind the column or retrieve the column using SQLGetData or SQLGetNestedHandle, as appropriate.

###3.7.4 Write Extensions for Variable Typed Columns

Applications can use the CONVERT scalar function to specify a SQL\_TYPE to be used when setting the value for a variable-typed column in an INSERT or UPDATE statement. In the absence of a CONVERT, the driver will store the value in an appropriate type given the value specified in the INSERT or UPDATE statement for the variable-typed column.

Applications can pass variable-typed parameters at execute time by setting the ParameterType in SQLBindParameter, or SQL\_DESC\_PARAMETER\_TYPE in the descriptor, to SQL\_VARIANT\_TYPE, and *StrLen\_or\_IndPtr* to indicate SQL\_DATA\_AT\_EXEC. After calling SQLExecute or SQLExecDirect, the driver will return SQL\_NEED\_DATA for any data-at-exec parameters. The application calls SQLParamData to discover which parameter the driver is ready to process. For variable-typed columns, the application binds the parameter using the appropriate type (and optionally SQL\_DESC\_TYPE\_NAME) for the current row and calls SQLParamData, or sets the SQL\_DESC\_PARAMETER\_TYPE (and optionally SQL\_DESC\_TYPE\_NAME) to the type of the parameter in the current row and calls SQLPutData or SQLGetNestedHandle, as appropriate, to send the information for the current parameter.

###3.7.5 Data Definition Extensions for Variable Typed Columns

Clients can call SQLGetTypeInfo with a *DataType* value of SQL\_VARIANT\_TYPE to determine the implementation-specific name of any types that can be used to hold variable-typed data.

##3.8 Variable Length Columns

ODBC 4.0 drivers report the expected size of string and binary columns in SQLColumns. Applications generally base the allocation of bound buffers on this size.

In some cases, the size of the data may exceed the length of the bound buffer. In this case, the application has the option of ignoring the extra data as in ODBC 3.x (the driver returns 01004, String data right truncated) or having the driver return SQL\_MORE\_DATA at fetch time.

When the driver returns SQL\_MORE\_DATA, the application calls SQLNextColumn in order to determine the column of partially fetched data. The bound buffer will be populated and the str\_len\_or\_ind\_ptr will indicate the total amount of data available, if known, or SQL\_NO\_TOTAL if the driver does not know how much additional data is to be written. At this point the application can call SQLGetData in order to retrieve the remainder of the string or binary data. The application calls SQLNextColumn in order to continue processing the fetch.

###3.8.1 SQL\_ATTR\_LENGTH\_EXCEPTION\_BEHAVIOR

The new SQL\_ATTR\_LENGTH\_EXCEPTION\_BEHAVIOR is a SQL\_USMALLINT value that allows the client to control how binary and string overflows are handled when retrieving values.

**SQL\_LE\_CONTINUE** – (Default) Set the *str\_type\_or\_ind\_ptr* value for the column to the total length available, if known, otherwise SQL\_NO\_TOTAL. When the driver completes fetching the row it will return SQL\_SUCCESS\_WITH\_INFO with a SQLSTATE of 01004, String data, right truncated.

**SQL\_LE\_REPORT** – Return SQL\_MORE\_DATA from the fetch operation and allow the application to retrieve the remainder of the column using SQLGetData.

##3.9 Dynamic Columns

Columns that are known and consistent across rows are exposed in SQLColumns. Applications can bind to these columns in the regular way.

ODBC is extended to support dynamic columns (columns not returned in SQLColumns). The set of such “Dynamic” columns may vary in order, type, or existence from row to row.

Dynamic columns are discovered by the application when retrieving results.

A table may be entirely unschematized (in which case SQLColumns returns an empty result set) or partially schematized (in which case SQLColumns returns the columns that are known and consistent across rows).

###3.9.1 SQL\_ATTR\_DYNAMIC\_COLUMNS Statement Attribute

The new SQL\_ATTR\_DYNAMIC\_COLUMNS statement attribute controls whether or not additional, dynamic columns may be returned for unbounded select results.

The default for this statement attribute is *False*.

Data sources with fixed schemas always return *False* for this value. Attempting to set this value for such a data source returns SQL\_SUCCESS\_WITH\_INFO with a diagnostic code of 01S02, Option Value Changed, and reading this value will continue to return *False*.

If set to true, SQLFetch, SQLFetchScroll, SQLGetData, and SQLCloseCursor for nested statement handles return SQL\_DATA\_AVAILABLE if new columns are discovered/added to the IRD while retrieving results.

If SQL\_ATTR\_DYNAMIC\_COLUMNS is true, then selected columns that don’t apply to the current row have their *len\_or\_ind\_ptr* set to SQL\_DATA\_UNAVAILABLE. If SQL\_ATTR\_DYNAMIC\_COLUMNS is false, such rows have their *len\_or\_ind\_ptr* set to SQL\_NULL.

###3.9.2 Schema Extensions for Dynamic Columns

If SQL\_ATTR\_DYNAMIC\_COLUMNS is true, tables that support properties not defined in SQLColumns are returned from SQLTables using the new “OPEN TABLE” table type. For ODBC 3.8 and lower applications, these, or where SQL\_ATTR\_DYNAMIC\_COLUMNS is false, such tables are returned with a table type of TABLE.

###3.9.3 Query Extensions for Dynamic Columns

A non-trivial select-list (i.e., anything other than \*) imposes a structure on the results. Such results should only contain the specified columns, while \* should also include any dynamic (unschematized) columns.

Structured columns specified in a select list implicitly select all columns (declared and dynamic) of the structured type.

For OPEN TABLEs, any valid identifier for the driver is allowed as a column reference in a SQL Statement (select-list, expression in a where-clause, etc.). Columns that aren’t defined for a particular row, or whose type is not compatible in an expression, are considered null in evaluating the expression. Such columns in a select-list are treated as [untyped columns](#_Untyped/Variable_Typed_Columns) prior to the first fetch.

For tables not reported as open (including all tables for pre-ODBC 4.0 clients), it remains an error to reference an undefined column.

###3.9.4 Response Extensions for Dynamic Columns

Result descriptor functions (SQLNumResultCols, SQLDescribeCol, SQLColAttribute(s), SqlGetDescField/Rec) called before the first call to SQLFetch/SQLFetchScroll describe the known/fixed schema columns. After that, these functions reflect the columns discovered so far. The set of columns actually available may vary from row to row and the driver may expand the set of discovered columns as the application reads data. Once discovered, a column is added to the IRD for the lifetime of the IRD (i.e., until the cursor is closed).

####3.9.4.1 Retrieving Dynamic Columns

If SQL\_ATTR\_DYNAMIC\_COLUMNS is set to true, the driver can return dynamic columns as data-at-fetch columns when fetching a row.

If the driver encounters a dynamic (unschematized) column that does not match any existing IRD records, the driver

1.  Creates a new IRD record for the column; this becomes a new, unbound column in the result. The application must bind the column, or set the str\_len\_or\_indicator\_ptr to SQL\_DATA\_AT\_FETCH, in order to retrieve the value in subsequent calls to SQLFetch. Once created, the type and length fields of the IRD record may change as new dynamic columns with the same name are discovered. Alternatively, the driver may create additional IRD records with the same name for dynamic (unschematized) columns of different types. For nested statement handles, the new columns are preserved in the IRD for subsequent nested handles associated with the same column in the parent IRD.

2.  Returns SQL\_DATA\_AVAILABLE from SQLFetch/SQLFetchScroll/SQLCloseCursor on a nested statement handle

    1.  Application calls [SQLNextColumn](#sqlnextcolumn) to retrieve the ordinal of the descriptor record that describes the column.

    2.  Application calls SQLBindCol (or equivalent) for the column, or retrieves the data directly using SQLGetData or SQLGetNestedHandle, as appropriate. The application must not both bind the data and retrieve the column using SQLGetData/SQLGetNestedData unless supported as a get data extension by the driver.

    3.  Application calls SQLNextColumn to continue fetching the current row. If the dynamic column has now been bound it is fetched into the bound buffer and processing continues.

If the driver returns SQL\_DATA\_AVAILABLE, the application must not assume that any bound columns have been populated, regardless of their ordinal. The driver sets *str\_len\_or\_indicator\_ptr* for bound columns that have not yet been fetched and whose *str\_len\_or\_indicator* does not specify SQL\_DATA\_AT\_FETCH to SQL\_DATA\_UNAVAILABLE. If str\_len\_or\_indicator\_ptr is null, then the driver continues without specifying which columns are unavailable.

###3.9.5 Write Extensions for Dynamic Columns

Open Tables may support inserting or updating dynamic columns.

To create/update dynamic columns, clients can specify column names outside of those returned in SQLColumns in the column list for an insert/update statement. An Insert or Update statement that specifies an unschematized column adds a dynamic column for that row in the table. Setting the value of a dynamic column to null removes that column from that row.

###3.9.6 Data Definition Extensions for Dynamic Columns

Applications can create tables and types that support the presence of dynamic columns using the new [*ODBC-open-escape*](#added-reserved-words) clause.

####3.9.6.1 Open Escape Clause

ODBC adds a new *ODBC-open-escape* clause that can be specified when creating a type or a table to specify that an instance of the type, or a row within the table, can have additional [dynamic](#_Unschematized_Columns_1) columns not specified in the definition:

*ODBC-open-escape* ::=
     *ODBC-esc-initiator* open *ODBC-esc-terminator *

For example:

CREATE {open} TYPE FULLNAME(Firstname varchar(25), LastName varchar (25))

> CREATE {open} TABLE Employees(FirstName varchar(25),
> LastName varchar(25))

Tables created with the ODBC-open-escape clause can have an empty column list, in which case all of the columns are unschematized:

> CREATE {open} TABLE Stuff()

##3.10 Structured Columns

ODBC 4.0 adds support for structured columns.

The ability to report structured columns leverages the extensible type facility supported in ODBC 3.8 and greater. If a client has not called SQLSetEnvAttr with SQL\_ATTR\_ODBC\_VERSION attribute set to ODBC\_OV\_ODBC3\_80 or greater, then the server must not return the SQL\_ROW or SQL\_UDT values in the DATA\_TYPE or SQL\_DATA\_TYPE attributes. In this case the server may either flatten the structured value into multiple columns, create a join table containing the nested results, return the nested results as a string, or omit the nested results, as appropriate for the data source.

###3.10.1 Schema Extensions for Structured Columns

Structured columns may have a named typed, an unnamed (anonymous) type, or may be untyped.

Named structured types are enumerated through the new [SQLStructuredTypes](#sqlstructuredtypes) schema function. Their structure is described in [SQLStructuredTypeColumns](#sqlstructuredtypecolumns).

Anonymous types cannot be enumerated through SQLStructuredTypes, but their structure can be described by passing a path as the TABLE\_NAME to SQLColumns, or the UDT\_NAME to SQLStructuredTypeColumns. The dot-separated path starts with the table or named structural type name in which the anonymous type is used, can traverse an arbitrary number of SQL\_ROW columns, and must terminate on a column whose type is SQL\_ROW.

If calling SQLColumns or SQLStructuredColumns in this way returns SQL\_ROW as the value of the DATA\_TYPE and SQL\_DATA\_TYPE columns, then the column is an untyped structural column. The members of an untyped structured column are treated as [dynamic columns](#schema-inference) by the client.

<span id="_SQLStructuredTypes" class="anchor"><span id="_SQLColumns_for_structured" class="anchor"><span id="_SQLStructuredTypeColumns" class="anchor"></span></span></span>
####3.10.1.1 Additional Columns in SQLColumns

The following columns are added, in order, to the set of columns returned by SQLColumns when requested by ODBC 4.0 or greater applications:

| **Column name**  | **Column number** | **Data type** | **Comments**                                                           |
|------------------|-------------------|---------------|------------------------------------------------------------------------|
| CHAR\_SET\_CAT   | 19                | Varchar       | Catalog of the Character set for the column, or null.                  |
| CHAR\_SET\_SCHEM | 20                | Varchar       | Schema of the Character set for the column, or null.                   |
| CHAR\_SET\_NAME  | 21                | Varchar       | Character set name for the column, or null.                            |
| COLLATION\_CAT   | 22                | Varchar       | Catalog of the Collation for the column, or null.                      |
| COLLATION\_SCHEM | 23                | Varchar       | Schema of the Collation for the column, or null.                       |
| COLLATION\_NAME  | 24                | Varchar       | Name of the Collation for the column, or null.                         |
| UDT\_CAT         | 25                | Varchar       | Catalog of the structured type if the column is a UDT, otherwise null. |
| UDT\_SCHEM       | 26                | Varchar       | Schema of the structured type if the column is a UDT, otherwise null.  |
| UDT\_NAME        | 27                | Varchar       | Name of the structured type if the column is a UDT, otherwise null.    |

The first six are set to null if not supported by the driver. The last three can be passed to SQLStructuredTypeColumns to describe the columns of a UDT, and are null for DATA\_TYPE/SQL\_DATA\_TYPE values other than SQL\_UDT.

###3.10.2 Query Extensions for Structured Columns

Referencing a structured column by name references the entire structured column.

For example:

Select EmpID, Address from Employee

would return a result with two columns; one named “EmpID” containing the employee ID, and one named “Address” containing the address as a structured column.

A subset of the properties for a structural type can be selected by following the reference to the structural column with a comma-separate list of properties, enclosed in parenthesis.

For example:

Select EmpID, Address(Region,Country) from Employee

would return a result with two columns; one named “EmpID” containing the employee ID, and one named “Address” containing only the Region and Country columns from the Address column. This is equivalent to the following query:

> Select EmpID, ROW(Region, Country) as Address
> from Employee

Two structured values are comparable if, and only if, they have the same structural members. For a comparison operation *op* between structured values *SV1* and *SV2*, the expression *SV1 op SV2* is considered true if, for every member *m1* in *SV1* and corresponding member *m2* in *SV2*, *m1 op m2* is true.

It is generally not possible to order by structured values, although it is possible to order by primitive members of a structural value.

Individual members of a structured column may be referenced as the name of the structured column, followed by a single dot (.), followed by the name of the member. This naming schema is recursive (i.e., when referencing a member of a structured member that is itself a member of a structured member.

For example:

Select EmpID, Address.Region, Address.Country from Employee

would return a result with three columns; one named “EmpID” containing the employee id, one named “Address.Region” containing the region member of the employee’s address, and one named “Address.Country” containing the country member from the employee’s address.

###3.10.3 Response Extensions for Structured Columns

A new function, SQLGetNestedHandle, is added in order to retrieve results from structured columns and arrays.

 <span id="_SQLGetNestedData" class="anchor"><span id="_SQLGetNestedHandle" class="anchor"></span></span>
####3.10.3.1 Result Descriptors for structured columns

For structured columns with named types, the value of the DATA\_TYPE and SQL\_DATA\_TYPE attributes is SQL\_UDT and the TYPE\_NAME attribute returns the qualified name of the type.

Additional descriptor fields can be used to describe UDTs in results:

SQL\_DESC\_USER\_DEFINED\_TYPE\_CATALOG (1026)

SQL\_DESC\_USER\_DEFINED\_TYPE\_SCHEMA (1027)

SQL\_DESC\_USER\_DEFINED\_TYPE\_NAME (1028)

Note: ANSI also defines USER\_DEFINED\_TYPE\_CODE (1045) with a value of “DISTINCT” (1) or “STRUCTURED” (2).

For structured columns not described by a named type, the value of the DATA\_TYPE and SQL\_DATA\_TYPE attributes is SQL\_ROW.

Calling GetNestedHandle for an unnamed structured type returns a statement handle with descriptor fields describing any known columns of the nested type. The members of an untyped structured column are treated as [dynamic columns](#schema-inference) by the client.

###3.10.4 Write Extensions for Structured Columns

Clients may specify the value for a member of a structured column in an INSERT or UPDATE statement by referencing the member directly. For Example:

UPDATE Employee SET FullName.FirstName = 'John', FullName.LastName='Smith'

Dynamic columns can similarly be added to structured columns defined as open, for example:

UPDATE Employee SET FullName.Foo = 'Bar'

Alternatively, clients may specify the value of a structured column in an INSERT or UPDATE statement using the *row-value-constructor* defined as follows:

*row-value-constructor* ::=
ROW *left-paren value-expression* \[*comma* *value-expression*\]… *right-paren*

For example:

UPDATE Employee SET FullName = ROW('John', 'Smith')

All members must be specified in the order they occur in the definition of the structured type being updated. To update a subset of properties, a column-list can be added to the structured column:

UPDATE Employee SET FullName(FirstName, LastName) = ROW('John', 'Smith')

In this case, the order of the members of the row-value-constructor must match the order of columns in the column-list applied to the structured column.

Appending a column-list to the structured column can also be used to add dynamic columns to the structured column, for example, the following statements add a “Foo” member to the FullName column:

UPDATE Employee SET FullName(Foo) = ROW('Bar')

UPDATE Employee SET FullName(FirstName, Foo) = ROW('John', 'Bar')

Side note: ANSI has two forms; one that is prefixed with “ROW” and can have one or more elements, and one that is not prefixed and must have at least two elements.

Alternatively, clients can use a typed-constructor.

*typed-value-constructor* ::=
*type-name* *left-paren value-expression* \[*comma* *value-expression*\]… *right-paren*

All members must be specified in the order they occur in the definition of the structured type being updated.

For example:

UPDATE Employee SET FullName = FULLNAME('John','Smith')

Clients may update an individual member within a structured column by referencing that member in the set list of an update statement.

For example:

UPDATE Employee SET Address.Country = ‘USA'

####3.10.4.1 Passing structured parameters at execute time

Structured parameters can be passed to the driver at execute time.

To pass a structured parameter at execute time, the application:

1.  Calls SQLBindParameter (or equivalent SQLDescField/SQLDescRec), specifying the following:

-   ValueType is SQL\_C\_DEFAULT (sets SQL\_DESC\_TYPE/SQL\_DESC\_CONCISE\_TYPE in APD)

-   ParameterType is SQL\_ROW or SQL\_UDT (sets SQL\_DESC\_TYPE/SQL\_DESC\_CONCISE\_TYPE in IPD)

    -   For SQL\_UDT, must set descriptor fields to specify:

        -   USER\_DEFINED\_TYPE\_CATALOG (1026)

        -   USER\_DEFINED\_TYPE\_NAME (1028)

        -   USER\_DEFINED\_TYPE\_SCHEMA (1027)

-   ParameterValuePtr is set to a 32bit value identifying the parameter (passed back to the app from SQLParamData when data is needed for this parameter)

-   StrLen\_or\_IndPtr is set to a buffer containing SQL\_DATA\_AT\_EXEC

1.  App calls SQLPrepare/SQLExecute or SQLExecDirect

    -   Driver returns SQL\_NEED\_DATA.

2.  App calls SQLParamData to find out which param the driver is asking about.

    -   Driver returns SQL\_NEED\_DATA and passes back value from ParameterValuePtr.

3.  To specify a null or a default parameter value, the application calls SQLPutData with *str\_len\_or\_ind* set to SQL\_NULL\_DATA or SQL\_DEFAULT\_PARAM, respectively, followed by SQLParamData to progress to the next parameter.

4.  For a non-null structured parameter, the app calls SQLGetNestedHandle for the specified parameter

    -   Driver returns a nested statement handle

5.  App calls SQLBindParameter/SQLSetDescField to bind the nested parameters as named parameters.

    -   Individual property names are specified by setting the SQL\_DESC\_NAME attribute in the Implementation Parameter Descriptor (Driver sets SQL\_DESC\_UNNAMED to SQL\_ NAMED)

6.  App sets the appropriate values in the buffer and calls SQLExecute on the nested handle to send the row

    -   Driver returns SQL\_NEED\_DATA if there are any data-at-execute parameters on the nested handle

7.  App calls SQLParamData to indicate it has sent all of the data for the parameter

    -   If the parameter is a structured- or collection-valued parameter whose value was set through a nested statement handle, calling SQLParamData implicitly frees that handle

    -   Driver returns SQL\_NEED\_DATA if there are any remaining data-at-execute parameters.

###3.10.5 Data Definition Extensions for Structured Columns

Structured types may be defined using the CREATE TYPE escape clause, as follows:

{ CREATE \[Open\] TYPE *type\_name* (*column-identifier data-type* \[*,column-identifier data-type*\]...) }

For example:

{CREATE OPEN TYPE FULLNAME(Firstname varchar(25), LastName varchar (25))}

These types can be enumerated through the [SQLStructuredTypes](#sqlstructuredtypes) schema function. Their structure is described through [SQLStructuredTypeColumns](#sqlstructuredtypecolumns).

The CREATE statement can include the new [*ODBC-open-escape*](#open-escape-clause) clause to specify that the type may contain additional properties not specified as part of its definition.

Structured columns can be created within a table using a named structural type or by using the ROW clause:

CREATE TABLE Employees(FullName ROW(FirstName varchar(25),
LastName varchar(25)), Address ADDRESS)

##3.11 Collection-valued Columns
-------------------------

ODBC adds support for collection-valued columns.

The ability to report collection-valued columns leverages the extensible type facility supported in ODBC 3.8 and greater. If a client has not called SQLSetEnvAttr with SQL\_ATTR\_ODBC\_VERSION attribute set to ODBC\_OV\_ODBC3\_80 or greater, then the server must not return the SQL\_ARRAY or SQL\_MULTISET values in the DATA\_TYPE columns or descriptor fields. In this case the server may either flatten the array values into multiple columns, create a join table containing the nested results, return the nested results as a string, or omit the nested results, as appropriate for the data source.

###3.11.1 Schema Extensions for Collection-valued Columns

Collection-valued table columns are described in SQLColumns. Collection-valued result columns are described through SQLColAttribute(s)/SqlDescribeCol/SqlGetDescriptor.

Collection-valued columns may be ordered or unordered.

####3.11.1.1 Describing Collections using SQLColumns

For ordered array-valued columns, the value of the DATA\_TYPE attributes is SQL\_ARRAY. For unordered array-valued columns, the value of the DATA\_TYPE attributes is SQL\_MULTISET.

The remaining columns, including the SQL\_DATA\_TYPE, describe the type of the element within the collection.

Additionally, clients may get the type of a collection-valued column by passing the name of the collection-valued column appended with “/$element” (recursively, for nested arrays), as the column name in SQLColumns/SQLTypeColumns.

###3.11.2 Query Extensions for Collections

The following query constructs are added in support of collections.

####3.11.2.1 Array Element Reference

Individual members of an array may be referenced using the *array-element-reference* expression.

*array-element-reference* ::=
*array-value-expression left-bracket numeric-value-expression right-bracket*

where *numeric-value-expression* yields an integer greater than zero and less than or equal to the number of elements in *array-value-expression*.

For example:

Select EmpID, PhoneNumbers\[1\] from Employee

returns the employee ids along with their first phone number.

Since multisets are not ordered, their members cannot be individually referenced.

Note that some data sources may use a bracketed string following a field name in the select list as a shortcut for aliasing the field. For fields representing arrays, if the bracketed value is a number the driver must ensure that it is used as an index, not as an alias.

####3.11.2.2 UNNEST

The ANSI SQL *unnest-operator* expands the collection into multiple rows.

*unnest-operator* ::=
UNNEST *left-paren collection-value-expression right-paren*

The result of the *unnest-operator* is a derived table that can then be used with a correlation

For example:

> Select P.Name, Child.Name
> From People P, Unnest(P.Children) as Child

would return a row each child for each person containing the person’s name and child’s name.

###3.11.3 Response Extensions for Collection-valued Columns

Collection-valued result columns are retrieved using [SQLGetNestedHandle](#sqlgetnestedhandle), which returns a statement handle (HSTMT) on which the nested results can be described, bound, and fetched using the appropriate standard ODBC function calls.

###3.11.4 Write Extensions for Collection-valued Columns

Clients may specify the value of an array-valued column in an INSERT or UPDATE statement using the *array-value-constructor* defined as follows:

*array-value-constructor* ::=
ARRAY *left-paren value-expression* \[*comma* *value-expression*\]… *right-paren*

*multiset-value-constructor* ::=
MULTISET *left-paren value-expression* \[*comma* *value-expression*\]… *right-paren*

Multisets and arrays are treated as atomic values when updating. That is, the array or multiset can be replaced in whole, but individual members cannot be added, removed, or updated.

Clients may update an individual member within an array-valued value through the use of the *array-element-reference* expression.

###3.11.5 Data Definition Extensions for Collection-valued Columns

Ordered collection-valued columns are defined using the array column type:

*array-column-type ::=*

*data-type* ARRAY \[ *left-bracket maximum-cardinality right-bracket* \]

For example:

CREATE TABLE *Employee(FullName FULLNAME, Address ADDRESS, PhoneNumbers varchar(25) ARRAY\[10\] ) *

Unordered collection-valued columns are defined using the MULTISET column type :

*multiset-column-type ::=*

*data-type* MULTISET

For example:

CREATE TABLE *Employee(FullName FULLNAME, Address ADDRESS, PhoneNumbers varchar(25) MULTISET ) *

####3.11.5.1 Passing collection-valued parameters at execute time

Collection-valued parameters can be passed to the driver at execute time.

To pass an array-valued parameter at execute time, the application:

1.  Application SQLBindCol (or equivalent SQLDescField/SQLDescRec), specifying the following:

-   ValueType is SQL\_C\_DEFAULT (sets SQL\_DESC\_TYPE/SQL\_DESC\_CONCISE\_TYPE in APD)

-   ParameterType is SQL\_ARRAY or SQL\_MULTISET (sets SQL\_DESC\_TYPE/SQL\_DESC\_CONCISE\_TYPE in IPD)

    -   Sets the SQL\_DESC\_SUBTYPE of the IPD to the type of the array or multiset, if known

    -   For SQL\_UDT sets descriptor fields to specify:

        -   USER\_DEFINED\_TYPE\_CATALOG

        -   USER\_DEFINED\_TYPE\_SCHEMA

        -   USER\_DEFINED\_TYPE\_NAME

-   ParameterValuePtr is set to a 32bit value identifying the parameter (passed back to the app from SQLParamData when data is needed for this parameter)

-   StrLen\_or\_IndPtr is set to a buffer containing SQL\_DATA\_AT\_EXEC

1.  Application calls SQLPrepare/SQLExecute or SQLExecDirect

    -   Driver returns SQL\_NEED\_DATA.

2.  Application calls SQLParamData to find out which param the driver is asking about.

    -   Driver returns SQL\_NEED\_DATA and passes back value from ParameterValuePtr.

3.  To specify a null value, a default value, or an empty array, the application calls SQLPutData with *str\_len\_or\_ind* set to SQL\_NULL\_DATA, SQL\_DEFAULT\_PARAM, or SQL\_EMPTY\_ARRAY, respectively, followed by SQLParamData to progress to the next parameter.

4.  To specify a non-empty array, the application calls GetNestedHandle for the specified parameter

    -   Driver returns a nested statement handle

5.  Application calls BindParameter/SetDescField to bind the nested columns.

    -   Individual column names are specified by setting the SQL\_DESC\_NAME attribute in the Implementation Parameter Descriptor.

6.  Application sets the appropriate values in the buffer and calls SQLExecute on the nested handle to send the row of data

    -   Driver returns SQL\_NEED\_DATA if there are any data-at-execute parameters on the nested handle

7.  Application repeats step 7 for each nested row

8.  Application calls SQLParamData to indicate it has sent all of the data for the parameter

    -   Driver returns SQL\_NEED\_DATA if there are any remaining data-at-execute parameters.

#4 Change Tracking

A number of data sources support the ability to determine changes from a certain point.

ODBC 4.0 provides common functions that services can use to expose this information.

ODBC adds a new *ODBC-supports-change-tracking-escape* that returns 1 if a given table supports change tracking, otherwise 0:

*ODBC-supports-change-tracking-escape* ::=
     *ODBC-esc-initiator* trackschanges(*tablename*) *ODBC-esc-terminator *

Where *tablename* is the unqualified, partially qualified, or fully qualified name of the table.

ODBC adds a new *ODBC-changeversion-escape* for getting the current change version (timestamp, version, or other high-water mark) defined as follows:

*ODBC-changeversion-escape* ::=
     *ODBC-esc-initiator* changeversion(*tablename*) *ODBC-esc-terminator *

The value returned by the *ODBC-changeversion-escape* is a binary value that can be passed to the *ODBC-changes-escape* to retrieve the latest values of any rows that have changed since the specified time:

*ODBC-changes-escape* ::=
     *ODBC-esc-initiator* changes(*tablename, changeversion*) *ODBC-esc-terminator *

The *ODBC-changes-escape* returns all rows from the specified *tablename* that have changed since the specified *changeversion*. The first column of the result is named “ColumnStatus” and contains the value “I” if the row is new since the specified changeversion, “U” if it has been updated since the specified changeversion, and “D” if it has been deleted since the specified change version. The additional columns of the result are the columns of the base table. Deleted rows must contain the “ColumnStatus” column, as well as any columns identified as BEST\_ROWID and ROWVER from SQLSpecialColumns.

#5 ODBC 3.0 Compatibility

How does a 3.0 Client consume a 4.0 Driver?
-------------------------------------------

> What role does the DM play?

How does a 4.0 Client consume a 3.0 Driver?
-------------------------------------------

> What role does the DM play?

What’s required to be a 4.0 driver
----------------------------------

> Data at fetch?
>
> We should support 3.0 drivers exposing new getinfos, etc. without claiming 4.0 support.

#6 New Functions
The following functions are added in ODBC 4.0.

##6.1 SQLNextColumn

In order to retrieve the column of data currently available to be read, the application can call SQLNextColumn.

SQLRETURN SQLNextColumn(
SQLHSTMT StatementHandle,
SQLUSMALLINT\* Col\_or\_Param\_Num);

SQLNextColumn can be called only after SQLFetch or SQLFetchScroll returns SQL\_DATA\_AVAILABLE, and until SQLNextColumn returns SQL\_SUCCESS or SQL\_SUCCESS\_WITH\_INFO.

The Driver Manager returns HY010, Function Sequence Error under the following conditions:

1.  The specified StatementHandle was not in the executed state.

2.  SQLFetch or SQLFetschScroll have not been called on the executed statement.

3.  The most recent call to SQLFetch, SQLFetchScroll, or SQLNextColumn did not return SQL\_DATA\_AVAILABLE

4.  There is an asynchronously executing function called on this statement handle, or the connection associated with this statement handle, that has not completed.

The driver manager returns HY010, Function sequence error, from SQLSetPos if the most recent call to SQLFetch, SQLFetchScroll, or SQLNextColumn returned a value other than SQL\_SUCCESS or SQL\_SUCCESS\_WITH\_INFO.

###6.1.1 Usage

While fetching a row that contains a column whose str\_len\_or\_indicator\_ptr contains SQL\_DATA\_AT\_FETCH, or when reading a dynamic column while SQL\_ATTR\_DYNAMIC\_COLUMNS is true, the driver returns SQL\_DATA\_AVAILABLE and the application calls SQLNextColumn in order to determine the ordinal of the next available column.

Once all DATA\_AT\_FETCH columns have been returned, SQLNextColumn returns SQL\_SUCCESS or SQL\_SUCCESS\_WITH\_INFO with any valid SQLState from SQLFetch/SQLFetchScroll.

Calling SQLNextColumn with a null pointer for *Col\_or\_Param\_Num* skips the rest of the DATA\_AT\_FETCH columns for this row.

##6.2 SQLGetNestedHandle

SQLGetNestedHandle is called in to retrieve a handle for reading or writing structured or collection-valued columns and parameters.

SQLRETURN SQLGetNestedHandle(
SQLHSTMT ParentStatementHandle,
SQLUSMALLINT Col\_or\_Param\_Num,
SQLHSTMT\* OutputChildStatementHandle);

The driver manager enforces many of the same sequencing and validity checks as in SQLGetData. In particular:

The Driver Manager returns 07009, Invalid descriptor index under the following conditions:

1.  The number of the specified column was less than or equal to the number of the highest bound column. This description does not apply to drivers that return the SQL\_GD\_ANY\_COLUMN bitmask for the SQL\_GETDATA\_EXTENSIONS option in **SQLGetInfo**.

2.  The application has already called **SQLGetData** or **SQLGetNestedHandle** for the current row; the number of the column specified in the current call was less than the number of the column specified in the preceding call; and the driver does not return the SQL\_GD\_ANY\_ORDER bitmask for the SQL\_GETDATA\_EXTENSIONS option in **SQLGetInfo**.

The Driver Manager returns 24000 Invalid Cursor State under the following conditions:

1.  The function was called without first calling **SQLFetch** or **SQLFetchScroll** to position the cursor on the row of data required.

2.  The *StatementHandle* was in an executed state, but no result set was associated with the *StatementHandle*.

The Driver Manager returns HY001 under the following condition:

1.  The Driver Manager was unable to allocate memory for the specified handle

The Driver Manager returns HY003, Program type out of range, under the following condition:

1.  The argument *Col\_or\_Param\_Num* was 0.

The Driver Manager returns HY009, Invalid use of a null pointer under the following condition:

1.  The argument OutputChildStatementHandle was a null pointer.

The Driver Manager returns HY010, Function Sequence Error under the following conditions:

1.  The specified StatementHandle was not in the executed state, or no result set was associated with the Statement Handle.

2.  SQLFetch or SQLFetschScroll have not been called on the executed statement.

3.  There is an asynchronously executing function called on this statement handle, or the connection associated with this statement handle, that has not completed.

4.  **SQLExecute**, **SQLExecDirect**, **SQLBulkOperations**, or **SQLSetPos** was called for the *StatementHandle* and returned SQL\_NEED\_DATA. This function was called before data was sent for all data-at-execution parameters or columns.

The Driver Manager returns HY117, Connection is suspended due to unknown transaction state if the connection is in a suspended state.

The driver manager returns IM001, Driver does not support this function, under the following condition:

1.  The driver associated with StatementHandle does not support the function.

###6.2.1 Usage

<span id="OLE_LINK1" class="anchor"><span id="OLE_LINK2" class="anchor"></span></span>After calling SQLFetch or SQLFetchScroll to retrieve bound columns, or SQLNextColumn returns the ordinal of a nested column that has been bound as a [data-at-execute column](#_Getting_Data_during), applications call SQLGetNestedHandle in order to retrieve a statement handle to be used to retrieve nested content.

SQLGetNestedHandle has the same sequencing restrictions as SQLGetData. Drivers report relaxed sequencing rules for both SQLGetData and SQLGetNestedHandle through SQL\_GET\_DATA\_EXTENSIONS. Unless SQL\_GET\_DATA\_EXTENSIONS specifies SQL\_GD\_ANY\_ORDER, *Col\_or\_Param\_num* must be greater than the number of the last bound column and greater than any column previously retrieved with SQLGetNestedHandle or SQLGetData.

After calling SQLGetNestedHandle, the application can use the returned ChildStatementHandle to describe and fetch results for that nested content.

Applications SHOULD call SQLCloseCursor/SQLFreeHandle on the nested handle when finished reading results.

Unless the new [SQL\_GD\_CONCURRENT](#_New_SQL_GetData) bit within SQL\_GETDATA\_EXTENSIONS is specified, a subsequent call to SQLGetData or SQLGetNestedHandle with a different value of *Col\_or\_Param\_Num* implicitly closes and frees the statement handle. If SQL\_GD\_CONCURRENT is specified, a subsequent call to SQLGetData or SQLGetNestedHandle does not affect the current statement handle.

A subsequent call to SQLFetch, SQLFetchScroll, or SQLFreeHandle for the parent statement handle implicitly closes and frees the statement handle if it has not already been freed by the application.

Note that drivers may return the same statement handle value in subsequent calls to SQLGetNestedHandle (i.e., when calling SQLGetNestedHandle for the same column in a subsequent row), but that applications should not assume this to be the case. In particular, applications must use SQLCopyDesc in order to re-use ARD bindings across child statement handles.

Calling SQLGetNestedHandle for a non-structured or collection valued column, or calling SQLGetData on a structured or collection-valued column, returns SQL\_ERROR with SQLState 07009, Invalid descriptor index.

TODO: describe output parameters. Do we need a new InputOutputType, or do we use the existing SQL\_PARAM\_\[INPUT\]/OUTPUT\_STREAM?

##6.3 SQLStructuredTypes

SQLStructuredTypes enumerates named structural types.

SQLRETURN SQLStructuredTypes(
     SQLHSTMT StatementHandle,

     SQLCHAR \* CatalogName,

     SQLSMALLINT NameLength1,

     SQLCHAR \* SchemaName,

     SQLSMALLINT NameLength2,

     SQLCHAR \* TypeName,

     SQLSMALLINT NameLength3);

The driver manager enforces the same sequencing and validity checks as in SQLTables. In particular:

The Driver Manager returns HY010, Function Sequence Error under the following conditions:

1.  There is an asynchronously executing function called on this statement handle, or the connection associated with this statement handle, that has not completed.

2.  **SQLExecute**, **SQLExecDirect**, or **SQLMoreResults** was called for the *StatementHandle* and returned SQL\_PARAM\_DATA\_AVAILABLE. This function was called before data was retrieved for all streamed parameters

3.  **SQLExecute**, **SQLExecDirect**, **SQLBulkOperations**, or **SQLSetPos** was called for the *StatementHandle* and returned SQL\_NEED\_DATA. This function was called before data was sent for all data-at-execution parameters or columns.

The Driver Manager returns HY090 under the following conditions:

1.  The value of one of the length arguments was less than 0 but not equal to SQL\_NTS.

2.  The value of one of the name length arguments exceeded the maximum length value for the corresponding name.

The Driver Manager returns HY117, Connection is suspended due to unknown transaction state if the connection is in a suspended state.

The driver manager returns IM001, Driver does not support this function, under the following condition:

1.  The driver associated with StatementHandle does not support the function.

###6.3.1 Usage

The result of SQLStructuredTypes mirrors the result of SQLTables:

| **Column name** | **Column number** | **Data type** | **Comments**                                                              |
|-----------------|-------------------|---------------|---------------------------------------------------------------------------|
| UDT\_CAT        | 1                 | Varchar       | Catalog name. See TABLE\_CAT in SQLTables.                                |
| UDT\_SCHEM      | 2                 | Varchar       | Schema name See TABLE\_SCHEM in SQLTables.                                |
| UDT\_NAME       | 3                 | Varchar       | Structured Type name.                                                     |
| UDT\_TYPE       | 4                 | Varchar       | UDT Type name; "TYPE", "OPEN\_TYPE", or a data source–specific type name. |
| REMARKS         | 5                 | Varchar       | A description of the Structured Type.                                     |

#6.4 SQLStructuredTypeColumns

SQLStructuredTypeColumns describes the columns of a named structural type.

SQLRETURN SQLStructuredTypeColumns(

     SQLHSTMT StatementHandle,

     SQLCHAR \* CatalogName,

     SQLSMALLINT NameLength1,

     SQLCHAR \* SchemaName,

     SQLSMALLINT NameLength2,

     SQLCHAR \* TypeName,

     SQLSMALLINT NameLength3,

     SQLCHAR \* ColumnName,

     SQLSMALLINT NameLength4);

The result of SQLStructuredTypeColumns mirrors the result of SQLColumns:

| **Column name**     | **Column number** | **Data type**     | **Comments**                                                                                                                     |
|---------------------|-------------------|-------------------|----------------------------------------------------------------------------------------------------------------------------------|
| UDT\_CAT            | 1                 | Varchar           | Catalog name; NULL if not applicable to the data source.                                                                         |
| UDT\_SCHEM          | 2                 | Varchar           | Schema name; NULL if not applicable to the data source.                                                                          |
| UDT\_NAME           | 3                 | Varchar not NULL  | Structured Type name.                                                                                                            |
| COLUMN\_NAME        | 4                 | Varchar not NULL  | Column name. The driver returns an empty string for a column that does not have a name.                                          |
| DATA\_TYPE          | 5                 | Smallint not NULL | SQL data type. See DATA\_TYPE column in SQLColumns.                                                                              |
| TYPE\_NAME          | 6                 | Varchar not NULL  | Data source–dependent data type name                                                                                             |
| COLUMN\_SIZE        | 7                 | Integer           | Size of the column. See COLUMN\_SIZE column in SQLColumns.                                                                       |
| BUFFER\_LENGTH      | 8                 | Integer           | The length in bytes of data transferred on an SQLGetData, SQLFetch, or SQLFetchScroll operation if SQL\_C\_DEFAULT is specified. |
| DECIMAL\_DIGITS     | 9                 | Smallint          | See DECIMAL\_DIGITS in SQLColumns.                                                                                               |
| NUM\_PREC\_RADIX    | 10                | Smallint          | See NUM\_PREC\_RADIX in SQLColumns.                                                                                              |
| NULLABLE            | 11                | Smallint not NULL | See NULLABLE in SQLColumns.                                                                                                      |
| REMARKS             | 12                | Varchar           | A description of the column.                                                                                                     |
| COLUMN\_DEF         | 13                | Varchar           | The default value of the column. See the COLUMN\_DEF column in SQLColumns.                                                       |
| SQL\_DATA\_TYPE     | 14                | Smallint not NULL | See SQL\_DATA\_TYPE in SQLColumns.                                                                                               |
| SQL\_DATETIME\_SUB  | 15                | Smallint          | See SQL\_DATETIME\_SUB in SQLColumns.                                                                                            |
| CHAR\_OCTET\_LENGTH | 16                | Integer           | The maximum length in bytes of a character or binary data type column.                                                           |
| ORDINAL\_POSITION   | 17                | Integer not NULL  | The ordinal position of the column in the structured type. The first column in the UDT is number 1.                              |
| IS\_NULLABLE        | 18                | Varchar           | See IS\_NULLABLE in SQLColumns.                                                                                                  |
| CHAR\_SET\_CAT      | 19                | Varchar           | Catalog of the Character set for the column, or null.                                                                            |
| CHAR\_SET\_SCHEM    | 20                | Varchar           | Schema of the Character set for the column, or null.                                                                             |
| CHAR\_SET\_NAME     | 21                | Varchar           | Character set name for the column, or null.                                                                                      |
| COLLATION\_CAT      | 22                | Varchar           | Catalog of the Collation for the column, or null.                                                                                |
| COLLATION\_SCHEM    | 23                | Varchar           | Schema of the Collation for the column, or null.                                                                                 |
| COLLATION\_NAME     | 24                | Varchar           | Name of the Collation for the column, or null.                                                                                   |
| UDT\_CAT            | 25                | Varchar           | Catalog of the structured type if the column is a UDT, otherwise null.                                                           |
| UDT\_SCHEM          | 26                | Varchar           | Schema of the structured type if the column is a UDT, otherwise null.                                                            |
| UDT\_NAME           | 27                | Varchar           | Name of the structured type if the column is a UDT, otherwise null.                                                              |
