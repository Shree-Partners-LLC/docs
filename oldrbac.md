# Access Management / RBAC – Module Deep‑Dive

> Companion to `CODE_ANALYSIS.md` and `CLIENT_MANAGEMENT.md`.
> Focus: how BTOS decides **who can see and do what** — the role levels,
> the per‑customer menu entitlement tables, the runtime authority object
> and the page‑level gates.

**Short answer to "do we have any RBAC tables?":**
Yes — but BTOS does **not** use a classic `ROLE` / `PERMISSION` / `ROLE_PERMISSION`
schema. Access control is a hybrid of:

1. A **role‑level integer** on the user (`MAST_CLIENT.WEB_ADMIN`).
2. A **per‑customer menu‑entitlement master table** read by `BTM_getTopMenuList`
   (each menu row carries `AVAILABLE` + `ADMIN_ONLY` flags). This is the
   closest thing to a "permissions" table.
3. A few **per‑client level fields** (`ADMISSION_LVL`, `DEPUTY_LVL`).
4. A **runtime, session‑cached** authority object (`RecAuthrity`) computed
   from 1 + 2, which the admin pages check.

---

## 1. Glossary

| Term (JA) | Term (EN) | Meaning in BTOS |
|---|---|---|
| 得意先 | **Customer** | The contracting company (`CUSTOMER_CODE1..4`). Menu entitlements are defined **per customer**. |
| 社員 / クライアント | **Client / User** | An employee row in `MAST_CLIENT`. The login subject whose `WEB_ADMIN` level drives admin access. |
| グループ管理者 | **Group admin** | `WEB_ADMIN` source value `1` → mapped to internal level **7** (broadest). |
| 会社管理者 | **Company admin** | source value `2` → internal level **5**. |
| 管理者 | **Admin** | source value `3` → internal level **3**. |
| 一般ユーザー | **General user** | source value `4` → internal level **1** (self only). |
| メニュー | **Menu / feature** | A functional capability keyed by `MENU_CODE`. Whether it is shown is decided by `AVAILABLE` and `ADMIN_ONLY`. |
| 権限 | **Authority** | The computed `RecAuthrity` flags held in session. |
| 代行レベル | **Deputy level** | `DEPUTY_LVL` — can this client book on behalf of others. |
| 承認レベル | **Admission level** | `ADMISSION_LVL` — approval capability. |

---

## 2. The data behind access control

### 2.1 Role level – `MAST_CLIENT.WEB_ADMIN`

The user's role is a single integer column loaded at login by
`BTM_getLoginUser` and re‑mapped in `DatUser.Login`:

```205:220:App_Code/DatUser.cs
        int iF1WebAdmin = ComFunc.ToInt(dtClient.Rows[0]["WEB_ADMIN"].ToString());
        switch (iF1WebAdmin)
        {
            case 1: // グループ管理者
                l_recUser.WEB_ADMIN = 7;
                break;
            case 2: // 会社管理者
                l_recUser.WEB_ADMIN = 5;
                break;
            case 3: // 管理者
                l_recUser.WEB_ADMIN = 3;
                break;
            case 4: //一般ユーザー
                l_recUser.WEB_ADMIN = 1;
                break;
        }
```

| Stored value | Internal value | Role | Scope |
|---|---|---|---|
| `1` | **7** | Group admin | Entire group (customer codes 1–2). |
| `2` | **5** | Company admin | Within `CUSTOMER_CODE1..4`. |
| `3` | **3** | Admin | Limited admin. |
| `4` | **1** | General user | Self only. |

`blAdminFlg = (recLogin.WEB_ADMIN >= 3)` is the single test that decides
whether `ADMIN_ONLY` menu rows are shown (see `WucTopMenu.ascx.cs`).

### 2.2 Menu entitlement master – read by `BTM_getTopMenuList`

This is the **core RBAC/entitlement table**. It is keyed per customer
(`@CODE1..4`) and returns one row per menu/feature with at least these
columns:

| Column | Meaning |
|---|---|
| `MENU_CODE` (primary key of the returned `DataTable`) | The feature id (see §3 for the catalogue of codes). |
| `AVAILABLE` | `1` = the feature is contracted/enabled for this customer, `0` = hidden. |
| `ADMIN_ONLY` | `1` = only visible to admins (`WEB_ADMIN >= 3`), `0` = all users. |

It is fetched via `ComDB2.GetTopMenuList` → stored proc `BTM_getTopMenuList`:

```38:56:App_Code/ComDB2.cs
    public static DataTable GetTopMenuList(SqlConnection conn, string intCode1, string intCode2, string intCode3, string intCode4)
    {

        int CommandTimeout = ComFunc.ToInt(WebConfigurationManager.AppSettings["COMMAND_TIMEOUT"]);

        SqlCommand cmd = new SqlCommand("BTM_getTopMenuList", conn);
        cmd.CommandType = CommandType.StoredProcedure;
        if (CommandTimeout > 0)
        {
            cmd.CommandTimeout = CommandTimeout;
        }
        cmd.Parameters.Add("@CODE1", SqlDbType.VarChar);
        cmd.Parameters["@CODE1"].Value = intCode1;
```

> The physical table name lives inside the SQL Server stored procedure
> `BTM_getTopMenuList` (not in this repo). From the column usage it is a
> per‑customer **menu/feature master** (commonly named like `WEB_TOP_MENU`
> / `MAST_MENU`). Treat `(CUSTOMER_CODE1..4, MENU_CODE) → {AVAILABLE, ADMIN_ONLY}`
> as the entitlement matrix.

### 2.3 Related entitlement / link tables

| Read via | Purpose |
|---|---|
| `BTM_getWebCustomerMenuLink` | Per‑customer custom external menu links (`LINK_KIND`, `NAME_JPN/ENG`, `LINK_URL`). Company‑specific add‑on menu. |
| `BTM_getTopSpecialArrangeList` | Per‑customer "special arrangement" menu entries (gated by `SPECIAL_ARRANGEMENT_KBN`). |

### 2.4 Per‑client capability levels (`WEB_CLIENT_USER` / `MAST_CLIENT`)

| Field (on `RecClient`) | Source | Meaning |
|---|---|---|
| `WEB_ADMIN` | `MAST_CLIENT.WEB_ADMIN` | Role level (see §2.1). |
| `ADMISSION_LVL` | client record | Approval / admission capability level. |
| `DEPUTY_LVL` | `WEB_CLIENT_USER` | Proxy‑booking (代行) capability — `> 0` required to be added as an agent. |

---

## 3. Menu code catalogue (`ComConst.cs`)

The `MENU_CODE` values referenced against the entitlement table are defined
as `cnsMenuCode*` constants:

| Code | Constant | Feature | Maps to authority flag |
|---|---|---|---|
| 10/11/12/13/14 | `cnsMenuCodeWorld_*` | Overseas trip arrange (ticket/online/other) | — (panel visibility) |
| 20/21/22 | `cnsMenuCodeJapan_*` | Domestic trip arrange | — |
| 30 | `cnsMenuCodeListTrip` | Trip list | — |
| 40 | `cnsMenuCodeProfile` | Personal data management | — |
| 50 | `cnsMenuCodeAdminListArrange` | Admin arrange list | `ArrangeStatusList3` |
| 60 | `cnsMenuCodeAdminListUser` | User list (admin) | `UserList` |
| 70 | `cnsMenuCodeAdminNews` | Internal notices | `InformationList` |
| 80 | `cnsMenuCodeCustomer_User` | Company web‑user config | `WebUserConfig` → `AdminMenu` |
| 90 | `cnsMenuCodeCustomer_Approve` | Approval‑route management | `ApproveRouteList` → `AdminMenu` |
| 100 | `cnsMenuCodeCustomer_Profile` | Profile bulk upload | `ProfileUpload` → `AdminMenu` |
| 110 | `cnsMenuCodeCustomer_CostCentor` | Cost‑centre upload | `CostCenterUpload` → `AdminMenu` |
| 120 | `cnsMenuCodeCustomer_ClientAgent` | Proxy‑booker management | `ClientAgentList` → `AdminMenu` |
| 130–220, 350 | `cnsMenuCodeReport_*` | Report downloads | `DownloadReport` |
| 230 | `cnsMenuCodeProfileControl` | Profile control matrix | `ProfileControl` |
| 240–330 | `cnsMenuCodeOther*` / external links | External links / campaigns | — |
| 290 | `cnsMenuCodeClick` | 機器管理 (CLICK) | — |
| 360 | `cnsMenuCodeRakutenPackage` | Rakuten package | — |

> Setting `AVAILABLE` for the **AdminMenu group** (codes 80/90/100/110/120)
> is what raises `recAuthrity.AdminMenu = true`; each sub‑feature also sets
> its own flag.

---

## 4. Runtime authority object – `RecAuthrity`

`RecAuthrity` (`App_Code/RecAuthrity.cs`) is **not** a table; it is the
in‑memory projection of the entitlement matrix for the current user. It is
a flat set of booleans:

```10:23:App_Code/RecAuthrity.cs
public class RecAuthrity
{
    public bool ArrangeStatusList3 { get; set; }
    public bool UserList { get; set; }
    public bool InformationList { get; set; }
    public bool AdminMenu { get; set; }
    public bool WebUserConfig { get; set; }
    public bool ApproveRouteList { get; set; }
    public bool ProfileUpload { get; set; }
    public bool CostCenterUpload { get; set; }
    public bool ClientAgentList { get; set; }
    public bool DownloadReport { get; set; }
    public bool ProfileControl { get; set; }
```

### 4.1 How it is computed

`com/WucTopMenu.ascx.ChangeViewAvailable()` runs once per top‑menu render:

1. Loads the user (`RecClient`) from session.
2. `blAdminFlg = (recLogin.WEB_ADMIN >= 3)`.
3. `dtMenu = GetTopMenuList(CUSTOMER_CODE1..4)` (the entitlement table).
4. For each menu code: a feature is enabled when
   `row.AVAILABLE == 1` **and** (`ADMIN_ONLY == 0` **or** `blAdminFlg`).
5. Each enabled admin feature sets the matching `RecAuthrity` flag, e.g.:

```279:288:com/WucTopMenu.ascx.cs
        row = dtMenu.Rows.Find(ComConst.cnsMenuCodeAdminListUser);
        if (row == null || (int)row["AVAILABLE"] == cnsAvailableNo || ((row["ADMIN_ONLY"].ToString() == cnsAdminOnlyYes && blAdminFlg == false)))
        {
            pnlAdminListUser.Visible = false;
        }
        else
        {
            intAdminMenuCount++;
            recAuthrity.UserList = true;
        }
```

6. Finally it is cached in session:

```601:601:com/WucTopMenu.ascx.cs
        ComSession.SetAuthrity(recAuthrity);
```

### 4.2 Where it is stored

```148:160:App_Code/ComSession.cs
    public static RecAuthrity GetAuthrity()
    {
        if (System.Web.HttpContext.Current.Session["Authrity"] != null)
        {
            return (RecAuthrity)System.Web.HttpContext.Current.Session["Authrity"];
        }
        return new RecAuthrity();
    }

    public static void SetAuthrity(RecAuthrity auth)
    {
        System.Web.HttpContext.Current.Session["Authrity"] = auth;
    }
```

So the "permission set" lives in `Session["Authrity"]` for the duration of
the session.

---

## 5. Page‑level gates (the enforcement points)

Every admin page re‑checks the relevant `RecAuthrity` flag and redirects
away if false:

| Page | Required flag | Source line |
|---|---|---|
| `apl/AdminMenu.aspx` | `AdminMenu` | `AdminMenu.aspx.cs:57` |
| `apl/UserList.aspx` | `UserList` | `UserList.aspx.cs:68` |
| `apl/ProfileEdit.aspx` (add mode) | `UserList` | `ProfileEdit.aspx.cs:271` |
| `apl/ClientAgentList.aspx` | `ClientAgentList` | `ClientAgentList.aspx.cs:118` |
| `apl/ApproveRouteList.aspx` | `ApproveRouteList` | `ApproveRouteList.aspx.cs:41` |
| `apl/ApproveRouteEdit.aspx` | `ApproveRouteList` | `ApproveRouteEdit.aspx.cs:29` |
| `apl/ClientList3.aspx` | `ApproveRouteList` | `ClientList3.aspx.cs:29` |
| `apl/ProfileUpload.aspx` | `ProfileUpload` | `ProfileUpload.aspx.cs:121` |
| `apl/CostCenterUpload.aspx` | `CostCenterUpload` | `CostCenterUpload.aspx.cs:64` |
| `apl/InformationList.aspx` | `InformationList` | `InformationList.aspx.cs:40` |
| `apl/InformationEdit.aspx` | `InformationList` | `InformationEdit.aspx.cs:32` |
| `apl/WebUserConfig.aspx` | `WebUserConfig` | `WebUserConfig.aspx.cs:37` |
| `apl/DownloadReport.aspx` | `DownloadReport` | `DownloadReport.aspx.cs:41` |
| `apl/ProfileControl.aspx` | `ProfileControl` | `ProfileControl.aspx.cs:39` |
| `apl/ArrangeStatusList3.aspx` (Admin mode) | `ArrangeStatusList3` | `ArrangeStatusList3.aspx.cs:189` |

Typical gate pattern:

```68:71:apl/UserList.aspx.cs
        if (!ComSession.GetAuthrity().UserList)
```

In addition, scope‑level checks (`IsLoginUser(custCode1..4)`) defend against
query‑string tampering — see `CLIENT_MANAGEMENT.md` §6.

---

## 6. End‑to‑end flow

```
Login (BTM_getLoginUser)
   └─ MAST_CLIENT.WEB_ADMIN  ── DatUser.Login maps 1/2/3/4 → 7/5/3/1
                                        │
Top menu render (WucTopMenu)            ▼
   └─ BTM_getTopMenuList(CUSTOMER_CODE1..4)
          returns rows: MENU_CODE, AVAILABLE, ADMIN_ONLY
                                        │
          for each admin menu:          ▼
              enabled = AVAILABLE==1 && (ADMIN_ONLY==0 || WEB_ADMIN>=3)
              → set RecAuthrity.<Flag> = true
                                        │
          ComSession.SetAuthrity(recAuthrity)  → Session["Authrity"]
                                        │
Admin page load                         ▼
   └─ if (!ComSession.GetAuthrity().<Flag>) Response.Redirect(away)
```

---

## 7. Summary of "RBAC" tables & artifacts

| Artifact | Type | Role in access control |
|---|---|---|
| `MAST_CLIENT.WEB_ADMIN` | Table column | The **role** (group/company/admin/general). |
| Menu master behind `BTM_getTopMenuList` | DB table (per customer) | The **permission/entitlement matrix** (`MENU_CODE` × `AVAILABLE`/`ADMIN_ONLY`). Closest thing to a permissions table. |
| Menu link master behind `BTM_getWebCustomerMenuLink` | DB table | Customer custom menu entitlements. |
| Special‑arrange master behind `BTM_getTopSpecialArrangeList` | DB table | Special‑arrangement feature entitlements. |
| `WEB_CLIENT_USER.DEPUTY_LVL`, `ADMISSION_LVL` | Table columns | Per‑client capability levels (deputy booking / approval). |
| `RecAuthrity` / `Session["Authrity"]` | Runtime object | Computed permission set checked by pages. |
| `ComConst.cnsMenuCode*` | Code constants | The catalogue of `MENU_CODE` ids. |

---

## 8. Risks & improvement notes (access‑management specific)

1. **No first‑class role/permission schema** — permissions are spread across
   a single `WEB_ADMIN` integer, a per‑customer menu master, and hard‑coded
   `cnsMenuCode*` constants. Adding a new gated feature requires touching the
   DB master **and** `ComConst` **and** `RecAuthrity` **and** `WucTopMenu`.
   Consider a normalized `ROLE` / `PERMISSION` / `ROLE_PERMISSION` model.
2. **Authority computed only in the top‑menu control** — `RecAuthrity` is
   populated in `WucTopMenu.ChangeViewAvailable()`. If a user reaches an
   admin page without the menu having rendered (e.g. deep link, SSO direct
   entry), `GetAuthrity()` returns a **default all‑false** object, which is
   safe (deny) but can produce confusing redirects. Compute authority at
   login instead.
3. **`WEB_ADMIN >= 3` is the only admin discriminator** — there is no
   separation between "admin" and "company admin" at the page‑gate level;
   the level only differs in data scope. Audit whether level 3 should reach
   every `ADMIN_ONLY` feature.
4. **Entitlement rows trusted blindly** — `AVAILABLE`/`ADMIN_ONLY` come from
   the DB with no server‑side re‑check inside the stored procedures that
   actually mutate data. Add defence‑in‑depth authorization in the write
   procedures.
5. **Session‑cached permissions** — `Session["Authrity"]` is not refreshed
   if a customer's entitlement or the user's `WEB_ADMIN` changes mid‑session.
   Re‑evaluate on privileged actions or shorten the cache lifetime.
