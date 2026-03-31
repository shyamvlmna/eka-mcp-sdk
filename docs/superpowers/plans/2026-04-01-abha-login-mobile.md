# ABHA Login via Mobile Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add ABHA mobile login as an isolated MCP tool that handles the full OTP-based login flow (init → verify → select profile → fetch card) in a single tool invocation using FastMCP's native `ctx.elicit()`.

**Architecture:** New 3-layer module (`AbhaClient` → `AbhaService` → `abha_tools`) extending `BaseEkaClient` for HTTP/auth reuse. Registered in `server.py` alongside existing doctor tools. No changes to existing tools or clients.

**Tech Stack:** Python, FastMCP 2.14.5, httpx (via BaseEkaClient), FastMCP elicitation API

**Spec:** `docs/superpowers/specs/2026-04-01-abha-login-mobile-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `eka_mcp_sdk/clients/abha_client.py` | Create | HTTP calls to 4 ABHA API endpoints |
| `eka_mcp_sdk/services/abha_service.py` | Create | Orchestrates multi-step login flow with elicitation |
| `eka_mcp_sdk/tools/abha_tools.py` | Create | MCP tool registration for `abha_login_via_mobile` |
| `eka_mcp_sdk/server.py` | Modify (line 56) | Import and call `register_abha_tools(mcp)` |
| `tests/test_abha_client.py` | Create | Unit tests for AbhaClient |
| `tests/test_abha_service.py` | Create | Unit tests for AbhaService flow logic |

---

### Task 1: AbhaClient — HTTP layer

**Files:**
- Create: `eka_mcp_sdk/clients/abha_client.py`
- Create: `tests/test_abha_client.py`

- [ ] **Step 1: Write failing tests for AbhaClient**

Create `tests/test_abha_client.py`:

```python
"""Unit tests for AbhaClient ABHA API methods."""

import asyncio
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from eka_mcp_sdk.clients.abha_client import AbhaClient


@pytest.fixture
def client():
    """Create AbhaClient with a dummy token."""
    with patch("eka_mcp_sdk.clients.base_client.settings") as mock_settings:
        mock_settings.client_id = "test-client-id"
        mock_settings.client_secret = None
        mock_settings.api_base_url = "https://api.eka.care"
        c = AbhaClient(access_token="test-token")
    return c


@pytest.fixture
def mock_make_request(client):
    """Patch _make_request on the client instance."""
    client._make_request = AsyncMock()
    return client._make_request


class TestLoginInit:
    def test_calls_correct_endpoint(self, client, mock_make_request):
        mock_make_request.return_value = {"txn_id": "txn-123", "hint": "OTP sent"}
        result = asyncio.run(client.login_init("mobile", "9876543210"))

        mock_make_request.assert_called_once_with(
            method="POST",
            endpoint="/abdm/na/v1/profile/login/init",
            data={"method": "mobile", "identifier": "9876543210"},
        )
        assert result["txn_id"] == "txn-123"


class TestLoginVerify:
    def test_calls_correct_endpoint(self, client, mock_make_request):
        mock_make_request.return_value = {
            "txn_id": "txn-456",
            "skip_state": "abha_end",
            "profile": {"abha_number": "1234"},
            "eka": {"oid": "oid-1", "uuid": "uuid-1", "min_token": "tok"},
        }
        result = asyncio.run(client.login_verify("123456", "txn-123"))

        mock_make_request.assert_called_once_with(
            method="POST",
            endpoint="/abdm/na/v1/profile/login/verify",
            data={"otp": "123456", "txn_id": "txn-123"},
        )
        assert result["skip_state"] == "abha_end"
        assert result["eka"]["oid"] == "oid-1"


class TestLoginPhr:
    def test_calls_correct_endpoint(self, client, mock_make_request):
        mock_make_request.return_value = {
            "txn_id": "txn-789",
            "skip_state": "abha_end",
            "profile": {"abha_address": "user@abdm"},
            "eka": {"oid": "oid-2", "uuid": "uuid-2", "min_token": "tok2"},
        }
        result = asyncio.run(client.login_phr("user@abdm", "txn-456"))

        mock_make_request.assert_called_once_with(
            method="POST",
            endpoint="/abdm/na/v1/profile/login/phr",
            data={"phr_address": "user@abdm", "txn_id": "txn-456"},
        )
        assert result["eka"]["oid"] == "oid-2"


class TestGetAbhaCard:
    def test_calls_correct_endpoint_and_returns_bytes(self, client):
        """get_abha_card should make a GET request with X-Pt-Id header and return raw bytes."""
        fake_png = b"\x89PNG\r\n\x1a\nfakeimage"

        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.content = fake_png

        client._http_client = AsyncMock()
        client._http_client.request = AsyncMock(return_value=mock_response)

        # Patch auth and settings used inside get_abha_card
        with patch("eka_mcp_sdk.clients.base_client.settings") as mock_settings:
            mock_settings.client_id = "test-client-id"
            mock_settings.api_base_url = "https://api.eka.care"
            client._auth_manager = AsyncMock()
            client._auth_manager.get_auth_context = AsyncMock(
                return_value=MagicMock(auth_headers={"Authorization": "Bearer test-token"})
            )

            result = asyncio.run(client.get_abha_card("oid-1"))

        assert result == fake_png

        call_kwargs = client._http_client.request.call_args
        assert call_kwargs.kwargs["method"] == "GET"
        assert "oid=oid-1" in call_kwargs.kwargs["url"]
        assert call_kwargs.kwargs["headers"]["X-Pt-Id"] == "oid-1"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/shyam/code/eka-mcp-sdk && python -m pytest tests/test_abha_client.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'eka_mcp_sdk.clients.abha_client'`

- [ ] **Step 3: Implement AbhaClient**

Create `eka_mcp_sdk/clients/abha_client.py`:

```python
"""ABHA API client for ABDM login and profile operations."""

from typing import Dict, Any
import logging

from .base_client import BaseEkaClient
from ..auth.models import EkaAPIError
from ..config.settings import settings

logger = logging.getLogger(__name__)


class AbhaClient(BaseEkaClient):
    """HTTP client for ABHA/ABDM APIs. Extends BaseEkaClient for shared auth and HTTP handling."""

    def get_api_module_name(self) -> str:
        return "ABHA"

    async def login_init(self, method: str, identifier: str) -> Dict[str, Any]:
        """Initiate ABHA login by sending OTP.

        Args:
            method: Login method — "mobile", "aadhaar_number", "abha_number", or "phr_address"
            identifier: Value matching the method (e.g. 10-digit mobile number)

        Returns:
            {"txn_id": str, "hint": str}
        """
        return await self._make_request(
            method="POST",
            endpoint="/abdm/na/v1/profile/login/init",
            data={"method": method, "identifier": identifier},
        )

    async def login_verify(self, otp: str, txn_id: str) -> Dict[str, Any]:
        """Verify the login OTP.

        Args:
            otp: OTP received on the user's mobile
            txn_id: Transaction ID from login_init response

        Returns:
            {"txn_id": str, "skip_state": str, "profile": dict, "abha_profiles": list, "eka": dict, "hint": str}
        """
        return await self._make_request(
            method="POST",
            endpoint="/abdm/na/v1/profile/login/verify",
            data={"otp": otp, "txn_id": txn_id},
        )

    async def login_phr(self, phr_address: str, txn_id: str) -> Dict[str, Any]:
        """Complete login by selecting an ABHA address (PHR address).

        Args:
            phr_address: The ABHA address selected by the user (e.g. "user@abdm")
            txn_id: Transaction ID from login_verify response

        Returns:
            {"txn_id": str, "skip_state": str, "profile": dict, "eka": dict, "hint": str}
        """
        return await self._make_request(
            method="POST",
            endpoint="/abdm/na/v1/profile/login/phr",
            data={"phr_address": phr_address, "txn_id": txn_id},
        )

    async def get_abha_card(self, oid: str) -> bytes:
        """Download the ABHA card as a PNG image.

        Makes a direct HTTP request (bypassing _make_request) because the
        response is binary PNG, not JSON.

        Args:
            oid: Eka user OID from the login response's eka.oid field

        Returns:
            Raw PNG image bytes
        """
        url = f"{settings.api_base_url}/abdm/v1/profile/asset/card"
        headers = {"X-Pt-Id": oid, "client-id": settings.client_id}

        if self.access_token or settings.client_secret:
            auth_context = await self._auth_manager.get_auth_context()
            headers.update(auth_context.auth_headers)

        response = await self._http_client.request(
            method="GET",
            url=url,
            headers=headers,
            params={"oid": oid},
        )

        if response.status_code >= 400:
            error_detail = await self._parse_error_response(response)
            raise EkaAPIError(
                message=error_detail["message"],
                status_code=response.status_code,
                error_code=error_detail.get("error_code"),
            )

        return response.content
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/shyam/code/eka-mcp-sdk && python -m pytest tests/test_abha_client.py -v`
Expected: All 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add eka_mcp_sdk/clients/abha_client.py tests/test_abha_client.py
git commit -m "feat(abha): add AbhaClient HTTP layer for ABHA login APIs"
```

---

### Task 2: AbhaService — orchestration layer

**Files:**
- Create: `eka_mcp_sdk/services/abha_service.py`
- Create: `tests/test_abha_service.py`

- [ ] **Step 1: Write failing tests for AbhaService**

Create `tests/test_abha_service.py`:

```python
"""Unit tests for AbhaService login flow orchestration."""

import asyncio
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from eka_mcp_sdk.services.abha_service import AbhaService
from eka_mcp_sdk.clients.abha_client import AbhaClient


def make_mock_client():
    client = MagicMock(spec=AbhaClient)
    client.login_init = AsyncMock()
    client.login_verify = AsyncMock()
    client.login_phr = AsyncMock()
    client.get_abha_card = AsyncMock()
    return client


def make_mock_ctx():
    ctx = AsyncMock()
    ctx.info = AsyncMock()
    ctx.error = AsyncMock()
    return ctx


def accepted(data):
    """Create a mock AcceptedElicitation."""
    result = MagicMock()
    result.action = "accept"
    result.data = data
    return result


def cancelled():
    """Create a mock CancelledElicitation."""
    result = MagicMock()
    result.action = "cancel"
    return result


VERIFY_RESPONSE_ABHA_END = {
    "txn_id": "txn-456",
    "skip_state": "abha_end",
    "profile": {"abha_number": "1234", "abha_address": "user@abdm", "first_name": "Test"},
    "abha_profiles": [],
    "eka": {"oid": "oid-1", "uuid": "uuid-1", "min_token": "tok"},
    "hint": "",
}

VERIFY_RESPONSE_ABHA_SELECT = {
    "txn_id": "txn-456",
    "skip_state": "abha_select",
    "profile": {},
    "abha_profiles": [
        {"abha_address": "alice@abdm", "name": "Alice"},
        {"abha_address": "bob@abdm", "name": "Bob"},
    ],
    "eka": {},
    "hint": "",
}

LOGIN_PHR_RESPONSE = {
    "txn_id": "txn-789",
    "skip_state": "abha_end",
    "profile": {"abha_address": "alice@abdm", "first_name": "Alice"},
    "abha_profiles": [],
    "eka": {"oid": "oid-2", "uuid": "uuid-2", "min_token": "tok2"},
    "hint": "",
}

FAKE_CARD = b"\x89PNG\r\n\x1a\nfakecard"


class TestLoginViaMobileAbhaEnd:
    """Test the happy path where skip_state is abha_end (single profile)."""

    def test_full_flow(self):
        client = make_mock_client()
        client.login_init.return_value = {"txn_id": "txn-123", "hint": "OTP sent"}
        client.login_verify.return_value = VERIFY_RESPONSE_ABHA_END
        client.get_abha_card.return_value = FAKE_CARD

        ctx = make_mock_ctx()
        # First elicit: OTP input
        ctx.elicit = AsyncMock(return_value=accepted(MagicMock(value="123456")))

        service = AbhaService(client)
        result = asyncio.run(service.login_via_mobile("9876543210", ctx))

        client.login_init.assert_called_once_with("mobile", "9876543210")
        client.login_verify.assert_called_once_with("123456", "txn-123")
        client.login_phr.assert_not_called()
        client.get_abha_card.assert_called_once_with("oid-1")

        assert result["success"] is True
        assert result["profile"]["abha_number"] == "1234"
        assert result["abha_card"] == FAKE_CARD


class TestLoginViaMobileAbhaSelect:
    """Test flow where skip_state is abha_select (multiple profiles)."""

    def test_full_flow_with_profile_selection(self):
        client = make_mock_client()
        client.login_init.return_value = {"txn_id": "txn-123", "hint": "OTP sent"}
        client.login_verify.return_value = VERIFY_RESPONSE_ABHA_SELECT
        client.login_phr.return_value = LOGIN_PHR_RESPONSE
        client.get_abha_card.return_value = FAKE_CARD

        ctx = make_mock_ctx()
        # First elicit: OTP, Second elicit: profile selection
        ctx.elicit = AsyncMock(
            side_effect=[
                accepted(MagicMock(value="123456")),
                accepted(MagicMock(data="alice@abdm")),
            ]
        )

        service = AbhaService(client)
        result = asyncio.run(service.login_via_mobile("9876543210", ctx))

        client.login_phr.assert_called_once_with("alice@abdm", "txn-456")
        client.get_abha_card.assert_called_once_with("oid-2")
        assert result["success"] is True
        assert result["profile"]["abha_address"] == "alice@abdm"


class TestLoginViaMobileAbhaCreate:
    """Test flow where skip_state is abha_create (unsupported)."""

    def test_returns_unsupported(self):
        client = make_mock_client()
        client.login_init.return_value = {"txn_id": "txn-123", "hint": "OTP sent"}
        client.login_verify.return_value = {
            **VERIFY_RESPONSE_ABHA_END,
            "skip_state": "abha_create",
        }

        ctx = make_mock_ctx()
        ctx.elicit = AsyncMock(return_value=accepted(MagicMock(value="123456")))

        service = AbhaService(client)
        result = asyncio.run(service.login_via_mobile("9876543210", ctx))

        assert result["success"] is False
        assert "not supported" in result["error"].lower()
        client.get_abha_card.assert_not_called()


class TestLoginViaMobileElicitationCancelled:
    """Test that cancelling OTP elicitation returns early."""

    def test_otp_cancelled(self):
        client = make_mock_client()
        client.login_init.return_value = {"txn_id": "txn-123", "hint": "OTP sent"}

        ctx = make_mock_ctx()
        ctx.elicit = AsyncMock(return_value=cancelled())

        service = AbhaService(client)
        result = asyncio.run(service.login_via_mobile("9876543210", ctx))

        assert result["success"] is False
        assert "cancelled" in result["error"].lower()
        client.login_verify.assert_not_called()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/shyam/code/eka-mcp-sdk && python -m pytest tests/test_abha_service.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'eka_mcp_sdk.services.abha_service'`

- [ ] **Step 3: Implement AbhaService**

Create `eka_mcp_sdk/services/abha_service.py`:

```python
"""ABHA login orchestration service.

Manages the multi-step ABHA mobile login flow within a single tool invocation,
using FastMCP's ctx.elicit() for mid-flow user input (OTP, profile selection).
"""

from typing import Any, Dict
import logging

from ..clients.abha_client import AbhaClient
from ..auth.models import EkaAPIError

logger = logging.getLogger(__name__)


class AbhaService:
    """Orchestrates the ABHA login flow."""

    def __init__(self, client: AbhaClient):
        self.client = client

    async def login_via_mobile(self, mobile_number: str, ctx) -> Dict[str, Any]:
        """Execute the full ABHA mobile login flow.

        Steps:
        1. Send OTP via login_init
        2. Elicit OTP from user
        3. Verify OTP via login_verify
        4. Handle skip_state (abha_end / abha_select / abha_create)
        5. Fetch ABHA card
        6. Return profile + card

        Args:
            mobile_number: 10-digit mobile number
            ctx: FastMCP Context for elicitation and logging

        Returns:
            {"success": True, "profile": dict, "abha_card": bytes} on success
            {"success": False, "error": str} on failure
        """
        # Step 1: Send OTP
        await ctx.info(f"[abha_login] Initiating OTP for mobile: {mobile_number}")
        init_response = await self.client.login_init("mobile", mobile_number)
        txn_id = init_response["txn_id"]
        await ctx.info(f"[abha_login] OTP sent. txn_id: {txn_id}")

        # Step 2: Elicit OTP from user
        otp_result = await ctx.elicit(
            "Enter the OTP sent to your mobile number",
            str,
        )
        if otp_result.action != "accept":
            return {"success": False, "error": "OTP input cancelled by user"}

        otp = otp_result.data.value

        # Step 3: Verify OTP
        await ctx.info("[abha_login] Verifying OTP...")
        verify_response = await self.client.login_verify(otp, txn_id)
        skip_state = verify_response.get("skip_state", "")
        txn_id = verify_response.get("txn_id", txn_id)

        # Step 4: Handle skip_state
        if skip_state == "abha_end":
            return await self._complete_login(verify_response, ctx)

        if skip_state == "abha_select":
            return await self._handle_profile_selection(verify_response, txn_id, ctx)

        if skip_state == "abha_create":
            return {
                "success": False,
                "error": "ABHA creation is not supported yet. Please create an ABHA account first.",
            }

        return {
            "success": False,
            "error": f"Unexpected skip_state: {skip_state}",
        }

    async def _handle_profile_selection(
        self, verify_response: Dict[str, Any], txn_id: str, ctx
    ) -> Dict[str, Any]:
        """Handle abha_select state: elicit profile choice, then complete login."""
        abha_profiles = verify_response.get("abha_profiles", [])
        if not abha_profiles:
            return {"success": False, "error": "No ABHA profiles found for selection"}

        # Build selection options: list of ABHA addresses
        address_list = [p.get("abha_address", "") for p in abha_profiles if p.get("abha_address")]
        if not address_list:
            return {"success": False, "error": "No valid ABHA addresses found"}

        await ctx.info(f"[abha_login] {len(address_list)} ABHA profiles found, requesting selection")
        selection_result = await ctx.elicit(
            "Select your ABHA profile",
            address_list,
        )
        if selection_result.action != "accept":
            return {"success": False, "error": "Profile selection cancelled by user"}

        selected_address = selection_result.data
        await ctx.info(f"[abha_login] Selected: {selected_address}, completing login...")

        phr_response = await self.client.login_phr(selected_address, txn_id)
        return await self._complete_login(phr_response, ctx)

    async def _complete_login(
        self, response: Dict[str, Any], ctx
    ) -> Dict[str, Any]:
        """Extract profile and fetch ABHA card."""
        eka = response.get("eka", {})
        oid = eka.get("oid")
        profile = response.get("profile", {})

        if not oid:
            return {
                "success": True,
                "profile": profile,
                "abha_card": None,
                "warning": "No OID returned, could not fetch ABHA card",
            }

        await ctx.info(f"[abha_login] Login complete. Fetching ABHA card for oid: {oid}")
        try:
            card_bytes = await self.client.get_abha_card(oid)
        except EkaAPIError as e:
            logger.warning(f"Failed to fetch ABHA card: {e.message}")
            card_bytes = None

        return {
            "success": True,
            "profile": profile,
            "abha_card": card_bytes,
            "oid": oid,
        }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/shyam/code/eka-mcp-sdk && python -m pytest tests/test_abha_service.py -v`
Expected: All 4 test classes PASS

- [ ] **Step 5: Commit**

```bash
git add eka_mcp_sdk/services/abha_service.py tests/test_abha_service.py
git commit -m "feat(abha): add AbhaService orchestration for mobile login flow"
```

---

### Task 3: MCP tool registration and server wiring

**Files:**
- Create: `eka_mcp_sdk/tools/abha_tools.py`
- Modify: `eka_mcp_sdk/server.py` (line 56, add registration call)

- [ ] **Step 1: Create abha_tools.py**

Create `eka_mcp_sdk/tools/abha_tools.py`:

```python
"""ABHA MCP tool registration."""

from typing import Any, Dict, Annotated
import logging

from fastmcp import FastMCP
from fastmcp.server.dependencies import get_access_token, AccessToken
from fastmcp.dependencies import CurrentContext
from fastmcp.server.context import Context

from ..clients.abha_client import AbhaClient
from ..services.abha_service import AbhaService
from ..auth.models import EkaAPIError

logger = logging.getLogger(__name__)


def register_abha_tools(mcp: FastMCP) -> None:
    """Register ABHA/ABDM MCP tools."""

    @mcp.tool(
        tags={"abha", "login", "abdm"},
    )
    async def abha_login_via_mobile(
        mobile_number: Annotated[str, "10-digit mobile number registered with ABHA"],
        ctx: Context = CurrentContext(),
    ) -> Dict[str, Any]:
        """
        Login to ABHA (Ayushman Bharat Health Account) using a mobile number.

        Handles the complete login flow in a single invocation:
        - Sends OTP to the mobile number
        - Collects OTP from user via elicitation
        - Verifies OTP and authenticates
        - If multiple ABHA profiles exist, asks user to select one
        - Returns ABHA profile and downloads the ABHA card (PNG image)

        When to Use This Tool
        Use this tool when a user wants to log into their ABHA account or link
        their ABDM health ID. This is the first step for any ABDM workflow.

        Trigger Keywords / Phrases
        abha login, abdm login, health id login, abha mobile login,
        link abha, connect abdm, ayushman bharat login

        What to Return
        Returns the user's ABHA profile data and their ABHA card image.
        """
        await ctx.info(f"[abha_login_via_mobile] Starting ABHA login for mobile: {mobile_number}")

        try:
            token: AccessToken | None = get_access_token()
            access_token = token.token if token else None
            client = AbhaClient(access_token=access_token)
            service = AbhaService(client)
            result = await service.login_via_mobile(mobile_number, ctx)

            if result.get("success"):
                await ctx.info("[abha_login_via_mobile] Login completed successfully")
            else:
                await ctx.info(f"[abha_login_via_mobile] Login flow ended: {result.get('error', 'unknown')}")

            return result
        except EkaAPIError as e:
            await ctx.error(f"[abha_login_via_mobile] Failed: {e.message}")
            return {
                "success": False,
                "error": {
                    "message": e.message,
                    "status_code": e.status_code,
                    "error_code": e.error_code,
                },
            }
```

- [ ] **Step 2: Register ABHA tools in server.py**

In `eka_mcp_sdk/server.py`, add the import at line 12 (after the existing doctor_tools import):

```python
from eka_mcp_sdk.tools.abha_tools import register_abha_tools
```

And add the registration call after line 56 (`register_doctor_tools(mcp)`):

```python
    register_abha_tools(mcp)
```

- [ ] **Step 3: Verify the server starts without errors**

Run: `cd /Users/shyam/code/eka-mcp-sdk && python -c "from eka_mcp_sdk.server import create_mcp_server; mcp = create_mcp_server(); print('Server created successfully')"`
Expected: `Server created successfully` (no import or registration errors)

- [ ] **Step 4: Commit**

```bash
git add eka_mcp_sdk/tools/abha_tools.py eka_mcp_sdk/server.py
git commit -m "feat(abha): register abha_login_via_mobile MCP tool"
```

---

### Task 4: Run full test suite and verify no regressions

- [ ] **Step 1: Run all tests**

Run: `cd /Users/shyam/code/eka-mcp-sdk && python -m pytest tests/test_abha_client.py tests/test_abha_service.py -v`
Expected: All tests PASS

- [ ] **Step 2: Verify existing imports are not broken**

Run: `cd /Users/shyam/code/eka-mcp-sdk && python -c "from eka_mcp_sdk.tools.doctor_clinic_tools import register_doctor_clinic_tools; print('OK')"`
Expected: `OK`

- [ ] **Step 3: Verify ABHA tool is importable end-to-end**

Run: `cd /Users/shyam/code/eka-mcp-sdk && python -c "from eka_mcp_sdk.tools.abha_tools import register_abha_tools; print('OK')"`
Expected: `OK`
