# CPOS Code Review

**Files reviewed:**
- `cpos_backend_1003_v1.txt` — Google Apps Script backend (~5,384 lines)
- `cpos_frontend_1003_v1.txt` — Single-file HTML/CSS/JS frontend (~12,719 lines)

---

## 1. Strengths

| # | Area | Description | Location |
|---|------|-------------|----------|
| S1 | Error handling | Consistent `try/catch` wrapping in all backend public-API functions; errors are logged and returned as `{ ok: false, error: ... }` rather than thrown to the caller | Backend: `getCPOSLiteTables`, `saveCPOSLiteAssessment`, `getAdvancedQuestionnaire`, etc. |
| S2 | Auto-initialisation | When required Google Sheets are missing or empty, the system automatically calls `INIT_LITE_TABLES()` / `INIT_QUESTIONNAIRE_SHEETS()` rather than failing silently | Backend: `getCPOSLiteTables`, `getCPOSQuestionnaireConfig` |
| S3 | Backward-compatibility | Public-API wrappers (`getCPOSData`, `evaluateCPOS_v2`, `srvr_getCPOSLiteTables`) preserve older calling conventions so upgrades don't break existing deployments | Backend: lines ~1490–1510 |
| S4 | Fuzzy column matching | `cpos_canonLiteBoundsRow_`, `cpos_pickField_` and `normalizeQuestionnaireFields` accept multiple casing/spacing variants of column names (e.g. `Variable_Name`, `VariableName`, `Variable Name`), making the system resilient to minor spreadsheet formatting differences | Backend: ~2050–2200 |
| S5 | Signal pipeline | A clean, ordered pipeline (Option → Derivation → Inference → Composite → Mapping → Routing → Probe → Checkpoint → Gate) processes each questionnaire answer with clear separation of concerns | Backend: `processAdvancedAnswer` |
| S6 | Input sanitisation | `escapeHtml` and `adminEscapeHtml` are used in the vast majority of `innerHTML` assignments in the frontend, significantly reducing XSS surface | Frontend: lines 2519–5784 |
| S7 | Bilingual support | Full English/Hindi translations for all UI strings, with language selection persisted in `CPOSL.language` | Frontend: `UI_TRANSLATIONS`, ~5790 |
| S8 | Data-integrity scoring | `calculateQualityScore` and `Avg_Confidence` fields give each saved assessment a computed quality indicator, useful for downstream analytics | Backend: `saveCPOSLiteAssessment`, `saveCPOSQuestionnaireSession` |
| S9 | UUID-based IDs | All assessments, sessions, drafts and responses use `Utilities.getUuid()` to generate unique IDs, preventing key collisions | Backend: `saveCPOSAssessment`, `cpos_upsertDraft_`, `saveCPOSSession` |
| S10 | TSV/CSV recovery | `cpos_getTableFast_` detects and handles the common case where a whole table was pasted as a single-column TSV or CSV inside a Google Sheet | Backend: ~610–635 |
| S11 | Comprehensive diagnostics | `DIAGNOSE_CPOS_ISSUE`, `DEBUG_FullQuestionnaireCheck`, `TEST_VERIFY_SETUP`, and related functions give administrators actionable step-by-step guidance when something goes wrong | Backend: ~3050–3500 |
| S12 | Responsive layout | CSS uses CSS custom properties (design tokens), media queries, and a collapsible sidebar so the app adapts to different viewport sizes | Frontend: CSS root vars, `.sidebar.collapsed`, `@media (max-width: 768px)` |
| S13 | Pathway eligibility | Server-side eligibility logic guards each carbon-credit pathway with explicit crop-type and practice-baseline checks (e.g. AWD only for continuously-flooded rice), preventing incorrect pathway recommendations | Backend: `evaluateCPOS_Lite_v1`, ~1240–1340 |
| S14 | Condition evaluation engine | `_cposEvalCondition_` supports 13 operators (EQUALS, IN, GT, LTE, RANGE, IS_SET …) providing a flexible, data-driven rule engine for the questionnaire | Backend: ~4045–4110 |

---

## 2. Weaknesses

| # | Area | Description | Location |
|---|------|-------------|----------|
| W1 | Monolithic file | The entire frontend is 12,719 lines in a single HTML file mixing HTML, CSS, and JavaScript. This makes the code very hard to maintain, test, or review independently | `cpos_frontend_1003_v1.txt` |
| W2 | Business-logic duplication | Eligibility rules, signal-to-class mapping, and derivation logic are implemented independently in both the backend (`evaluateCPOS_Lite_v1`) and the frontend (`applyDerivationsForQuestion`, `FALLBACK_DERIVATION_RULES`). A change in one place does not automatically update the other | Backend: ~1200–1430; Frontend: ~8000–8200 |
| W3 | No server-side caching | Every call to `getCPOSLiteTables` or `getCPOSQuestionnaireConfig` reads all sheets from Sheets API from scratch. For large question banks this will be slow and may hit Apps Script quota limits | Backend: `getCPOSData_Lite`, `getCPOSQuestionnaireConfig` |
| W4 | Mixed variable declarations | `var`, `let`, and `const` are mixed throughout the frontend JavaScript. Older `var` declarations are function-scoped and can cause subtle hoisting bugs | Frontend: throughout |
| W5 | Global state | All frontend state lives in large global objects (`CPOSL`, `ADVANCED`, `window.AdminQE`). There are no module boundaries, so any script tag could accidentally overwrite state | Frontend: `CPOSL = {...}`, `var ADVANCED = {...}` |
| W6 | Production `console.log` spam | Dozens of `console.log` statements (and `Logger.log` in backend) are left in production code paths, degrading browser developer-tools usability and leaking internal data structures | Frontend: throughout; Backend: throughout |
| W7 | No frontend unit tests | There is no test framework or test file for any frontend logic. Only backend `TEST_*` and `DEBUG_*` functions exist, and those must be run manually in the Apps Script IDE | Both files |
| W8 | Inline styles overload | Hundreds of lines of pixel-by-pixel inline `style=` attributes are used inside `innerHTML` templates. This makes theming or accessibility changes very difficult | Frontend: ~4290, ~6465, ~10106, etc. |
| W9 | `Session.getActiveUser()` unreliable | `Session.getActiveUser().getEmail()` returns an empty string when the web-app is accessed by anonymous (non-Google-authenticated) users, yet the result is stored directly in `Created_By` without any guard or fallback label | Backend: `saveCPOSLiteAssessment`, `saveCPOSSession`, `saveCPOSQuestionnaireSession` |
| W10 | Comment-style inconsistency | Back end mixes JSDoc, inline `//`, and bare `/* */` blocks inconsistently; the CSS section of the frontend uses `//` single-line comments (see UI Issues #U1) | Both files |

---

## 3. Bugs / Potential Bugs

| # | Severity | Description | Location | Fix |
|---|----------|-------------|----------|-----|
| B1 | **High** | **`SIGNAL_VALUE_NORMALIZATIONS` duplicate key `'NONE'` **: The object literal defines `'NONE': 'NONE'` (in the Tillage section) and then later `'NONE': 'RAINFED'` (in the Irrigation section). In JavaScript the last definition wins, so **every signal whose value is `'NONE'` — including `TILLAGE_TYPE = 'NONE'` for no-till fields — is incorrectly normalised to `'RAINFED'`**. This silently corrupts tillage, organic-amendment, and other signals | Frontend: lines ~8039 & ~8098 | Remove the duplicate and use context-aware normalisation (e.g., separate maps per signal domain, or add a signal-key prefix to the lookup key) |
| B2 | **High** | **`SIGNAL_VALUE_NORMALIZATIONS` duplicate key `'MEDIUM'`**: `'MEDIUM': 'MODERATE'` (Tillage section) is later overridden by `'MEDIUM': 'NEUTRAL'` (Soil pH section). Any signal value of `'MEDIUM'` — regardless of which variable it belongs to — will be normalised to `'NEUTRAL'`, misrepresenting tillage intensity, SOC levels, etc. | Frontend: lines ~8045 & ~8069 | Same fix as B1; context-sensitive normalisation |
| B3 | **High** | **`saveCPOSLiteAssessment` duplicate data**: The row array maps both column index 9 (`Lite_Variables_JSON`) and index 10 (`Selected_Classes_JSON`) to `JSON.stringify(payload.selections || {})`. `Pathway_Results_JSON` should be at index 8 and hold `payload.results`, but index 9 also incorrectly holds `payload.results` (as seen in lines 248–250). `Lite_Variables_JSON` and `Selected_Classes_JSON` end up holding identical data; the actual pathway results are stored twice | Backend: `saveCPOSLiteAssessment`, ~248–250 | Map each column to its semantically correct source field |
| B4 | **Medium** | **AWD eligibility operator-precedence bug**: The condition `mainCrop !== 'RICE_FLOODED' && mainCrop !== 'RICE-FLOODED' && !includes('rice') \|\| includes('aerobic')` is evaluated as `(A && B && C) \|\| D` due to `&&` having higher precedence than `\|\|`. This means any crop that includes the substring `'aerobic'` is always ineligible for AWD, even before checking the other conditions. While the intent may be correct, the lack of explicit parentheses makes the logic fragile and likely to regress on future edits | Backend: `evaluateCPOS_Lite_v1`, ~1252–1254 | Wrap the full condition in explicit parentheses: `(mainCrop !== 'RICE_FLOODED' && mainCrop !== 'RICE-FLOODED' && !includes('rice')) \|\| includes('aerobic')` |
| B5 | **Medium** | **Duplicate `'Group_Code'` key in `normalizeQuestionnaireFields` fieldMap**: The string `'Group_Code': 'GroupCode'` appears twice in the same object literal. JavaScript silently discards the first definition. While both map to the same target, this is a code defect that could mask a real mapping conflict if the duplicate entries were to diverge | Backend: `normalizeQuestionnaireFields`, lines ~1890 & ~1901 | Remove the duplicate entry |
| B6 | **Medium** | **`eval()` in diagnostic function**: `DIAGNOSE_CPOS_ISSUE` uses `eval(funcName)` to check whether a function exists. In certain Google Apps Script execution contexts (e.g., library scripts) this can throw a `ReferenceError` that crashes the diagnostic instead of reporting the missing function. `eval` also bypasses Apps Script's security model | Backend: `DIAGNOSE_CPOS_ISSUE`, line ~3256 | Replace with `typeof globalThis[funcName] === 'function'` or a try/catch around a direct reference |
| B7 | **Medium** | **`reviewConflict` XSS via `violation.ruleId`**: `warningHtml` injects `violation.ruleId` directly into an `onclick` attribute string with no HTML or JS escaping: `onclick="reviewConflict(\'' + violation.ruleId + '\')"`. If `ruleId` contains a single quote, backslash, or `</script>`, an attacker who can influence rule data in the spreadsheet can execute arbitrary JavaScript in the user's browser | Frontend: line ~4111 | Escape `ruleId` with `adminEscapeHtml` before embedding in the attribute, or attach the event listener programmatically via `addEventListener` |
| B8 | **Low** | **`saveCPOSSession` comment vs. actual column count mismatch**: The function docstring says "14 columns" but the row array contains 15 values (the extra `'v1.0'` version field). This will cause the last value to silently overflow into an adjacent column if the sheet has exactly 14 columns | Backend: `saveCPOSSession`, ~400–425 | Align the docstring with actual schema, or add a header-driven mapper like `saveCPOSAssessment` uses |
| B9 | **Low** | **`saveCPOSResponse` swallows errors and returns `ok: true`**: If saving an individual response fails, the catch block logs a warning but still returns `{ ok: true, error: ... }`. Callers that check only `ok` will believe the save succeeded | Backend: `saveCPOSResponse`, ~447–451 | Return `ok: false` in the catch block, or document the intentional silent-failure design |
| B10 | **Low** | **Potential signal/normalization mismatch for AWD**: The frontend normalisation map converts `'AWD' → 'FLOOD_INT'`. If a derivation rule sets `WATER_MANAGEMENT = 'AWD'`, after normalisation the backend receives `'FLOOD_INT'`. The backend's AWD exclusion check tests for `'AWD_MILD'`, `'AWD_SEVERE'`, or substrings including `'awd'` — none of which match `'FLOOD_INT'`. A farmer already practicing AWD could thus be incorrectly marked eligible | Frontend: `SIGNAL_VALUE_NORMALIZATIONS` line ~8101; Backend: `evaluateCPOS_Lite_v1` line ~1259 | Ensure derivation rules output the exact values the backend tests against, or remove the `'AWD' → 'FLOOD_INT'` normalisation entry |

---

## 4. Breakages / Potential Breakages

| # | Severity | Description | Location | Fix |
|---|----------|-------------|----------|-----|
| BR1 | **Critical** | **Hardcoded real Spreadsheet ID in source code**: `const CPOS_SS_ID = '1f3rcNISHiDHChnboN7eMrjSef0LGuGjbwLbUejk5_QE'` is committed as-is. Anyone who reads or forks this code obtains the production spreadsheet ID, which may allow unwanted read access if the spreadsheet has broad sharing settings | Backend: line 82 | Replace with a placeholder value (`'PASTE_YOUR_SPREADSHEET_ID_HERE'`) and use Apps Script Script Properties or the container-bound approach exclusively |
| BR2 | **High** | **`SpreadsheetApp.getActiveSpreadsheet()` returns `null` in standalone scripts**: `cpos_getSS_()` calls `getActiveSpreadsheet()` and trusts the result without a null-check before using it. In a standalone (non-container-bound) web app without a valid `CPOS_SS_ID`, every API call will throw a cryptic `TypeError: Cannot read property ... of null` | Backend: `cpos_getSS_()`, ~85–95 | Add an explicit null check: `if (active) return active;` (the active check is present but the variable is returned immediately without validation) — actually the logic looks correct but the `getActiveSpreadsheet()` can return an object in unexpected states; add defensive `active && active.getId` guard |
| BR3 | **High** | **`cpos_getSheet_()` throws on missing sheets**: If a required sheet (e.g. `CPOS_Q_Sessions`, `CPOS_Q_Responses`) has been deleted or renamed, `cpos_getSheet_` throws an `Error`, propagating all the way up to the caller and breaking the entire save operation. There is no graceful degradation | Backend: `cpos_getSheet_()`, `saveCPOSQuestionnaireSession`, `saveCPOSLiteAssessment` | Wrap critical sheet fetches in try/catch and return a `{ ok: false, error: 'Sheet missing: ...' }` response |
| BR4 | **Medium** | **`Session.getActiveUser().getEmail()` returns empty string for anonymous users**: Deployed web apps with "Anyone" access level return an empty email. This empty string is stored in `Created_By` columns in all assessment tables without any fallback. Reports or dashboards that filter by creator email will miss these records entirely | Backend: `saveCPOSLiteAssessment`, `saveCPOSSession`, `saveCPOSQuestionnaireSession`, `saveCPOSAssessment` | Use `Session.getActiveUser().getEmail() \|\| 'anonymous'` |
| BR5 | **Medium** | **`cpos_getSpreadsheet_()` function exists but is not used for sheet access**: The backend defines a third spreadsheet-access helper (`cpos_getSpreadsheet_()`) that checks a Script Property `CPOS_SPREADSHEET_ID`, but all actual sheet reads/writes go through `cpos_getSS_()` which does not check that property. The Script-Property fallback path is therefore dead code, and administrators who set `CPOS_SPREADSHEET_ID` as a property will be surprised that it has no effect | Backend: `cpos_getSpreadsheet_()` ~2050; `cpos_getSS_()` ~84 | Either remove `cpos_getSpreadsheet_()` or wire it into `cpos_getSS_()` |
| BR6 | **Medium** | **Mobile sidebar has no re-open button**: On viewports ≤ 768 px the sidebar is hidden via `transform: translateX(-100%)`. The CSS defines `.sidebar.mobile-open` to show it, but there is no hamburger/toggle button rendered on mobile. Once the sidebar is dismissed (or on first load) there is no way for mobile users to switch modules, making the app effectively broken on phones | Frontend: CSS `@media (max-width: 768px)`, HTML navigation | Add a mobile hamburger button that toggles the `mobile-open` class |
| BR7 | **Low** | **`normalizeQuestionnaireFields` has a data-loss risk for unknown keys**: Fields from the spreadsheet that are not in `fieldMap` are passed through unchanged. However, the back-compat mirror block only adds mirrors for the handful of known fields. New columns added to the spreadsheet will arrive in their original snake_case and may be invisible to frontend code that accesses PascalCase keys | Backend: `normalizeQuestionnaireFields`, ~1985–2000 | Document the explicit list of supported fields and add a warning log for unmapped columns |

---

## 5. UI Issues

| # | Severity | Description | Location | Fix |
|---|----------|-------------|----------|-----|
| U1 | **High** | **Invalid CSS `//` single-line comments**: The `<style>` block uses `//` syntax (e.g., `// ADVANCED ASSESSMENT MODULE - INDEPENDENT STYLES`) which is not valid CSS. Browsers may silently ignore the following CSS rule or misparse the entire block depending on parser strictness | Frontend: CSS section, lines ~810, ~813 | Replace all `//` CSS comments with `/* ... */` |
| U2 | **High** | **Version/branding inconsistency**: The `<title>` is `SCP-OS v2.0 Multi-Module`, the backend header comment says `CP-OS v1.1 DSS`, and internal code comments say `CP-OS_v2 Lite`. Users and support staff will see different version strings depending on where they look | Frontend: `<title>` line ~6; Backend: header comment line ~2 | Standardise on a single product name and version string used everywhere |
| U3 | **Medium** | **No loading spinner/skeleton on initial data fetch**: While `getCPOSLiteTables` and `getCPOSQuestionnaireConfig` are awaited, the UI shows whatever partial state is left from the previous render. On slow connections or large sheets this gives the impression the app is frozen | Frontend: `init()` function, ~9150 | Show a visible loading indicator before the async calls and hide it in the `then`/`catch` handlers |
| U4 | **Medium** | **Button text uses `innerHTML` for i18n translations**: `updateButtonLabels` sets `el.innerHTML = text` whenever the translation string contains `<strong>`. Translation strings are developer-controlled and currently safe, but this pattern means any future translation typo (e.g. an unclosed tag) can break button rendering | Frontend: `updateButtonLabels`, ~9124 | Use `textContent` for plain strings; if bold formatting is needed in a button, use a dedicated child `<strong>` element that is styled separately |
| U5 | **Medium** | **Sidebar `user-avatar` always shows initials "A" / generic gradient**: The user avatar element is a fixed coloured circle with a hardcoded initial rather than the real user's name or photo, giving no personalisation signal | Frontend: sidebar HTML, `.user-avatar` CSS | Populate the avatar from `Session.getActiveUser()` on the backend `doGet` side or accept it as a URL parameter |
| U6 | **Low** | **Tab overflow on small screens not handled gracefully**: The `.tabs` row has `overflow: auto` (horizontal scroll), but there is no visual affordance (e.g. fade edge, scroll indicator) that the tabs are scrollable. On narrow screens with 3+ tabs the active tab may be off-screen with no hint | Frontend: `.tabs` CSS, ~780–800 | Add a scroll-shadow or arrow indicator when tab content overflows |
| U7 | **Low** | **Disabled buttons retain hover cursor**: `.btn:disabled` sets `cursor: not-allowed` but the `opacity: 0.55` on hover is not reset; `.btn:hover` increases opacity for non-disabled buttons but there is no `:disabled:hover` rule to suppress the highlight, so disabled buttons still get the hover treatment in some browsers | Frontend: `.btn:disabled`, `.btn:hover` CSS, ~1065–1075 | Add `.btn:disabled:hover { background: ... }` rule to suppress hover effects |
| U8 | **Low** | **`<meta name="viewport">` does not include `user-scalable=no`**: Without it, double-tap-to-zoom on mobile can break the fixed sidebar layout because zooming shifts the coordinate system | Frontend: `<meta name="viewport">` line ~8 | Consider adding `minimum-scale=1, maximum-scale=1` (or alternatively ensure the layout is robust under zoom) |
| U9 | **Low** | **`Coming Soon` modules are non-functional but not clearly disabled**: Sidebar items for future modules show a "Coming Soon" badge but clicking them still triggers the module navigation. Users who click these get an empty panel with no feedback | Frontend: sidebar nav HTML and module containers | Visually disable the nav items (pointer-events: none, reduced opacity) and show a tooltip/modal explaining the feature is upcoming |
| U10 | **Low** | **Long `final_text` strings in pathway cards can overflow card boundaries**: The `mutedBox` element that displays the evaluation narrative uses `white-space: pre-line` but has no `overflow-wrap: break-word` or max-height, so very long single-word tokens (e.g. variable names) or run-on sentences can break the card layout | Frontend: `.mutedBox` CSS, ~1285–1295 | Add `overflow-wrap: break-word; word-break: break-word;` to `.mutedBox` |

---

## Summary

| Category | Count | Highest Severity |
|----------|-------|-----------------|
| Strengths | 14 | — |
| Weaknesses | 10 | — |
| Bugs | 10 | High (B1, B2, B3) |
| Breakages | 7 | Critical (BR1) |
| UI Issues | 10 | High (U1, U2) |

### Top Priorities
1. **BR1 / B1–B3**: Remove the hardcoded spreadsheet ID; fix the `SIGNAL_VALUE_NORMALIZATIONS` duplicate keys (`'NONE'`, `'MEDIUM'`) and the `saveCPOSLiteAssessment` column-mapping errors — these directly corrupt stored data.
2. **B7**: Escape `violation.ruleId` before injecting into an `onclick` attribute to close the XSS risk.
3. **BR6 / U1**: Add a mobile hamburger button and fix invalid CSS `//` comments so the app is usable on phones.
4. **BR4**: Guard `Session.getActiveUser().getEmail()` with an `|| 'anonymous'` fallback.
