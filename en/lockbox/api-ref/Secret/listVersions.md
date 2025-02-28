---
editable: false
sourcePath: en/_api-ref/lockbox/v1/api-ref/Secret/listVersions.md
---

# Lockbox API, REST: Secret.ListVersions {#ListVersions}

Retrieves the list of versions of the specified secret.

## HTTP request

```
GET https://{{ api-host-lockbox }}/lockbox/v1/secrets/{secretId}/versions
```

## Path parameters

#|
||Field | Description ||
|| secretId | **string**

Required field. ID of the secret to list versions for. ||
|#

## Query parameters {#yandex.cloud.lockbox.v1.ListVersionsRequest}

#|
||Field | Description ||
|| pageSize | **string** (int64)

The maximum number of results per page to return. If the number of available
results is larger than `page_size`, the service returns a [ListVersionsRequest.next_page_token]
that can be used to get the next page of results in subsequent list requests.
Default value: 100. ||
|| pageToken | **string**

Page token. To get the next page of results, set `page_token` to the
[ListVersionsRequest.next_page_token] returned by a previous list request. ||
|#

## Response {#yandex.cloud.lockbox.v1.ListVersionsResponse}

**HTTP Code: 200 - OK**

```json
{
  "versions": [
    {
      "id": "string",
      "secretId": "string",
      "createdAt": "string",
      "destroyAt": "string",
      "description": "string",
      "status": "string",
      "payloadEntryKeys": [
        "string"
      ],
      // Includes only one of the fields `passwordPayloadSpecification`
      "passwordPayloadSpecification": {
        "passwordKey": "string",
        "length": "string",
        "includeUppercase": "boolean",
        "includeLowercase": "boolean",
        "includeDigits": "boolean",
        "includePunctuation": "boolean",
        "includedPunctuation": "string",
        "excludedPunctuation": "string"
      }
      // end of the list of possible fields
    }
  ],
  "nextPageToken": "string"
}
```

#|
||Field | Description ||
|| versions[] | **[Version](#yandex.cloud.lockbox.v1.Version)**

List of versions for the specified secret. ||
|| nextPageToken | **string**

This token allows you to get the next page of results for list requests. If the number
of results is greater than the specified [ListVersionsRequest.pageSize](#yandex.cloud.lockbox.v1.ListVersionsRequest), use
the `next_page_token` as the value for the [ListVersionsRequest.pageToken](#yandex.cloud.lockbox.v1.ListVersionsRequest) query parameter
in the next list request. Each subsequent list request will have its own
`next_page_token` to continue paging through the results. ||
|#

## Version {#yandex.cloud.lockbox.v1.Version}

#|
||Field | Description ||
|| id | **string**

ID of the version. ||
|| secretId | **string**

ID of the secret that the version belongs to. ||
|| createdAt | **string** (date-time)

Time when the version was created.

String in [RFC3339](https://www.ietf.org/rfc/rfc3339.txt) text format. The range of possible values is from
`0001-01-01T00:00:00Z` to `9999-12-31T23:59:59.999999999Z`, i.e. from 0 to 9 digits for fractions of a second.

To work with values in this field, use the APIs described in the
[Protocol Buffers reference](https://developers.google.com/protocol-buffers/docs/reference/overview).
In some languages, built-in datetime utilities do not support nanosecond precision (9 digits). ||
|| destroyAt | **string** (date-time)

Time when the version is going to be destroyed. Empty unless the status
is `SCHEDULED_FOR_DESTRUCTION`.

String in [RFC3339](https://www.ietf.org/rfc/rfc3339.txt) text format. The range of possible values is from
`0001-01-01T00:00:00Z` to `9999-12-31T23:59:59.999999999Z`, i.e. from 0 to 9 digits for fractions of a second.

To work with values in this field, use the APIs described in the
[Protocol Buffers reference](https://developers.google.com/protocol-buffers/docs/reference/overview).
In some languages, built-in datetime utilities do not support nanosecond precision (9 digits). ||
|| description | **string**

Description of the version. ||
|| status | **enum** (Status)

Status of the secret.

- `STATUS_UNSPECIFIED`
- `ACTIVE`: The version is active and the secret payload can be accessed.
- `SCHEDULED_FOR_DESTRUCTION`: The version is scheduled for destruction, the time when it will be destroyed
is specified in the `Version.destroyAt` field.
- `DESTROYED`: The version is destroyed and cannot be recovered. ||
|| payloadEntryKeys[] | **string**

Keys of the entries contained in the version payload. ||
|| passwordPayloadSpecification | **[PasswordPayloadSpecification](#yandex.cloud.lockbox.v1.PasswordPayloadSpecification)**

Includes only one of the fields `passwordPayloadSpecification`. ||
|#

## PasswordPayloadSpecification {#yandex.cloud.lockbox.v1.PasswordPayloadSpecification}

#|
||Field | Description ||
|| passwordKey | **string**

Required field. key of the entry to store generated password value ||
|| length | **string** (int64)

password length; by default, a reasonable length will be decided ||
|| includeUppercase | **boolean**

whether at least one A..Z character is included in the password, true by default ||
|| includeLowercase | **boolean**

whether at least one a..z character is included in the password, true by default ||
|| includeDigits | **boolean**

whether at least one 0..9 character is included in the password, true by default ||
|| includePunctuation | **boolean**

whether at least one punctuation character is included in the password, true by default
punctuation characters by default (there are 32): !"#$%&'()*+,-./:;<=>?@[\]^_`{\|}~
to customize the punctuation characters, see included_punctuation and excluded_punctuation below ||
|| includedPunctuation | **string**

If include_punctuation is true, one of these two fields (not both) may be used optionally to customize the punctuation:
a string of specific punctuation characters to use (at most, all the 32) ||
|| excludedPunctuation | **string**

a string of punctuation characters to exclude from the default (at most 31, it's not allowed to exclude all the 32) ||
|#