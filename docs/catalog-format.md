# How to read `catalog/message-spec.json`

The catalog is keyed by ECF record name (`@ECFClass` name, e.g.
`Epic.Mobile.Core.Login.Request`):

```json
{
  "<Epic.Mobile.X.Y.Request>": {
    "class": "com.epic.x.Y$Request",       // primary backing class
    "aliasClasses": ["com.epic.x.Z"],      // optional: classes sharing this record name
    "kind": "request | response | other",
    "superClass": "java.lang.Object",
    "multipart": true,                     // optional: any member is byte[]/InputStream
    "members": [
      {
        "wire": "Password",                    // final wire property name
        "type": "string",                      // readable type (see table below)
        "javaMember": "getEncryptedPassword",  // accessor or field the name came from
        "multipart": false,
        "fieldAccess": true,                   // optional: getter-less field member
        "enum": true, "enumValues": [...]      // optional: type is a Java enum
      }
    ]
  }
}
```

- `kind`: the `ecfName` suffix `.Request`/`.Response`; if there is no such suffix the
  Java simple name (`...$Request`/`$Response`) is used as fallback; everything else is
  `other`. Record names not ending in Request/Response are not necessarily messages:
  models, enum-like records, and legacy SOAP `urn:epicsystems-com:...` records also
  live in the catalog.

## Wire-name derivation

Catalog `wire` names follow the client's serialization metadata:

1. Default: getter-derived — `getXxx` → `Xxx` with the first letter's case preserved
   (e.g. `getintEPTID` → `intEPTID`); `isXxx` booleans strip the `is`
   (`isDualSign` → `DualSign`). Developers frequently override these back to
   `IsXxx`/`PascalCase` via an explicit transport name, which confirms the rule.
2. Override: an explicit `@transportName` annotation wins over the derived name
   (244 members actually differ from the derived name, e.g. `getEncryptedPassword` →
   `Password`).
3. Getter-less public fields (Kotlin data classes / Java beans) are serialized by
   field name verbatim, including a leading `_` (marked `fieldAccess: true`;
   2,937 members).
4. Synthetic members are dropped (`$stable`, `$annotations` metadata).

The first-letter-case question for the JSON transport (PascalCase vs camelCase on the
wire) is the open item tracked in `SPEC.md` §9.

## Type conversion table

JVM descriptors are converted to readable types (generic type arguments are erased
and therefore absent):

| Descriptor | Type |
|---|---|
| `Z` `B` `S` `I` `J` `F` `D` `C` | `boolean` `byte` `short` `int` `long` `float` `double` `char` |
| `Ljava/lang/String;` | `string` |
| `Ljava/lang/Integer;` / `Boolean` / `Long` / `Double` / `Float` / `Short` / `Byte` | `int` / `boolean` / `long` / `double` / ... |
| `Ljava/lang/Object;` | `object` |
| `Ljava/util/Date;` | `Date` |
| `Ljava/util/List;` / `ArrayList` / `Map` / `HashMap` / `Set` / `Collection` | `List` / `ArrayList` / `Map` / `HashMap` / `Set` / `Collection` |
| `Ljava/math/BigDecimal;` | `BigDecimal` |
| `Ljava/io/InputStream;` | `InputStream` |
| `Lorg/json/JSONObject;` | `JSONObject` |
| `Lcom/epic/Foo;` | `com.epic.Foo` (dotted, as-is) |
| `[Lcom/epic/Foo;` / `[Ljava/lang/String;` / `[[I` | `com.epic.Foo[]` / `string[]` / `int[][]` |

## Multipart

A member whose type is `byte[]` or `InputStream` is transmitted as a binary
multipart part (26 members across 18 records, e.g. `Epic.Mobile.Core.UploadLog.
Request.LogData`, `SaveRecordingChunks.Request.Data`, Consents signature blobs).
Records where any member is multipart carry `"multipart": true`. The rule is
recursive for nested DTOs — consumers should apply it recursively over `type`
references.

## Enums

Only `com.epic.bridges.model.DaemonStatus` is a Java enum inside the catalog; 10
members reference it (8 `Epic.EDI.Mobile.DaemonStatus` constants plus
`CommStatusEnum`/`FilerStatusEnum` on `Epic.EDI.Mobile.InterfaceObject`), and their
`enumValues` are the enum's static constants. Enum types not themselves in the
catalog are left as plain dotted class names.

## Catalog statistics

- 2,851 records; 15,963 member entries.
- kinds: 881 request / 871 response / 1,099 other.
- 115 zero-member records — all genuinely empty DTOs (empty request/response bodies,
  e.g. `OAuth2Login$Request` = `{}` for CreateSession).
- Every property and every non-synthetic field of every ECF-annotated class is
  represented (verified by audit, 0 gaps).

## Data-quality caveats

1. **Duplicate record names (167).** The same record name is declared by several
   classes — versioned twins (`GetOpenIDConnectSettings$V1/V2Response`) and subtype
   families sharing one record (26 `LoadMessageList$Response` variants under
   `Epic.Mobile.InBasket.LoadMessageList.Response`). These are merged: `class` is
   the member-richest class, others go to `aliasClasses`, and members are unioned.
   Merged records therefore show the union of all versions' members.
2. **Genuine wire-name collisions inside one record (4).**
   `Epic.Mobile.Clinical.Diagnosis.SearchDx.Request` (`DxSearchType`, `Mode`),
   `Epic.Mobile.Core.Login.Response` (`ServerInfo`), `Epic.Mobile.OrderEntry.
   OEEntryMed` (`PatientTakingStatus`), `Epic.Mobile.UnifiedDirectory.ContactItem`
   (`ContactType`). The JSON key is ambiguous in the client itself; treat the
   duplicate-typed member as authoritative per actual usage.
3. **Cross-version duplicates by wire name** appear in merged records (1,275
   wire-duplicate pairs) — same key from two versioned classes with different
   `javaMember`/type; payload-compatible.
4. **Void-typed properties (77)** return `V` in the metadata; kept as members with
   `type: "void"` (a deserializer may use them as out-parameters).
5. **141 ECF-annotated classes have no record name** and are excluded (mostly
   legacy facade DTOs — `ClinicalReviewList$Request`, `LoadPreAdminData$*`,
   `StaffMessage`, `Medication`, flowsheet models, `LegacyLogoutRequest`). A SOAP
   facade reimplementation must recover these separately from the SOAP-side
   annotations.
6. Short single-letter class names (e.g. `com.epic.inbasket.model.g`) are the
   client's own internal names and are kept as-is.
