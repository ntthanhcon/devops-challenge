Provide your CLI command here:

## CLI Command

```bash
cat transaction-log.txt | jq -r 'select(.symbol == "TSLA" and .side == "sell") | .order_id' | while read order_id; do curl -s "https://example.com/api/$order_id"; done > output.txt
```

## Explanation

This command performs the following operations:

1. Reads the transaction log file (`transaction-log.txt`)
2. Uses `jq` to filter for Tesla ("TSLA") sell orders and extract their order IDs
3. For each order ID, makes a GET request to the API endpoint
4. Saves all API responses to `output.txt`

The command will process orders 12346 and 12362 (the TSLA sell orders) and make GET requests to:
- https://example.com/api/12346
- https://example.com/api/12362

