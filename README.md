Cryptocurrency Matching Engine
A high-performance cryptocurrency matching engine implementing REG NMS-inspired principles of price-time priority and internal order protection.

System Architecture
The system is built with the following components:

Matching Engine Core: Implements the order matching logic with price-time priority.
Order Book: Maintains the order book for each trading pair with efficient data structures.
REST API: For order submission and market data retrieval.
WebSocket API: For real-time market data dissemination and trade execution feeds.
Persistence Layer: SQLite-based storage for recovering state after crashes or restarts.
Data Structures
The system uses the following key data structures:

SortedDict for Price Levels:

Bids are sorted in descending order (highest price first)
Asks are sorted in ascending order (lowest price first)
This allows O(1) access to the best price levels
Order Lists at Each Price Level:

Orders at the same price are stored in a list in time priority order (FIFO)
This ensures that orders are matched according to time priority
Order ID Lookup Dictionary:

Allows O(1) access to any order by its ID
Used for efficient order cancellation and modification
Pending Trigger Orders:

Stores stop and take-profit orders waiting to be triggered
Orders are checked against each new trade price
Matching Algorithm
The matching algorithm implements strict price-time priority:

For incoming marketable orders, the system first checks the opposite side of the book.
Orders are matched at the best available price first.
At each price level, orders are matched in time priority (FIFO).
The system prevents internal trade-throughs by always matching at the best available price.
Order Types
The system supports seven order types:

Market Order: Executes immediately at the best available price(s).
Limit Order: Executes at the specified price or better. Rests on the book if not immediately marketable.
Immediate-Or-Cancel (IOC): Executes all or part of the order immediately and cancels any unfilled portion.
Fill-Or-Kill (FOK): Executes the entire order immediately or cancels the entire order if it cannot be fully filled.
Stop-Loss Order: Becomes a market order when the price reaches the specified stop price.
Stop-Limit Order: Becomes a limit order when the price reaches the specified stop price.
Take-Profit Order: Executes when the price reaches a specified profit target.
API Specifications
REST API
POST /orders: Submit a new order
DELETE /orders/{order_id}: Cancel an existing order
GET /orders/{order_id}: Get details of an existing order
GET /market-data/{symbol}/bbo: Get the current Best Bid and Offer
GET /market-data/{symbol}/order-book: Get the current order book
GET /market-data/{symbol}/trades: Get recent trades
WebSocket API
/ws/bbo: Stream real-time BBO updates
/ws/order-book: Stream real-time order book updates
/ws/trades: Stream real-time trade execution updates
Persistence Layer
The system includes a SQLite-based persistence layer that:

Saves state automatically:

Every 60 seconds during normal operation
During graceful shutdowns (SIGINT, SIGTERM)
Recovers state automatically on startup:

Orders (open, partially filled, and pending trigger)
Fee schedules
Recent trades
Maintains data integrity across application restarts:

Order books are reconstructed with correct price-time priority
Pending trigger orders are restored and monitored
Custom fee schedules are preserved
The database schema includes tables for:

Orders
Trades
Fee schedules
Default fee rates
Trade-off Decisions
In-Memory with SQLite Persistence:

The system is designed as an in-memory matching engine for maximum performance.
A SQLite persistence layer is implemented to survive crashes and application restarts.
State is automatically saved periodically and during graceful shutdowns.
Data Structure Choices:

SortedDict provides O(log n) insertion/deletion and O(1) access to the best price.
This is a good balance between performance and code simplicity.
Concurrency Model:

The current implementation is single-threaded for simplicity.
In a production environment, a more sophisticated concurrency model would be needed.
Installation and Setup
Install the required dependencies:
pip install -r requirements.txt
Run the application:
python -m app.main
By default, the database is stored in trading_app.db. You can change this by setting the DB_PATH environment variable:

DB_PATH=/path/to/database.db python -m app.main
Usage Guide
Submitting Orders via REST API
# Submit a limit buy order
curl -X POST "http://localhost:8000/orders" \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "BTC-USDT",
    "order_type": "limit",
    "side": "buy",
    "quantity": 1.0,
    "price": 50000.0
  }'

# Submit a market sell order
curl -X POST "http://localhost:8000/orders" \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "BTC-USDT",
    "order_type": "market",
    "side": "sell",
    "quantity": 0.5
  }'

# Submit a stop-loss order
curl -X POST "http://localhost:8000/orders" \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "BTC-USDT",
    "order_type": "stop_loss",
    "side": "sell",
    "quantity": 1.0,
    "stop_price": 49000.0
  }'

# Submit a stop-limit order
curl -X POST "http://localhost:8000/orders" \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "BTC-USDT",
    "order_type": "stop_limit",
    "side": "buy",
    "quantity": 1.0,
    "stop_price": 51000.0,
    "limit_price": 51500.0
  }'

# Submit a take-profit order
curl -X POST "http://localhost:8000/orders" \
  -H "Content-Type: application/json" \
  -d '{
    "symbol": "BTC-USDT",
    "order_type": "take_profit",
    "side": "buy",
    "quantity": 1.0,
    "stop_price": 49000.0
  }'
Connecting to WebSocket Feeds
// Connect to the BBO feed
const bbosocket = new WebSocket('ws://localhost:8000/ws/bbo');

bbosocket.onopen = function(e) {
  // Subscribe to a symbol
  bbosocket.send(JSON.stringify({
    action: 'subscribe',
    symbol: 'BTC-USDT'
  }));
};

bbosocket.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('BBO Update:', data);
};

// Connect to the trades feed
const tradesSocket = new WebSocket('ws://localhost:8000/ws/trades');

tradesSocket.onopen = function(e) {
  // Subscribe to a symbol
  tradesSocket.send(JSON.stringify({
    action: 'subscribe',
    symbol: 'BTC-USDT'
  }));
};

tradesSocket.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('Trade Update:', data);
};
Running Tests
pytest
For testing advanced order types specifically:

pytest tests/test_advanced_orders.py
For testing the persistence layer:

pytest tests/test_persistence.py
Fee Model
The system implements a maker-taker fee model:

Maker fees (typically lower): Applied to orders that provide liquidity (limit orders that don't execute immediately)
Taker fees (typically higher): Applied to orders that remove liquidity (market orders or limit orders that execute immediately)
Default fee rates:

Maker fee: 0.1% (0.001)
Taker fee: 0.2% (0.002)
Custom fee schedules can be set for specific trading pairs:

# Set custom fee schedule for BTC-USDT
curl -X POST "http://localhost:8000/fee-schedules/BTC-USDT?maker_rate=0.0005&taker_rate=0.001"

# Get fee schedule for BTC-USDT
curl -X GET "http://localhost:8000/fee-schedules/BTC-USDT"
