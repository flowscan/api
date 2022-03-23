# Temporary Flowscan API Documentation

> :warning: This documentation is temporary, we will do a stable release of our APIs at the end of March 2022. 

In this document we would like to describe our API endpoints that can be used.

Currently due to infrastructure limitations we need to know our users usage before we can provide the necessary documentation to use our API. At the end of this month we will migrate our system to GCP so that we can scale our infrastructure better, along with the necessary API key management system and a stable public documentation of our APIs. Until then, please read this document below to view the possibilities and limitations of our current API state.

## General Limitations

- Every request to our API endpoint is still rate-limited at 6 requests / second per-IP
- GraphQL types does not have descriptions yet 

## GraphQL Queries

Our production endpoint is at `https://flowscan.org/query`

Please check out the `schema.graphql` file in this repository as a reference of the types used. 

#### Pagination

A lot of our queries below will need to use some kind of pagination. We are using [Relay's Connection Specification](https://relay.dev/graphql/connections.htm) to serve paginated results. In short: 

All connection results will have a `PageInfo` to describe the current page. They contains `hasNextPage` to know if there are more data after the current page.

Some connections have a `count` field to get the total count of something without quering the whole data set.

To do pagination, other than using offsets for pagination, use cursors. 

#### Decimals

Some cadence values are using [Fixed-point](https://en.wikipedia.org/wiki/Fixed-point_arithmetic) values using **8 decimal places** to represent numbers with accurate decimal values. Since the JSON specification does not have a big number support, we're using `string` to represent them accurately.

So for example, this string: `"1234567890"` is used to represent `12.3456789` decimal value, while this string: `"700000000000"` is used to represent the value of `7000`. The last 8 digits of the string is used for decimals.

### Stable Queries 

These queries are stable, 
- They won't have any major changes, although minor changes such as the change of field names are possible (we will communicate these first in our Discord channel)
- These queries should be quite performant

#### Querying token transfers from/to an account

```graphql
query AccountTransfersQuery (
    $address: ID!, 
    $limit: Int!, 
    $after: ID
) {
  account(id: $address) {
    id
    queryResult: transfers(
      take: $limit
      after: $after
    ) {
      pageInfo {
        hasNextPage
        endCursor
      }
      count
      edges {
        node {
          transaction {
            id
            time
          }
          fungibleTokenTransfers {
            type
            amount {
              token {
                id
              }
              value
            }
            transaction {
              id
              time
            }

            # Counterparty is the other account as the sender / receiver of this transfer
            counterparty {
              id
            }
            counterpartiesCount
          }
          nftTransfers {
            from {
              id
            }
            to {
              id
            }
            nft {
              contract {
                id
              }
              nftId
            }
            transaction {
              id
              time
            }
          }
        }
      }
    }
  }
}
```

Result with variables: `{ "address": "0x6394e988297f5ed2", "limit": 3 }`: 
```JSON
{
  "data": {
    "account": {
      "id": "0x6394e988297f5ed2",
      "queryResult": {
        "pageInfo": {
          "hasNextPage": true,
          "endCursor": "45572413bb42a16b87e990418415fad202387bf3545e10677b723eea2cab7270"
        },
        "count": 144, // this is the TOTAL count of all transfers from/to this address
        "edges": [
          {
            "node": {
              "transaction": {
                "id": "5b1ce4157799885c80f1425993eb692f936255d069b51b30ff27f1f155b9c3da",
                "time": 1646289399
              },
              "fungibleTokenTransfers": [
                {
                  "type": "Deposit",
                  "amount": {
                    "token": {
                      "id": "FT.A.1654653399040a61.FlowToken"
                    },
                    "value": "22379776"
                  },
                  "transaction": {
                    "id": "5b1ce4157799885c80f1425993eb692f936255d069b51b30ff27f1f155b9c3da",
                    "time": 1646289399
                  },
                  "counterparty": {
                    "id": "0x2b223b9088d8634a"
                  },
                  "counterpartiesCount": 2
                }
              ],
              "nftTransfers": []
            }
          },
          {
            "node": {
              "transaction": {
                "id": "880dd29c19fc28b49f867ce687bf0d5261bc935735f6ab996d1d6975aec647ca",
                "time": 1645771229
              },
              "fungibleTokenTransfers": [
                {
                  "type": "Withdraw",
                  "amount": {
                    "token": {
                      "id": "FT.A.0f9df91c9121c460.BloctoToken"
                    },
                    "value": "596279340000"
                  },
                  "transaction": {
                    "id": "880dd29c19fc28b49f867ce687bf0d5261bc935735f6ab996d1d6975aec647ca",
                    "time": 1645771229
                  },
                  "counterparty": {
                    "id": "0x9c4370568ea7bf19"
                  },
                  "counterpartiesCount": 1
                }
              ],
              "nftTransfers": []
            }
          },
          {
            "node": {
              "transaction": {
                "id": "45572413bb42a16b87e990418415fad202387bf3545e10677b723eea2cab7270",
                "time": 1645771190
              },
              "fungibleTokenTransfers": [
                {
                  "type": "Withdraw",
                  "amount": {
                    "token": {
                      "id": "FT.A.0f9df91c9121c460.BloctoToken"
                    },
                    "value": "357767604000"
                  },
                  "transaction": {
                    "id": "45572413bb42a16b87e990418415fad202387bf3545e10677b723eea2cab7270",
                    "time": 1645771190
                  },
                  "counterparty": {
                    "id": "0xd3203f2c8a59dae1"
                  },
                  "counterpartiesCount": 1
                }
              ],
              "nftTransfers": []
            }
          }
        ]
      }
    }
  }
}
```

##### Pagination 

As you can see, this query returns a connection type, therefore you can query next page by adding the `$after` variable to the `endCursor` value (in the above example, it is `45572413bb42a16b87e990418415fad202387bf3545e10677b723eea2cab7270`): 

Variables: `{ "address": "0x6394e988297f5ed2", "limit": 3, "after": "45572413bb42a16b87e990418415fad202387bf3545e10677b723eea2cab7270" }`: 
```JSON
{
  "data": {
    "account": {
      "id": "0x6394e988297f5ed2",
      "queryResult": {
        "pageInfo": {
          "hasNextPage": true,
          "endCursor": "e395c6d2e44b9fa8bc691e5a4810fbd437210fecbd74f59afd4bc400c5a9747f"
        },
        "count": 144,
        "edges": [
          {
            "node": {
              "transaction": {
                "id": "7615384c65bab8d93daef0c9637ef5321041b3dbeeb320c0d751e72ff270cbab",
                "time": 1645771166
              },
              "fungibleTokenTransfers": [
                {
                  "type": "Withdraw",
                  "amount": {
                    "token": {
                      "id": "FT.A.0f9df91c9121c460.BloctoToken"
                    },
                    "value": "596279340000"
                  },
                  "transaction": {
                    "id": "7615384c65bab8d93daef0c9637ef5321041b3dbeeb320c0d751e72ff270cbab",
                    "time": 1645771166
                  },
                  "counterparty": {
                    "id": "0x8af95ef8117f2e1e"
                  },
                  "counterpartiesCount": 1
                }
              ],
              "nftTransfers": []
            }
          },
          {
            "node": {
              "transaction": {
                "id": "aecb3e6e7ae506d8ad0232e8ae802b1882f9b760c63a360bfdbd5847ff3b8a84",
                "time": 1645771126
              },
              "fungibleTokenTransfers": [
                {
                  "type": "Withdraw",
                  "amount": {
                    "token": {
                      "id": "FT.A.0f9df91c9121c460.BloctoToken"
                    },
                    "value": "298139670000"
                  },
                  "transaction": {
                    "id": "aecb3e6e7ae506d8ad0232e8ae802b1882f9b760c63a360bfdbd5847ff3b8a84",
                    "time": 1645771126
                  },
                  "counterparty": {
                    "id": "0x5363721f0e222b55"
                  },
                  "counterpartiesCount": 1
                }
              ],
              "nftTransfers": []
            }
          },
          {
            "node": {
              "transaction": {
                "id": "e395c6d2e44b9fa8bc691e5a4810fbd437210fecbd74f59afd4bc400c5a9747f",
                "time": 1645771072
              },
              "fungibleTokenTransfers": [
                {
                  "type": "Withdraw",
                  "amount": {
                    "token": {
                      "id": "FT.A.0f9df91c9121c460.BloctoToken"
                    },
                    "value": "715535208000"
                  },
                  "transaction": {
                    "id": "e395c6d2e44b9fa8bc691e5a4810fbd437210fecbd74f59afd4bc400c5a9747f",
                    "time": 1645771072
                  },
                  "counterparty": {
                    "id": "0x03667c1b50d8fab3"
                  },
                  "counterpartiesCount": 1
                }
              ],
              "nftTransfers": []
            }
          }
        ]
      }
    }
  }
}
```

#### Querying account token balances 

Reminder that `value` is a fixed point decimal.

```graphql
query AccountBalancesQuery ($address: ID!){
  account(id: $address) {
    balances {
      count
      edges {
        amount {
          token {
            id
            label
            ticker
            description
            price
          }
          value # THIS IS A FIXED POINT
        }
      }
    }
  }
}
```

Result: 
```json
{
  "data": {
    "account": {
      "balances": {
        "count": 4,
        "edges": [
          {
            "amount": {
              "token": {
                "id": "FT.A.3c5959b568896393.FUSD",
                "label": "Flow USD",
                "ticker": "FUSD",
                "description": "USD stablecoin on Flow",
                "price": 1
              },
              "value": "274498169014" // This is a fixed point! It represents 2744.98169014
            }
          },
          {
            "amount": {
              "token": {
                "id": "FT.A.1654653399040a61.FlowToken",
                "label": "Flow Token",
                "ticker": "FLOW",
                "description": "The native currency for the Flow blockchain",
                "price": 5.64
              },
              "value": "384400329"
            }
          },
          {
            "amount": {
              "token": {
                "id": "FT.A.0f9df91c9121c460.BloctoToken",
                "label": "Blocto Token",
                "ticker": "BLT",
                "description": null,
                "price": 0.429519
              },
              "value": "0"
            }
          },
          {
            "amount": {
              "token": {
                "id": "FT.A.cfdd90d4a00f7b5b.TeleportedTetherToken",
                "label": "Teleported USDT",
                "ticker": "tUSDT",
                "description": "USDT teleported from Ethereum using the Blocto bridge",
                "price": 1
              },
              "value": "0"
            }
          }
        ]
      }
    }
  }
}
```

#### Querying token transfers by contract

Used to query token transfers by contract.

```graphql
query TokenTransfersQuery (
    $id: ID!, 
    $limit: Int!, 
    $after: ID
) {
  contract(id: $id) {
    tokenInfo {
      label
      transfers(
        take: $limit
        after: $after
      ) {
          pageInfo {
            hasNextPage
            endCursor
          }
          count
          edges {
            node {
              transaction {
                id
                time
              }
              fungibleTokenTransfers {
                account {
                  id
                }
                type
                amount {
                  value
                }
              }
            }
          }
        }
      }
    }
  }
}
```

Result with variables `{ "id": "A.1654653399040a61.FlowToken", "limit": 3 }`: 
```JSON
{
  "data": {
    "contract": {
      "tokenInfo": {
        "label": "Flow Token",
        "transfers": {
          "pageInfo": {
            "hasNextPage": true,
            "endCursor": "61ebf5acc0f4ea623d65c24441d8ad164906c52e36bb3f60e9df37e817ad8761"
          },
          "count": 87998060,
          "edges": [
            {
              "node": {
                "transaction": {
                  "id": "87e5722d78a5c7a46b1da40d07ff94e9bb7823b2a3ac4a0423b8573836ecec6e",
                  "time": 1646728956
                },
                "fungibleTokenTransfers": [
                  {
                    "account": {
                      "id": "0x18eb4ee6b3c026d2"
                    },
                    "type": "Withdraw",
                    "amount": {
                      "value": "1000"
                    }
                  },
                  {
                    "account": {
                      "id": "0xf919ee77447b7497"
                    },
                    "type": "Deposit",
                    "amount": {
                      "value": "1000"
                    }
                  }
                ]
              }
            },
            {
              "node": {
                "transaction": {
                  "id": "6141fffd675a9ece553920b6094ffc51638fa7ebe49584d1c9a3cf1069ec498a",
                  "time": 1646728954
                },
                "fungibleTokenTransfers": [
                  {
                    "account": {
                      "id": "0x18eb4ee6b3c026d2"
                    },
                    "type": "Withdraw",
                    "amount": {
                      "value": "1000"
                    }
                  },
                  {
                    "account": {
                      "id": "0xf919ee77447b7497"
                    },
                    "type": "Deposit",
                    "amount": {
                      "value": "1000"
                    }
                  }
                ]
              }
            },
            {
              "node": {
                "transaction": {
                  "id": "61ebf5acc0f4ea623d65c24441d8ad164906c52e36bb3f60e9df37e817ad8761",
                  "time": 1646728954
                },
                "fungibleTokenTransfers": [
                  {
                    "account": {
                      "id": "0x18eb4ee6b3c026d2"
                    },
                    "type": "Withdraw",
                    "amount": {
                      "value": "1000"
                    }
                  },
                  {
                    "account": {
                      "id": "0xf919ee77447b7497"
                    },
                    "type": "Deposit",
                    "amount": {
                      "value": "1000"
                    }
                  }
                ]
              }
            }
          ]
        }
      }
    }
  }
}
```

Same with [account transfers query](#querying-token-transfers-from/to-an-account), this query is paginate-able with cursors. 

### Unstable Queries

These queries are unstable, expect major changes to happen. Although we **will communicate these first in our Discord channel**.

#### Querying account transactions

> :warning: This query can still be paginated, however it is still using an `offset`-based pagination which is **REALLY** slow. Before we can fix this issue, please don't query further than ~100 records using offset.

Expect this query to be finalized in the end of March 2022 with a cursor-based pagination and better performance. This is one of our top priority at the moment.

```graphql
  query AccountTransactionsQuery($address: ID!, $role: TransactionRole, $limit: Int!, $offset: Int) {
  account(id: $address) {
    transactions(
      first: $limit
      skip: $offset
      role: $role
    ) {
      count
      edges {
        node {
          hash
          time
          authorizers {
            address
          }
          payer {
            id
          }
          proposer {
            id
          }
          contractInteractions {
            id
          }
          status
        }
      }
    }
  }
}
```

Result with `{ "address": "0x6394e988297f5ed2", "limit": 3 }`:
```json
{
  "data": {
    "account": {
      "transactions": {
        "count": 120,
        "edges": [
          {
            "node": {
              "hash": "5b1ce4157799885c80f1425993eb692f936255d069b51b30ff27f1f155b9c3da",
              "time": 1646289399,
              "authorizers": [
                {
                  "address": "0x2b223b9088d8634a"
                }
              ],
              "payer": {
                "id": "0x55ad22f01ef568a1"
              },
              "proposer": {
                "id": "0x6394e988297f5ed2"
              },
              "contractInteractions": [
                {
                  "id": "A.1654653399040a61.FlowToken"
                },
                {
                  "id": "A.f919ee77447b7497.FlowFees"
                }
              ],
              "status": "SEALED"
            }
          },
          {
            "node": {
              "hash": "880dd29c19fc28b49f867ce687bf0d5261bc935735f6ab996d1d6975aec647ca",
              "time": 1645771229,
              "authorizers": [
                {
                  "address": "0x6394e988297f5ed2"
                }
              ],
              "payer": {
                "id": "0x55ad22f01ef568a1"
              },
              "proposer": {
                "id": "0x6394e988297f5ed2"
              },
              "contractInteractions": [
                {
                  "id": "A.0f9df91c9121c460.BloctoToken"
                },
                {
                  "id": "A.1654653399040a61.FlowToken"
                },
                {
                  "id": "A.f919ee77447b7497.FlowFees"
                }
              ],
              "status": "SEALED"
            }
          },
          {
            "node": {
              "hash": "45572413bb42a16b87e990418415fad202387bf3545e10677b723eea2cab7270",
              "time": 1645771190,
              "authorizers": [
                {
                  "address": "0x6394e988297f5ed2"
                }
              ],
              "payer": {
                "id": "0x55ad22f01ef568a1"
              },
              "proposer": {
                "id": "0x6394e988297f5ed2"
              },
              "contractInteractions": [
                {
                  "id": "A.0f9df91c9121c460.BloctoToken"
                },
                {
                  "id": "A.1654653399040a61.FlowToken"
                },
                {
                  "id": "A.f919ee77447b7497.FlowFees"
                }
              ],
              "status": "SEALED"
            }
          }
        ]
      }
    }
  }
}
```

##### Pagination 

This is an old query so it's still using an offset-based pagination. Set the offset value to paginate into next records. However high offset value could result in a really slow query, before we can fix this issue, please don't query further than ~100 records using offset.

#### Metrics queries

Query time-series data.

```graphql
metrics(metric: "TRANSACTION", interval: Daily) {
  time
  value
}
```

Available metrics: 
- `"BLOCK"` for blocks / day
- `"TRANSACTION"` for transactions / day
- `"CREATED_ACCOUNT"` for created accounts / day
- `"UNIQUE_ADDRESS"` for number of unique users that sent a transaction in a day
