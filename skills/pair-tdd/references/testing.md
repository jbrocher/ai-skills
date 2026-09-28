# Behavioral Python tests

Use pytest fixtures to make test context explicit and reusable without hiding the behavior being specified.

```python
import pytest

from shop.checkout import Cart, checkout


@pytest.fixture
def cart_with_one_book() -> Cart:
    cart = Cart()
    cart.add(sku="book-1", unit_price_cents=1_500, quantity=1)
    return cart


def test_customer_can_checkout_a_valid_cart(cart_with_one_book: Cart) -> None:
    receipt = checkout(cart_with_one_book, payment_token="valid-token")

    assert receipt.status == "confirmed"
    assert receipt.total_cents == 1_500
```

This test observes the checkout interface and uses known literals from the example. It does not assert which internal service was called.

Avoid implementation-coupled tests such as patching a private helper and asserting its call count. Also avoid tautological expectations:

```python
def test_total_for_two_lines() -> None:
    lines = [(1_000, 2), (500, 1)]
    expected = sum(price * quantity for price, quantity in lines)  # Bad: duplicates the algorithm.

    assert calculate_total(lines) == expected
```

Use an independently known result instead:

```python
def test_total_for_two_lines() -> None:
    assert calculate_total([(1_000, 2), (500, 1)]) == 2_500
```

One logical behavior may need several assertions to describe one returned result. Do not force a single assertion when doing so would obscure the specification.
