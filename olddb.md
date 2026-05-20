# Client Management – Module Deep‑Dive

> Companion to `CODE_ANALYSIS.md`.
> Focus: how BTOS models, stores, finds, edits and authorises **clients**
> (= individual employees / travellers / approvers / agents) inside a
> corporate **customer** (= 得意先 / company).

This document covers everything a developer or admin needs to understand,
extend or audit the "client manage" surface of BTOS:

1. Domain terms (Customer / Client / Tripper / Agent / Companion / Arranger).
2. The pages that constitute client management.
3. The data classes (`Dat*` + `Rec*`) and the session graph.
4. The `WEB_*` / `CLIENT_*` tables involved and how they relate.
5. The full set of stored procedures used.
6. Authority (role / menu) gating and SSO touch‑points.
7. Typical flows (search → select → assign agent / approver / companion;
   profile edit & history; bulk profile upload; user deletion).
8. Risks & improvement notes specific to client management.

---

## 1. Glossary

| Term (JA) | Term (EN) | Meaning in BTOS |
|---|---|---|
| 得意先 | **Customer** | The contracting **company** (corp code = `WEB_CUSTOMER_USERID`). One customer owns many clients. Identified by 4 customer codes `CUSTOMER_CODE1..4`. |
| 社員 / クライアント | **Client** | An **employee** of a customer. The login subject. Primary key `CODE` in `MAST_CLIENT`. Web login id = `WEB_CLIENT_USERID`. |
| 出張者 | **Tripper / Traveller** | A client when they are the subject of a trip. Same `MAST_CLIENT` row – role is contextual. |
| 代行者 | **Agent** (deputy booker) | A client allowed to make/edit reservations on behalf of another client. Stored in `WEB_CLIENT_USER_AGENT`. Requires `DEPUTY_LVL > 0`. |
| 同行者 | **Companion** | A client that travels together with another client. Stored in `WEB_CLIENT_USER_COMPANION`. |
| 手配者 | **Arranger** | A client connected to a reservation as the requester (see `WEB_CLIENT_ARRANGER`). |
| 承認者 | **Approver** | A client that participates in an approval route (see `CLIENT_APPROVE` / `CLIENT_APPROVE_DETAIL`). |
| 担当 | **Burden / In‑charge** | A free‑form code (`BURDEN_CODE`). |

The same person (`MAST_CLIENT.CODE`) can simultaneously be a tripper, an
approver, an agent and a companion – the role is determined by which
table their `CODE` appears in.

---

## 2. UI surface – Client management pages

All pages live under `apl/` and are protected by session + authority checks
(see §6).

### 2.1 Search / selection (popups)

| Page | Purpose | Server proc | Notes |
|---|---|---|---|
| `apl/ClientList2.aspx` | Generic employee picker popup used by `WucClientInput2/3/4`. Multi‑select. Supports filtering for **deputy / reservation‑agent / companion** flags. | `BTM_getMastClientList2`, optionally `BTM_getClientArranger` | Up to 100 rows (constant `limitRow`). `LIMIT_ROW=101` is used to detect "too many results" and toggle `lbSearches`. Returns codes/names to opener via JS. |
| `apl/ClientList3.aspx` | Approval‑route variant of the picker (single select). Requires `RecAuthrity.ApproveRouteList`. | `BTM_getMastClientList3` | Same data shape, different stored procedure for the "approver list" use case. |
| `com/WucClientInput2.ascx` / `3` / `4` | Reusable input controls that open the picker. They embed the calling page's input id (`hdnCtrlClientCode`/`hdnCtrlClientName`) so the JS callback writes back the selection. |  |  |
| `apl/AutoComplete.aspx` | Autocomplete JSON endpoint for free‑text client / station / airport lookups. | `BTM_getAutoCompleteValueList` |  |

### 2.2 Profile / per‑client editing

| Page | Purpose | Key stored procs |
|---|---|---|
| `apl/ProfileEdit.aspx` (10 600+ LOC) | The **giant profile editor** for the logged‑in client – personal info, secret questions, passport, visa, credit card, FFP cards, hotel & rent‑a‑car cards, custom report items, approval routes, deputy approvers, default settings. | `BTM_getWebClientUser`, `BTM_getClientRequest`, `BTM_getClientPassport`, `BTM_getClientVisa`, `BTM_getClientCard`, `BTM_getClientFFP`, `BTM_getClientHotelFFP`, `BTM_getClientRentCarCard`, `BTM_getClientConnect`, `BTM_getClientOccupation`, `BTM_getClientUseReportItem`, `BTM_getWebClientSecretQuestion`, plus history+insert procs (see §5) |
| `apl/ProfileControl.aspx` | Admin‑level profile control (required‑fields matrix per customer). | `BTM_getCustomerControlWebProfile` |
| `apl/ProfileUpload.aspx` | CSV bulk upload of clients & their reports. | `BTM_getClientOccupation2`, `BTM_getClientReportItemLength` |
| `apl/PrivacyProfile.aspx` | Consent / withdrawal of personal data sharing. | `BTM_updMastClientPrivacyAgreeDate`, `BTM_updMastClientPrivacyDisagreeDate` |
| `apl/TripperInfoEdit.aspx` (3 700+ LOC) | Per‑traveller info for a specific reservation – inherits report items from the client profile. | `BTM_getClientUseReportItem_Traveler`, `BTM_getClientOccupation` |
| `apl/CustomerFreeArea.aspx` | Customer‑defined free‑form area for client comments. |  |
| `apl/CustomerReportItemList.aspx` | List of report items required for a customer. | `BTM_getCustomerReportItem` |

### 2.3 Relationship management

| Page | Purpose | Key stored procs |
|---|---|---|
| `apl/ClientAgentList.aspx` | Define **代行予約 (proxy booking)** agents for a given traveller. Requires `RecAuthrity.ClientAgentList`. CRUD on `WEB_CLIENT_USER_AGENT`. Reloads the logged‑in user after save. | `BTM_getMastClient`, `BTM_getWebClientUserAgent`, `BTM_insWebClientUserAgent`, `BTM_delWebClientUserAgent` |
| `apl/ClientApproveRouteEdit.aspx` | Maintain a **per‑client** approval route (overrides company default). | `BTM_getClientApproveRouteList`, `BTM_insClientApprove`, `BTM_insClientApproveDetail` |
| `apl/ApproveRouteEdit.aspx` / `apl/ApproveRouteList.aspx` | Maintain **customer‑level** (company) approval routes. Requires `RecAuthrity.ApproveRouteList`. |  |
| `apl/ApproverSelect.aspx`, `apl/ApproverRouteSelect.aspx` | Pick approvers for an individual reservation. |  |

### 2.4 Admin operations

| Page | Purpose | Key stored procs |
|---|---|---|
| `apl/UserList.aspx` | Admin list of all clients for the customer; supports soft‑delete. Requires `RecAuthrity.UserList`. | `BTM_updWebClientUserDelFlg` (and the various list / mail follow‑up procs) |
| `apl/AdminMenu.aspx` | Landing page for admin actions. Requires `RecAuthrity.AdminMenu`. |  |
| `apl/WebUserConfig.aspx` | Per‑customer web‑user configuration. Requires `RecAuthrity.WebUserConfig`. |  |
| `apl/CostCenterUpload.aspx` | CSV upload of cost centres referenced by clients. Requires `RecAuthrity.CostCenterUpload`. |  |
| `apl/DownloadReport.aspx` | Download client / reservation reports. Requires `RecAuthrity.DownloadReport`. |  |

### 2.5 Authentication around clients

| Page | Purpose |
|---|---|
| `apl/Login.aspx` / `apl/LoginE.aspx` | Standard CorpCode + UserID + Password login. |
| `apl/LoginForProfile.aspx` | Login flow specifically for profile administration. |
| `apl/UserInfoSettingLogin.aspx` | Login + first‑time secret question registration. |
| `apl/IdentityVerification.aspx` | Verifies secret questions during reset. |
| `apl/PasswordReset.aspx` / `apl/PasswordSetting.aspx` | Forgot password + one‑time‑password flow. |
| `apl/SubscriptionFinish.aspx` | Final step after self‑registration (provisional `CODE = -1` becomes a real client). |

---

## 3. Data layer

```
RecClient (≈ 2 900 lines, [Serializable])
   └─ All client fields, derived flags, validation methods
DatUser    (App_Code/DatUser.cs)
   ├─ Login / OneTimeLogin                → BTM_getLoginUser / BTM_ExistOnetimePassword
   ├─ LoadMastClient / LoadCLOccupation
   ├─ LoadDeputyApprovers                 → BTM_getDeputyApproveRequestMembers
   ├─ LoadClientHrRelation                → BTM_getClientHrRelationByClientCode
   ├─ LoadClientUserAgent (代行)           → BTM_getWebClientUserAgent
   ├─ LoadClientUserCompanion (同行)       → BTM_getWebClientUserCompanion
   ├─ LoadClientMailAddress                → BTM_getClientMailAddress
   ├─ LoadTopMenu                          → BTM_getTopMenuList (drives RecAuthrity)
   └─ LoadOnetime / SaveOnetimePassword
DatClient  (App_Code/DatClient.cs)
   ├─ DefRecClient()                       — factory with safe defaults
   └─ GetClient(code)                      → BTM_getMastClient + BTM_getClientRequest
RecAuthrity (App_Code/RecAuthrity.cs)
   └─ Per‑menu boolean flags (ArrangeStatusList3, UserList,
      InformationList, AdminMenu, WebUserConfig, ApproveRouteList,
      ProfileUpload, CostCenterUpload, ClientAgentList,
      DownloadReport, ProfileControl)
```

`RecClient` is the **god‑object** of client management. Its fields mirror
the rows of (at least) the following tables, grouped by source comment in
the file:

| Source table | Selected fields |
|---|---|
| `MAST_CLIENT` | `CODE`, `NAME_CODE`, `BURDEN_CODE`, `ENGLISH_NAME`, `JAPAN_NAME`, `ANK_NAME` (katakana), `ENGLISH_ADDRESS`, `JAPAN_ADDRESS`, `TEL_NO`, `SEAT`, `SEAT_DOM`, `PORTABLE_NO`, `OVERSEAS_PORTABLE_NO`, `RAIL_REQUEST_SEAT`, `RAIL_SMOKING`, `HOTEL_SMOKING`, `MEAL_REQ`, `SEX`, `BIRTH_DATE`, `CUSTOMER_CODE1..4`, `CUSTOMER_EMPLOYEE_CODE`, `MAIL_ADDRESS1..3`, `APPROVE_NO`, `WEB_USER_ID`, `WEB_PASSWORD`, `WEB_ADMIN`, `PW_LAST_UPDATE`, `PW_WARNING` |
| `MAST_CUSTOMER_DETAIL` | `JAPAN_NAME_BRANCH1`, `JAPAN_NAME_DIVISION1/2`, `JAPAN_NAME_SECTION1/2`, English equivalents, `PROFILE_CHANGE_MAIL` |
| `CUSTOMER_BTM` | `WEB_CORP_CODE`, `WEB_CORP_PASSWORD`, `POLICY_KBN`, `WF_KBN`, `ARRANGER_KBN`, `SPECIAL_ARRANGEMENT_KBN`, `BILINGUAL_KBN`, `FLIGHT_PLUS_KBN`, `HOTEL_PLUS_KBN`, `HOTEL_MAST_KBN`, `CLICK_KBN`, `MAIL_KBN`, `LINK_KBN`, `TRAVELER_KBN`, `TRAVELER_MAX`, `TRAVELER_MAX_DOM`, `APPROVAL_MAX` |
| `CUSTOMER_BTM_APPROVAL` | `OUT_APPROVAL_KBN`, `OUT_APPROVAL_TIMING`, `INT_TRIP_APPROVAL_KBN/TIMING`, `DOM_TRIP_APPROVAL_KBN/TIMING`, `TRIP_(CREATE|UPDATE)_APPROVAL_KBN`, `OUT_TRIP_(CREATE|UPDATE)_APPROVAL_KBN` |
| `CLIENT_REQUEST` | Seat/smoking/meal preference (loaded by `DatClient.GetClient`) |
| `CLIENT_PASSPORT`, `CLIENT_VISA`, `CLIENT_CARD`, `CLIENT_FFP`, `CLIENT_HOTEL_FFP`, `CLIENT_RENT_CAR_CARD` | Travel‑document and card lists, edited via `ProfileEdit.aspx` |
| `WEB_CUSTOMER_USER` (corp credentials) | `RACCO_CORPCD/ID/PASSWD`, `JALAN_PARTNER_CODE`/`AGENT_CODE`, `JYALAN_CORPCD`, `HRS_CORPCD`, `JALOL_2_*`, `ANABIZ_*`, `JALONLINE_*`, `ANADESK_*`, `SFBIZ_*`, `STARFLYER_ACCESS_CD/PASSWD`, `EXPRESS_*` |
| `WEB_CLIENT_USER` (user credentials) | `WEB_CLIENT_USERID`, `WEB_CLIENT_USERPW`, `RACCO_USERID/PW`, `HRS_USERNAME`, supplier admin IDs, deputy levels, etc. |
| `WEB_CLIENT_USER_AGENT` | `IsReservation()`, `IsCompanion()` helpers query loaded lists |
| `WEB_CLIENT_USER_COMPANION` | Companions per client |
| `CLIENT_HR_RELATION` | HR org relationship per client |
| `CLIENT_REPORT_ITEM` / `CLIENT_USE_REPORT_ITEM` | Custom per‑customer report items, mandatory‑flag checks |
| `WEB_CLIENT_SECRET_QUESTION` | 3 secret questions for self‑service reset |
| `WEB_CLIENT_PASSWORD_HISTORY` | Rotation check |
| `WEB_CLIENT_VISA_HISTORY`, `WEB_CLIENT_CREDIT_CARD_HISTORY`, `WEB_CLIENT_FFP_HISTORY`, `WEB_CLIENT_HOTEL_CARD_HISTORY`, `WEB_CLIENT_RENT_CAR_CARD_HISTORY` | Append‑only audit of profile card/visa changes |

### Session graph

After `DatUser.Login`, the session keeps:

```
Session["DatLoginInfo"]            = DatUser            (holds RecClient via .UserInfo)
Session["DatArrange"]              = DatArrange         (current reservation in flight)
Session["SELECTED_APPROVERS"]      = List<RecApprove>
Session["FINAL_APPROVER"]          = RecApprove
Session["SELECTED_APPROVER_CODE_IN_TOPMAIN"] = int
Session["DirectUserInfo"]          = RecClient          (for /api direct login)
Session["LANG"]                    = "J" | "E"
Session["FOR_PROFILE"]             = 0 | 1              (set on profile‑only login)
```

`RecAuthrity` is computed from `BTM_getTopMenuList` after login and
cached via `ComSession.GetAuthrity()`.

---

## 4. Logical schema (client side)

> Names below are inferred from comments / column references in
> `RecClient.cs`, `DatUser.cs`, `DatClient.cs`, `ClientAgentList.aspx.cs`,
> `ProfileEdit.aspx.cs`, etc. They are not the canonical DDL.

```
                   ┌───────────────────────────────────────┐
                   │            MAST_CUSTOMER_DETAIL       │
                   │  (branch / division / section names)  │
                   └───────────────▲───────────────────────┘
                                   │ CUSTOMER_CODE1..4
┌──────────────────┐               │
│   CUSTOMER_BTM   │◄──────────────┤
│ (BTOS per‑corp   │               │
│  flags & limits) │               │
└────────▲─────────┘               │
         │                         │
         │                         │   ┌──────────────────────────────────┐
         │                         └──►│           MAST_CLIENT             │◄────┐
         │                             │  CODE (PK), NAME_CODE,            │     │
         │                             │  ENGLISH/JAPAN_NAME, BIRTH_DATE,  │     │
         │                             │  MAIL_ADDRESS1..3,                │     │
         │                             │  WEB_USER_ID, WEB_PASSWORD,       │     │
         │                             │  WEB_ADMIN, BURDEN_CODE,          │     │
         │                             │  CUSTOMER_CODE1..4,               │     │
         │                             │  CUSTOMER_EMPLOYEE_CODE           │     │
         │                             └──┬────┬────┬────┬────┬────┬───────┘     │
         │                                │    │    │    │    │    │             │
         │     ┌──────────────────────────┘    │    │    │    │    │             │
         │     ▼                               ▼    ▼    ▼    ▼    ▼             │
         │  CLIENT_REQUEST   CLIENT_PASSPORT  CLIENT_VISA  CLIENT_CARD            │
         │  (seat/smoke/                                  CLIENT_FFP              │
         │   rail/meal)                                   CLIENT_HOTEL_FFP        │
         │                                                CLIENT_RENT_CAR_CARD    │
         │                                                                        │
         │  WEB_CLIENT_USER ─────────────► WEB_CLIENT_SECRET_QUESTION              │
         │     │                                                                  │
         │     ├──► WEB_CLIENT_PASSWORD_HISTORY                                    │
         │     ├──► WEB_CLIENT_USER_AGENT      (TRAVELLER_CLIENT_CODE,             │
         │     │                                AGENT_CLIENT_CODE)─────────────────┤
         │     ├──► WEB_CLIENT_USER_COMPANION  (CLIENT_CODE,                       │
         │     │                                COMPANION_CODE) ──────────────────┘
         │     └──► WEB_CLIENT_ARRANGER         (CUSTOMER_USER, CLIENT_USER)
         │
         │  CLIENT_APPROVE  ◄── CLIENT_APPROVE_DETAIL  (per‑client approval route)
         │
         │  CLIENT_REPORT_ITEM / CLIENT_USE_REPORT_ITEM
         │     └─ inherits required items from CUSTOMER_REPORT_ITEM /
         │        MAST_CUSTOMER_DETAIL via CUSTOMER_CODE1..4
         │
         └─► WEB_CUSTOMER_USER (per‑corp supplier credentials)
                ├── WEB_CLIENT_USER             (per‑client supplier credentials,
                │                                 e.g. JALOL_2, ANABIZ, RACCO…)
                ├── WEB_CLIENT_USER_AGENT / COMPANION
                └── audit *_HISTORY tables
```

Key foreign‑key conventions:

- A row in any `CLIENT_*` table is anchored on `MAST_CLIENT.CODE`
  (parameter `@CODE` in the procs).
- A row in any `WEB_CLIENT_*` table is anchored on the natural key
  (`WEB_CUSTOMER_USERID`, `WEB_CLIENT_USERID`) **plus** the four
  `CUSTOMER_CODE1..4` for legacy joins.
- Agent / companion tables use **two** client codes – the subject
  (`TRAVELLER_CLIENT_CODE`) and the relation (`AGENT_CLIENT_CODE`
  or `COMPANION_CODE`).

---

## 5. Stored procedures used by client management

### 5.1 Read

| Stored procedure | Purpose | Called from |
|---|---|---|
| `BTM_getLoginUser` | Login + load client record. | `DatUser.Login` |
| `BTM_getMastClient` | Load a single client by `CODE`. | `DatClient.GetClient`, `ClientAgentList`, `ComFunc`, `ComMail`, `RecApprove` |
| `BTM_getMastClientList2` | Employee search (multi‑select picker), with optional `IS_DEPUTY`, `COMPANION_CODE`, `AGENT_CLIENT_CODE` filters. | `ClientList2.aspx` |
| `BTM_getMastClientList3` | Approval‑route picker variant. | `ClientList3.aspx` |
| `BTM_getCustomerDetail` | Customer (company) details for the logged‑in client. | `ComFunc`, `RecApprove`, `ComMail` |
| `BTM_getCustomerCode` | Resolve customer codes from `WEB_CORP_CODE` + password. | `ComFunc.GetCustomerCode` (used in self‑registration) |
| `BTM_getCustomerControlWebProfile` | "What is mandatory / hidden in the profile" matrix per customer. | `RecClient.IsRequiredInput`, `ProfileControl.aspx` |
| `BTM_getCustomerReportItem` | Customer‑level definition of report fields. | `RecClient`, `ComMail`, `DatArrange`, `CustomerReportItemList.aspx` |
| `BTM_getClientReportItem` | Per‑client report fields (overrides). | `RecClient.IsRequiredInput`, `DatArrange` |
| `BTM_getClientUseReportItem` / `_Traveler` | Report items as used by the traveller version (`TripperInfoEdit`). | `TripperInfoEdit`, `TripRT_EnglishPolicy`, `ReserveLogin`, `ProfileEdit` |
| `BTM_getClientReportItemLength` | Field‑length metadata for CSV upload validation. | `ProfileUpload.aspx` |
| `BTM_getClientArranger` | Arranger list of a client. | `ClientList2.aspx`, `ComFunc` |
| `BTM_getClientRequest` | Seat / smoking / meal preferences. | `DatClient.GetClient`, `DatUser`, `ProfileEdit.aspx` |
| `BTM_getClientPassport` | Passport list. | `ProfileEdit.aspx`, `ComFunc`, `TopMain` |
| `BTM_getClientVisa` | Visa list. | `ProfileEdit.aspx`, `ComFunc`, `DatArrange` |
| `BTM_getClientCard` | Credit card list. | `ProfileEdit.aspx`, `RecClient.IsRequiredInput`, `DatArrange`, `TripRT_EditStay/EditCar` |
| `BTM_getClientFFP` | Frequent Flyer Program cards. | `ProfileEdit.aspx`, `DatArrange` |
| `BTM_getClientHotelFFP` | Hotel loyalty cards. | `ProfileEdit.aspx`, `DatArrange` |
| `BTM_getClientRentCarCard` / `BTM_getClientRentCarCardNo` | Rent‑a‑car corporate cards. | `ProfileEdit.aspx`, `TripRT_EditCar` |
| `BTM_getClientConnect` | Client ↔ corp connection metadata. | `ComFunc`, `ProfileEdit` |
| `BTM_getClientOccupation` / `BTM_getClientOccupation2` / `BTM_getCLOccupation` | Occupation / org info per client. | `ProfileEdit`, `DatArrange`, `ProfileUpload`, `TripperInfoEdit`, `DatUser` |
| `BTM_getClientHrRelationByClientCode` | HR relations (manager / dotted‑line). | `DatUser` |
| `BTM_getClientMailAddress` | All mail addresses of a client. | `DatUser` |
| `BTM_getClientApproveRouteList` | Per‑client approval route. | `ClientApproveRouteEdit` |
| `BTM_getDeputyApproveRequestMembers` | List of clients for whom the current user can act as deputy. | `DatUser` |
| `BTM_getWebClientUser` | Web (online) user record for a client. | `ProfileEdit`, `CytricOnlineLogin`, supplier `TripRT_*` user controls |
| `BTM_getWebClientUserAnaDesk` / `BTM_getWebClientUserAnaBiz` | Supplier‑specific web user (ANA@DESK / ANA Biz). | `TripRT_Selecter`, `ArrangeDetail3`, `UsageStatementPrint` |
| `BTM_getWebClientUserAgent` | Agents who can book for a traveller. | `DatUser`, `ClientAgentList`, `DatArrange` |
| `BTM_getWebClientUserCompanion` | Companions of a client. | `DatUser`, `DatArrange` |
| `BTM_getWebClientPasswordHistory` | Password rotation history. | `ComFunc.CheckPasswordHistory` |
| `BTM_getWebClientSecretQuestion` | Secret Q&A. | `PasswordSetting`, `LoginForProfile`, `IdentityVerification`, `RecClient.IsRequiredInput`, `DatArrange` |

### 5.2 Write

| Stored procedure | Purpose | Called from |
|---|---|---|
| `BTM_insClientVisa` / `BTM_delClientVisa` | Add / remove visa. | `ProfileEdit.aspx` |
| `BTM_insClientCard` / `BTM_delClientCardList` | Add / remove credit card. | `ProfileEdit.aspx` |
| `BTM_insClientFFP` / `BTM_delClientFFPList` | Add / remove FFP card. | `ProfileEdit.aspx` |
| `BTM_insClientHotelFFP` / `BTM_delClientHotelFFP` | Add / remove hotel FFP. | `ProfileEdit.aspx` |
| `BTM_insClientRentCarCard` / `BTM_delClientRentCarCard` | Add / remove rent‑a‑car card. | `ProfileEdit.aspx` |
| `BTM_insWebClientVisaHistory` | Append visa change history. | `ProfileEdit.aspx` |
| `BTM_insWebClientCreditCardHistory` | Append credit card change history. | `ProfileEdit.aspx` |
| `BTM_insWebClientFfpHistory` | Append FFP change history. | `ProfileEdit.aspx` |
| `BTM_insWebClientHotelCardHistory` | Append hotel card history. | `ProfileEdit.aspx` |
| `BTM_insWebClientRentCarCardHistory` | Append rent‑a‑car card history. | `ProfileEdit.aspx` |
| `BTM_insClientApprove` / `BTM_insClientApproveDetail` | Save per‑client approval route. | `ClientApproveRouteEdit.aspx` |
| `BTM_insWebClientUserAgent` / `BTM_delWebClientUserAgent` | Maintain proxy bookers. | `ClientAgentList.aspx`, `DatArrange` |
| `BTM_insWebClientUserCompanion` / `BTM_delWebClientUserCompanion` | Maintain travel companions. | `DatArrange` |
| `BTM_updWebClientUserDelFlg` | Soft‑delete a client. | `UserList.aspx` |
| `BTM_updMastClientPrivacyAgreeDate` / `BTM_updMastClientPrivacyDisagreeDate` | Privacy consent timestamps. | `PrivacyProfile.aspx` |
| `BTM_SaveOnetimePassword` / `BTM_ExistOnetimePassword` | One‑time password issuance and verification. | `DatUser.OneTimeLogin`, `PasswordReset` |
| `BTM_insWebSystemLog` | Audit login / admin operations. | `Login.aspx`, `DatUser`, multiple |

---

## 6. Authorisation model

`RecAuthrity` (computed from `BTM_getTopMenuList` after login, cached in
the user record) gates each admin page:

| Page | Required flag |
|---|---|
| `apl/AdminMenu.aspx` | `AdminMenu` |
| `apl/UserList.aspx` | `UserList` |
| `apl/ClientAgentList.aspx` | `ClientAgentList` |
| `apl/ApproveRouteList.aspx` / `apl/ClientList3.aspx` | `ApproveRouteList` |
| `apl/ProfileUpload.aspx` | `ProfileUpload` |
| `apl/CostCenterUpload.aspx` | `CostCenterUpload` |
| `apl/InformationList.aspx` | `InformationList` |
| `apl/WebUserConfig.aspx` | `WebUserConfig` |
| `apl/DownloadReport.aspx` | `DownloadReport` |
| `apl/ProfileControl.aspx` | `ProfileControl` |
| `apl/ArrangeStatusList3.aspx` | `ArrangeStatusList3` |

`WEB_ADMIN` levels (after mapping in `DatUser.Login`):

- **7** Group admin – broadest scope (entire group of customer codes 1–2).
- **5** Company admin – within `CUSTOMER_CODE1..4`.
- **3** Admin – limited.
- **1** General user – self only.

Most management pages also re‑verify `IsLoginUser(custCode1..4)` against
the session so that direct‑link tampering of `CustCode*` query params
cannot escalate scope (see `ClientList2.IsLoginUser`).

Every page begins with:

```c#
try { ComSession.SessionErrExCheck((DatUser)Session["DatLoginInfo"]); }
catch { Response.Redirect("~/SessionClose.aspx"); }
```

and many call `Response.Cache.SetCacheability(NoCache)` in `Page_PreRender`.

---

## 7. Key flows

### 7.1 Searching for a client (popup)

1. Page A embeds `<uc:WucClientInput2 …>` (or `…3` / `…4`).
2. The user clicks the button → JS `wrtClient(...)` opens
   `ClientList2.aspx?Code=…&hdnName=…&parentClickBtm=…&LoginCode=…&CustCode1..4=…`.
3. `ClientList2` validates `CustCode1..4` against the session
   (`IsLoginUser`), then calls `BTM_getMastClientList2`.
4. The user ticks rows + clicks **Select** → `btnSelect_Click`
   serialises selected codes/names into JS and writes them back to the
   opener's hidden fields, then fires `parentClickBtm.click()` to
   trigger Page A's postback.

### 7.2 Adding a proxy booker (代行予約)

`apl/ClientAgentList.aspx.cs`:

1. **Select traveller** via `WucClientInput4` → `btnShowTraveller_Click` →
   `BTM_getMastClient` + `BTM_getWebClientUserAgent` → fills
   `AgentList` (in `ViewState`).
2. **Add an agent** via `WucClientInput_2` → `BtnAddAgent_Click`:
   - rejects duplicates,
   - rejects the traveller themselves,
   - rejects clients with `DEPUTY_LVL == 0`.
3. **Delete** marks `UPDATE_FLG = -1` for existing rows or removes
   unsaved entries.
4. **Save** (`ibtEntry_Click`) opens a transaction:
   - `BTM_delWebClientUserAgent` for rows where `IS_ENTRY && UPDATE_FLG = -1`
   - `BTM_insWebClientUserAgent` for rows where `!IS_ENTRY && UPDATE_FLG = 1`
   - `clsCon.Commit()` then refresh `DatLoginInfo` via `DatUser.Login`
     (re‑loads relationships) and redirect to `AdminMenu.aspx`.

### 7.3 Editing your profile

`apl/ProfileEdit.aspx` loads everything via the read procs in §5.1 and
saves via the write procs in §5.2. Each card/visa change is **double‑written** –
once into the canonical `CLIENT_*` table (`BTM_insClient*` / `BTM_delClient*`)
and once into the corresponding `WEB_CLIENT_*_HISTORY` table for audit.
`RecClient.IsRequiredInput()` is the post‑edit validator – it consults
`BTM_getCustomerControlWebProfile` to decide which fields are mandatory
for the customer, and short‑circuits to false with a structured
`NLogService.PrintProcLog` entry listing what was missing.

### 7.4 Bulk profile upload

`apl/ProfileUpload.aspx` parses a CSV against the field‑length matrix
returned by `BTM_getClientReportItemLength`. It is gated by
`RecAuthrity.ProfileUpload`. Errors are surfaced row‑by‑row.

### 7.5 Soft‑deleting a user

`apl/UserList.aspx` calls `BTM_updWebClientUserDelFlg`. Pending
reservations attributed to the user trigger mail notifications via
`mailTargetRequestList` / `mailTargetDeclineList` (see `UserList.aspx.cs`).

### 7.6 Privacy consent

`apl/PrivacyProfile.aspx` writes the agree/disagree timestamps with
`BTM_updMastClientPrivacyAgreeDate` / `…DisagreeDate`. Subsequent logins
short‑circuit to this page until consent is given.

---

## 8. Risks & improvement notes (client‑management specific)

1. **Direct‑link tampering** – many pickers receive `CustCode1..4` and
   `LoginCode` via query string. The only protection is
   `IsLoginUser()`. Add server‑side authorisation in the stored
   procedures themselves (defence in depth).
2. **Stored XSS via picker callback** – `ClientList2.btnSelect_Click`
   composes JavaScript by string‑concatenating `strCodes`/`strNames`
   straight from grid hidden fields. The grid values come from the DB,
   but if a client's `JAPAN_NAME` contains `'` or `<script>`, the
   opener's hidden field receives the raw string and any subsequent
   `innerHTML` use will execute. Apply `ComFunc.HtmlEncode` /
   `HttpUtility.JavaScriptStringEncode` consistently.
3. **`RecClient` god‑object** – ≈ 2 900 lines mixing data, validation,
   masters lookup and per‑supplier credentials. Split into smaller
   aggregates (e.g. `ClientProfile`, `ClientCards`, `ClientSecurity`,
   `ClientSupplierIds`). This will also de‑couple `IsRequiredInput`
   from `ComConst.PROFILE_CODE_*` knowledge.
4. **ViewState payload** – `ClientAgentList` stores `List<RecClient>`
   in `ViewState`. With supplier credentials inside `RecClient`, this
   means encrypted‑but‑round‑trippable PII flows through every
   postback. Consider storing only the minimal projection
   (`CODE`, `JAPAN_NAME`, `ENGLISH_NAME`, `MAIL_ADDRESS`) and pulling
   the rest on demand.
5. **`BTOS_USER` value** – `BTOS_<CORP>_<CODE>` is used as the audit user
   (passed as `@USER` to `BTM_insWebClientUserAgent`, etc.). Make sure
   the receiving procedure logs (DB‑side `created_by`/`updated_by` columns).
6. **Reload after save** – `ClientAgentList.ibtEntry_Click` calls
   `DatUser.Login(pwd, …)` *with the hashed password from session*.
   If a user’s password is rotated mid‑session this will silently
   redirect to `SessionClose.aspx`. Replace with a dedicated
   `DatUser.Reload(userCode)` method that does not need the password.
7. **Hard‑coded "100" page size** – `ClientList2.limitRow` is a magic
   number; `UserList` uses `GvPageSize_UserList` from `web.config`.
   Centralise.
8. **Double‑write history** – the pattern of writing
   `CLIENT_CARD` then `WEB_CLIENT_CREDIT_CARD_HISTORY` is not
   transactional in `ProfileEdit.aspx`. Wrap each card/visa save in
   `ComDb.BeginTrans()` / `Commit()` to avoid drift between the
   canonical table and its audit log.

---

## 9. Quick recipes

### 9.1 Add a new boolean preference to every client

1. Add column to `MAST_CLIENT` (DB).
2. Add property + backing field to `RecClient`.
3. Map it in `DatUser.Login` (read) and `DatClient.DefRecClient` (default).
4. Extend `BTM_getMastClient` / `BTM_getLoginUser` to return it and
   `BTM_setMastClient` / similar to persist.
5. Render in `apl/ProfileEdit.aspx` (and `apl/TripperInfoEdit.aspx`
   if relevant).

### 9.2 Add a new client‑level role flag (e.g. "auditor")

1. Extend `MAST_CLIENT` + `BTM_getTopMenuList`.
2. Add a property to `RecAuthrity` and toggle it from `DatUser` when
   the menu row is found.
3. Gate the relevant page (e.g. `apl/AuditList.aspx`) with
   `if (!ComSession.GetAuthrity().AuditList) Response.Redirect(...)`.

### 9.3 Surface a new supplier ID per client

1. Add `<supplier>_USERID/PASSWD` columns to `WEB_CLIENT_USER`.
2. Add properties to `RecClient` (mimic the JALOL_2 / ANABIZ patterns).
3. Load in `DatUser.Login` and `DatClient.DefRecClient`.
4. Show & save in `apl/ProfileEdit.aspx` under the supplier panel.
5. Use in a new `apl/TripRT_<Supplier>.ascx` control and the
   `apl/<Supplier>OnlineLogin.aspx` SSO entry.

### 9.4 Adjust mandatory‑profile rules

Edit the customer‑level matrix returned by
`BTM_getCustomerControlWebProfile` (`PROFILE_CODE`, `DISPLAY_FLG`,
`REQUIRED_FLG`). No code change needed – `RecClient.IsRequiredInput`
will pick the new rule up on the next save.
s
