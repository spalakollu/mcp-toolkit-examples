# Trading Simulation Example

This example demonstrates how to design safe MCP tools for a simulated trading domain. **This is a simulation only—no real money or market integrations are involved.** The patterns shown here generalize to any domain where agents interact with financial or critical systems.

## Domain Overview

A trading simulation allows agents to:
- View account positions and balances (read)
- Place buy/sell orders (write)
- Liquidate accounts or close positions (destructive)

This domain is ideal for demonstrating scope design because different operations require different permission levels, and mistakes can have significant (simulated) consequences.

## Tool Design

### Read Tool: Get Positions

```python
def get_positions(account_id: str) -> dict:
    """
    Retrieve all open positions for a trading account.
    
    Scope: read
    Idempotent: Yes
    Side effects: None
    
    Args:
        account_id: Unique identifier for the trading account
    
    Returns:
        {
            "account_id": str,
            "positions": [
                {
                    "position_id": str,
                    "symbol": str,
                    "quantity": float,
                    "average_price": float,
                    "current_price": float,
                    "unrealized_pnl": float
                }
            ],
            "total_unrealized_pnl": float
        }
    
    Raises:
        NotFoundError: If account_id does not exist
        ValidationError: If account_id format is invalid
    """
    # Validate input
    if not is_valid_uuid(account_id):
        raise ValidationError(f"Invalid account_id format: {account_id}")
    
    # Fetch account
    account = db.accounts.find_one({"account_id": account_id})
    if not account:
        raise NotFoundError(f"Account {account_id} not found")
    
    # Fetch positions
    positions = db.positions.find({"account_id": account_id, "status": "open"})
    
    # Calculate current values (simulated prices)
    position_list = []
    total_pnl = 0.0
    
    for pos in positions:
        current_price = get_simulated_price(pos["symbol"])
        unrealized_pnl = (current_price - pos["average_price"]) * pos["quantity"]
        
        position_list.append({
            "position_id": pos["position_id"],
            "symbol": pos["symbol"],
            "quantity": pos["quantity"],
            "average_price": pos["average_price"],
            "current_price": current_price,
            "unrealized_pnl": unrealized_pnl
        })
        total_pnl += unrealized_pnl
    
    return {
        "account_id": account_id,
        "positions": position_list,
        "total_unrealized_pnl": total_pnl
    }
```

**Why this design is safe**:
- No state changes; purely informational
- Validates inputs before processing
- Returns structured data, not raw database objects
- Requires only read scope

### Write Tool: Place Order

```python
def place_order(
    account_id: str,
    symbol: str,
    side: str,
    quantity: float,
    order_type: str = "market",
    idempotency_key: str | None = None
) -> dict:
    """
    Place a buy or sell order for a security.
    
    Scope: write
    Idempotent: Yes (if idempotency_key provided)
    Side effects: Creates order, updates account balance, opens/closes positions
    
    Args:
        account_id: Unique identifier for the trading account
        symbol: Security symbol (e.g., "AAPL", "BTC-USD")
        side: "buy" or "sell"
        quantity: Number of shares/units to trade (must be positive)
        order_type: "market" or "limit" (default: "market")
        idempotency_key: Optional UUID to prevent duplicate orders
    
    Returns:
        {
            "order_id": str,
            "account_id": str,
            "symbol": str,
            "side": str,
            "quantity": float,
            "filled_quantity": float,
            "average_fill_price": float,
            "status": "filled" | "partially_filled" | "rejected",
            "created_at": str (ISO 8601)
        }
    
    Raises:
        NotFoundError: If account_id does not exist
        ValidationError: If inputs are invalid
        InsufficientFundsError: If account lacks funds for buy order
        InsufficientPositionError: If account lacks position for sell order
    """
    # Validate inputs
    if not is_valid_uuid(account_id):
        raise ValidationError(f"Invalid account_id format: {account_id}")
    
    if side not in ["buy", "sell"]:
        raise ValidationError(f"side must be 'buy' or 'sell', got '{side}'")
    
    if quantity <= 0:
        raise ValidationError(f"quantity must be positive, got {quantity}")
    
    if order_type not in ["market", "limit"]:
        raise ValidationError(f"order_type must be 'market' or 'limit', got '{order_type}'")
    
    # Check idempotency
    if idempotency_key:
        existing_order = db.orders.find_one({"idempotency_key": idempotency_key})
        if existing_order:
            return format_order_response(existing_order)
    
    # Fetch account
    account = db.accounts.find_one({"account_id": account_id})
    if not account:
        raise NotFoundError(f"Account {account_id} not found")
    
    # Get simulated market price
    market_price = get_simulated_price(symbol)
    total_cost = market_price * quantity
    
    # Validate sufficient funds/position
    if side == "buy":
        if account["balance"] < total_cost:
            raise InsufficientFundsError(
                f"Account balance {account['balance']} insufficient for order cost {total_cost}"
            )
    else:  # sell
        position = db.positions.find_one({
            "account_id": account_id,
            "symbol": symbol,
            "status": "open"
        })
        if not position or position["quantity"] < quantity:
            raise InsufficientPositionError(
                f"Account has insufficient {symbol} position for sell order"
            )
    
    # Create order
    order_id = generate_uuid()
    order = {
        "order_id": order_id,
        "account_id": account_id,
        "symbol": symbol,
        "side": side,
        "quantity": quantity,
        "order_type": order_type,
        "status": "filled",  # Market orders fill immediately in simulation
        "filled_quantity": quantity,
        "average_fill_price": market_price,
        "idempotency_key": idempotency_key,
        "created_at": datetime.utcnow()
    }
    db.orders.insert_one(order)
    
    # Update account balance and positions
    if side == "buy":
        db.accounts.update_one(
            {"account_id": account_id},
            {"$inc": {"balance": -total_cost}}
        )
        # Update or create position
        db.positions.update_one(
            {"account_id": account_id, "symbol": symbol, "status": "open"},
            {
                "$inc": {"quantity": quantity},
                "$setOnInsert": {
                    "position_id": generate_uuid(),
                    "average_price": market_price,
                    "created_at": datetime.utcnow()
                }
            },
            upsert=True
        )
    else:  # sell
        db.accounts.update_one(
            {"account_id": account_id},
            {"$inc": {"balance": total_cost}}
        )
        # Reduce position
        db.positions.update_one(
            {"account_id": account_id, "symbol": symbol, "status": "open"},
            {"$inc": {"quantity": -quantity}}
        )
        # Close position if quantity reaches zero
        position = db.positions.find_one({
            "account_id": account_id,
            "symbol": symbol,
            "status": "open"
        })
        if position and position["quantity"] <= 0:
            db.positions.update_one(
                {"position_id": position["position_id"]},
                {"$set": {"status": "closed", "closed_at": datetime.utcnow()}}
            )
    
    return format_order_response(order)
```

**Why this design is safe**:
- Idempotent via idempotency_key
- Validates all inputs (account exists, sufficient funds/position, valid enums)
- Requires write scope (cannot be called with read-only access)
- Returns structured response with clear status

### Destructive Tool: Liquidate Account

```python
def liquidate_account(account_id: str, confirm: bool = False) -> dict:
    """
    Liquidate all positions in an account and close it.
    
    Scope: destructive
    Idempotent: Yes
    Side effects: Sells all positions, transfers funds, closes account
    
    Args:
        account_id: Unique identifier for the trading account
        confirm: Must be explicitly set to True to execute liquidation
    
    Returns:
        {
            "account_id": str,
            "preview": bool,
            "positions_to_close": int,
            "estimated_transfer": float,
            "executed_at": str (ISO 8601) | None
        }
    
    Raises:
        NotFoundError: If account_id does not exist
        ValidationError: If confirm is not True when executing
    """
    # Validate account exists
    account = db.accounts.find_one({"account_id": account_id})
    if not account:
        raise NotFoundError(f"Account {account_id} not found")
    
    # Get current positions
    positions = db.positions.find({"account_id": account_id, "status": "open"})
    position_list = list(positions)
    
    # Calculate estimated transfer value
    estimated_transfer = account["balance"]
    for pos in position_list:
        current_price = get_simulated_price(pos["symbol"])
        estimated_transfer += current_price * pos["quantity"]
    
    # If not confirmed, return preview only
    if not confirm:
        return {
            "account_id": account_id,
            "preview": True,
            "positions_to_close": len(position_list),
            "estimated_transfer": estimated_transfer,
            "message": "This is a preview. Set confirm=True to execute liquidation."
        }
    
    # Execute liquidation
    closed_count = 0
    for pos in position_list:
        # Sell position at market price
        current_price = get_simulated_price(pos["symbol"])
        sale_value = current_price * pos["quantity"]
        
        # Update account balance
        db.accounts.update_one(
            {"account_id": account_id},
            {"$inc": {"balance": sale_value}}
        )
        
        # Close position
        db.positions.update_one(
            {"position_id": pos["position_id"]},
            {"$set": {"status": "closed", "closed_at": datetime.utcnow()}}
        )
        closed_count += 1
    
    # Transfer all funds (simulated: just mark account as closed)
    final_balance = account["balance"] + sum(
        get_simulated_price(pos["symbol"]) * pos["quantity"]
        for pos in position_list
    )
    
    # Mark account as closed
    executed_at = datetime.utcnow()
    db.accounts.update_one(
        {"account_id": account_id},
        {
            "$set": {
                "status": "closed",
                "closed_at": executed_at,
                "final_balance": final_balance
            }
        }
    )
    
    return {
        "account_id": account_id,
        "preview": False,
        "positions_to_close": closed_count,
        "estimated_transfer": final_balance,
        "executed_at": executed_at.isoformat()
    }
```

**Why this design is safe**:
- Requires explicit confirmation (confirm=True)
- Provides preview mode to show what would happen
- Idempotent (can be called multiple times safely)
- Requires destructive scope (highest permission level)
- Returns clear execution results

## How Scopes Protect Against Catastrophic Actions

### Scenario 1: Agent with Read-Only Access

```
Agent scopes: ["read"]

Agent attempts:
1. get_positions(account_id="123") → ✓ Success
2. place_order(account_id="123", ...) → ✗ Error: Requires write scope
3. liquidate_account(account_id="123") → ✗ Error: Requires destructive scope

Result: Agent can explore but cannot make changes.
```

### Scenario 2: Agent with Write Access

```
Agent scopes: ["read", "write"]

Agent attempts:
1. get_positions(account_id="123") → ✓ Success
2. place_order(account_id="123", ...) → ✓ Success
3. liquidate_account(account_id="123") → ✗ Error: Requires destructive scope

Result: Agent can trade but cannot liquidate accounts.
```

### Scenario 3: Agent with Destructive Access

```
Agent scopes: ["read", "write", "destructive"]

Agent attempts:
1. get_positions(account_id="123") → ✓ Success
2. place_order(account_id="123", ...) → ✓ Success
3. liquidate_account(account_id="123", confirm=False) → Preview returned
4. liquidate_account(account_id="123", confirm=True) → ✓ Success (with confirmation)

Result: Agent can perform all operations, but destructive operations require explicit confirmation.
```

## Why This Pattern Generalizes to Real Systems

The trading simulation demonstrates patterns that apply to any domain:

1. **Read operations are safe for exploration**: Agents can discover system state before making decisions
2. **Write operations require validation**: Input validation, existence checks, and idempotency prevent errors
3. **Destructive operations need confirmation**: Human-in-the-loop patterns prevent catastrophic mistakes
4. **Scopes enforce boundaries**: Different permission levels create natural safety boundaries

These same patterns apply to:
- Cloud infrastructure management (list instances, create VMs, terminate instances)
- Database administration (query tables, create records, drop tables)
- File systems (list files, create files, delete directories)
- API management (list endpoints, create APIs, delete APIs)

The key is designing tools that are safe, scoped, and idempotent, regardless of the specific domain.

## Summary

This trading simulation example shows how to design MCP tools with appropriate scopes:
- **Read tools** for safe exploration
- **Write tools** with validation and idempotency
- **Destructive tools** with confirmation requirements

The scope system ensures agents can only perform operations they're authorized for, preventing accidental or malicious damage while enabling productive agent behavior.
