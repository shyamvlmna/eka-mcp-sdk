# ABHA Login via Mobile — Design Spec

## Overview

Add ABHA login via mobile number as a new module in `eka-mcp-sdk`. ABHA is an internal system exposing APIs for ABDM (Ayushman Bharat Digital Mission) use cases. This implementation covers Milestone 1 — specifically the mobile login flow.

The module is fully isolated from existing EMR tools. It reuses only the shared HTTP/auth infrastructure (`BaseEkaClient`).

## Scope

**In scope:**
- ABHA login via mobile number (OTP-based, single orchestrator tool)
- Handle `abha_select` (user picks from multiple ABHA profiles) and `abha_end` (login complete) skip states
- Fetch and return ABHA card (PNG) post-login

**Out of scope (for now):**
- `abha_create` skip state (new ABHA registration)
- Login via Aadhaar, ABHA number, or ABHA address
- ABHA enrollment/registration flows
- Care context linking, consent management (M2/M3/M4)

## Architecture

### New Files

| File | Purpose |
|------|---------|
| `eka_mcp_sdk/clients/abha_client.py` | HTTP client for ABHA APIs, extends `BaseEkaClient` |
| `eka_mcp_sdk/services/abha_service.py` | Orchestrates the multi-step login flow |
| `eka_mcp_sdk/tools/abha_tools.py` | MCP tool registration |

### Modified Files (non-breaking)

| File | Change |
|------|--------|
| `eka_mcp_sdk/utils/tool_registration.py` | Add `register_abha_tools` import and call in `register_all_tools()` |

### No Changes To

`ClientFactory`, `BaseEMRClient`, `EkaEMRClient`, settings, or any existing service/tool/client files.

## Components

### 1. `AbhaClient(BaseEkaClient)`

Thin HTTP layer. Reuses `BaseEkaClient._make_request()` for auth headers and error handling.

```python
class AbhaClient(BaseEkaClient):
    def get_api_module_name(self) -> str:
        return "ABHA"

    async def login_init(self, method: str, identifier: str) -> Dict[str, Any]:
        """POST /abdm/na/v1/profile/login/init"""
        # Returns: {"txn_id": "...", "hint": "..."}

    async def login_verify(self, otp: str, txn_id: str) -> Dict[str, Any]:
        """POST /abdm/na/v1/profile/login/verify"""
        # Returns: {"txn_id", "skip_state", "profile", "abha_profiles", "eka": {"oid", "uuid", "min_token"}, "hint"}

    async def login_phr(self, phr_address: str, txn_id: str) -> Dict[str, Any]:
        """POST /abdm/na/v1/profile/login/phr"""
        # Returns: {"txn_id", "skip_state", "profile", "abha_profiles", "eka": {"oid", ...}, "hint"}

    async def get_abha_card(self, oid: str) -> bytes:
        """GET /abdm/v1/profile/asset/card?oid=<oid> with X-Pt-Id header"""
        # Returns: raw PNG bytes
```

### 2. `AbhaService`

Single orchestrator method that manages the full login flow. Holds `txn_id` state internally across the multi-step flow.

```python
class AbhaService:
    def __init__(self, client: AbhaClient):
        self.client = client

    async def login_via_mobile(self, mobile_number: str, ctx: Context) -> Dict[str, Any]:
        """
        Full ABHA login flow:
        1. Call login_init(method="mobile", identifier=mobile_number)
        2. Elicit OTP from user via ctx (FastMCP elicitation)
        3. Call login_verify(otp, txn_id)
        4. Check skip_state:
           - "abha_end" → extract oid, fetch card, return
           - "abha_select" → elicit profile selection from user, call login_phr, then fetch card
        5. Fetch ABHA card via get_abha_card(oid) using oid from eka response
        6. Return profile data + raw card image bytes
        """
```

**Elicitation via `ctx.elicit()` (FastMCP native):**

FastMCP's `Context.elicit()` supports mid-tool pause-and-wait for user input. This is used (not the older `is_elicitation` flag pattern) to keep the full flow in a single tool invocation:

- After `login_init`: `await ctx.elicit("Enter OTP sent to your mobile", str)` → returns OTP string
- After `login_verify` with `abha_select`: `await ctx.elicit("Select ABHA profile", list_of_address_strings)` → returns selected address

If the user cancels/declines either elicitation, the tool returns early with an appropriate message.

### 3. `abha_tools.py`

Single MCP tool registration function.

```python
def register_abha_tools(mcp: FastMCP) -> None:
    @mcp.tool(
        tags={"abha", "login", "abdm"},
    )
    async def abha_login_via_mobile(
        mobile_number: Annotated[str, "10-digit mobile number"],
        ctx: Context = CurrentContext()
    ) -> Dict[str, Any]:
        """
        Login to ABHA (Ayushman Bharat Health Account) using mobile number.

        Handles the complete login flow:
        - Sends OTP to the mobile number
        - Collects OTP from user
        - Verifies and authenticates
        - Returns ABHA profile and card

        Trigger Keywords:
        abha login, abdm login, health id login, abha mobile login
        """
        token = get_access_token()
        access_token = token.token if token else None
        client = AbhaClient(access_token=access_token)
        service = AbhaService(client)
        return await service.login_via_mobile(mobile_number, ctx)
```

## API Details

### Login Init
- **Method:** `POST`
- **Path:** `/abdm/na/v1/profile/login/init`
- **Body:** `{"method": "mobile", "identifier": "<10-digit-number>"}`
- **Response:** `{"txn_id": "string", "hint": "string"}`
- **Auth:** Bearer token (from existing `get_access_token()`)

### Login Verify
- **Method:** `POST`
- **Path:** `/abdm/na/v1/profile/login/verify`
- **Body:** `{"otp": "string", "txn_id": "string"}`
- **Response:**
  ```json
  {
    "txn_id": "string",
    "skip_state": "abha_select | abha_end | abha_create",
    "profile": { "abha_number", "abha_address", "first_name", "last_name", "gender", "mobile", ... },
    "abha_profiles": [{ "abha_address", "name", "kyc_status", ... }],
    "eka": { "oid": "string", "uuid": "string", "min_token": "string" },
    "hint": "string"
  }
  ```

### Login PHR (for `abha_select` state)
- **Method:** `POST`
- **Path:** `/abdm/na/v1/profile/login/phr`
- **Body:** `{"phr_address": "string", "txn_id": "string"}`
- **Response:** Same shape as Login Verify response

### ABHA Card
- **Method:** `GET`
- **Path:** `/abdm/v1/profile/asset/card?oid=<oid>`
- **Headers:** `X-Pt-Id: <oid>` (required)
- **Response:** Raw PNG image bytes (Content-Type: image/png)

## Data Flow

```
User provides mobile_number
        │
        ▼
   login_init(mobile, number)
        │
        ▼
   txn_id returned
        │
        ▼
   Elicit OTP from user ◄── user enters OTP
        │
        ▼
   login_verify(otp, txn_id)
        │
        ├─── skip_state == "abha_end"
        │         │
        │         ▼
        │    Extract oid from eka.oid
        │         │
        │         ▼
        │    get_abha_card(oid) with X-Pt-Id header
        │         │
        │         ▼
        │    Return {profile, card_image}
        │
        ├─── skip_state == "abha_select"
        │         │
        │         ▼
        │    Elicit profile selection ◄── user picks ABHA address
        │         │
        │         ▼
        │    login_phr(selected_address, txn_id)
        │         │
        │         ▼
        │    Extract oid from eka.oid
        │         │
        │         ▼
        │    get_abha_card(oid) with X-Pt-Id header
        │         │
        │         ▼
        │    Return {profile, card_image}
        │
        └─── skip_state == "abha_create"
                  │
                  ▼
             Return {error: "ABHA creation not supported yet"}
```

## Error Handling

- Uses existing `EkaAPIError` pattern for all API errors
- ABHA API errors follow `GenericError` schema: `{"code": int, "error": str, "source_error": {"code": str, "message": str}}`
- Invalid OTP → surfaced as API error from verify endpoint
- Expired txn_id → surfaced as API error
- Elicitation cancelled by user → return early with appropriate message
- `abha_create` skip state → return unsupported message (out of scope)

## Testing

- Unit tests for `AbhaClient` methods (mock HTTP responses)
- Unit tests for `AbhaService` flow logic (mock client + elicitation)
- Integration test for the full tool (end-to-end with mocked APIs)
