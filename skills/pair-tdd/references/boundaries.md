# Ports, adapters, and dependency overrides

## Prefer ports and adapters

Define a narrow application-owned port for an external capability, inject it into application behavior, and keep protocol details in an adapter.

```python
from typing import Protocol


class PaymentPort(Protocol):
    def charge(self, *, token: str, amount_cents: int) -> str: ...
```

An application-level test can use a small fixture implementing this port. This is a boundary fake, not a mock of an internal collaborator.

```python
import pytest


class AcceptingPayments:
    def charge(self, *, token: str, amount_cents: int) -> str:
        return "payment-123"


@pytest.fixture
def payment_port() -> AcceptingPayments:
    return AcceptingPayments()
```

## FastAPI dependency overrides

Exercise the HTTP seam with the application and override only the external boundary dependency. Always clear overrides so state does not leak between tests.

```python
import pytest
from fastapi.testclient import TestClient

from shop.api import app, get_payment_port


class AcceptingPayments:
    def charge(self, *, token: str, amount_cents: int) -> str:
        return "payment-123"


@pytest.fixture
def client() -> TestClient:
    app.dependency_overrides[get_payment_port] = lambda: AcceptingPayments()
    with TestClient(app) as test_client:
        yield test_client
    app.dependency_overrides.clear()


def test_customer_can_checkout_through_the_api(client: TestClient) -> None:
    response = client.post(
        "/checkouts",
        json={"sku": "book-1", "payment_token": "valid-token"},
    )

    assert response.status_code == 201
    assert response.json() == {
        "status": "confirmed",
        "payment_id": "payment-123",
    }
```

Use a dedicated app factory instead of a global app when that is already the project's convention.

## Test real HTTP adapters at the protocol boundary

Do not replace a concrete HTTP adapter with a fake when the adapter itself is under test: its job is to form a real request and interpret a response. Intercept HTTP at the transport boundary and assert the adapter's public result.

For adapters built with `httpx`, suggest installing `pytest-httpx` and use its `httpx_mock` fixture:

```bash
python -m pip install pytest-httpx
```

Adapt the install command to the project's package manager and dependency groups; do not install anything without the developer's authorization.

```python
from pytest_httpx import HTTPXMock

from shop.payments import HttpPaymentAdapter


def test_payment_adapter_returns_the_provider_payment_id(
    httpx_mock: HTTPXMock,
) -> None:
    httpx_mock.add_response(
        method="POST",
        url="https://payments.example.test/charges",
        json={"id": "payment-123", "state": "accepted"},
        status_code=201,
    )
    adapter = HttpPaymentAdapter(
        base_url="https://payments.example.test",
        api_key="test-key",
    )

    payment_id = adapter.charge(token="valid-token", amount_cents=1_500)

    assert payment_id == "payment-123"
```

If the adapter uses `requests`, reach for `requests-mock` instead. Prefer the library matching the actual client over hand-patching `httpx`, `requests`, sockets, or internal adapter methods.
