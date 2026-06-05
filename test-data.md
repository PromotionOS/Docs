# PromotionOS — Real World Test Data

> Dense synthetic data shaped to mirror real Kroger retail scenarios.
> Covers happy paths, edge cases, boundary conditions, and volume scenarios.
> This is the source of truth — all domain logic, contracts, and validation build on top of this.

---

## Tenant

```json
[
  {
    "tenantId": "tenant-kroger-001",
    "name": "Kroger",
    "region": "US"
  }
]
```

---

## Divisions (Geo Hierarchy)

```json
[
  { "id": "division-all",       "name": "All Divisions",  "stores": 2800 },
  { "id": "division-midwest",   "name": "Midwest",        "stores": 620  },
  { "id": "division-southeast", "name": "Southeast",      "stores": 510  },
  { "id": "division-west",      "name": "West",           "stores": 430  },
  { "id": "division-northeast", "name": "Northeast",      "stores": 390  },
  { "id": "division-southwest", "name": "Southwest",      "stores": 280  }
]
```

---

## Catalog — Categories and UPCs

### Category Hierarchy

```json
[
  {
    "id": "cat-beverages",
    "name": "Beverages",
    "parentId": null,
    "excluded": false,
    "excludeReason": null,
    "children": ["cat-soft-drinks", "cat-juice", "cat-water", "cat-alcohol", "cat-energy-drinks"]
  },
  {
    "id": "cat-soft-drinks",
    "name": "Soft Drinks",
    "parentId": "cat-beverages",
    "excluded": false,
    "excludeReason": null,
    "children": []
  },
  {
    "id": "cat-juice",
    "name": "Juice",
    "parentId": "cat-beverages",
    "excluded": false,
    "excludeReason": null,
    "children": []
  },
  {
    "id": "cat-water",
    "name": "Water",
    "parentId": "cat-beverages",
    "excluded": false,
    "excludeReason": null,
    "children": []
  },
  {
    "id": "cat-energy-drinks",
    "name": "Energy Drinks",
    "parentId": "cat-beverages",
    "excluded": false,
    "excludeReason": null,
    "children": []
  },
  {
    "id": "cat-alcohol",
    "name": "Alcohol",
    "parentId": "cat-beverages",
    "excluded": true,
    "excludeReason": "REGULATORY",
    "children": ["cat-wine", "cat-beer", "cat-spirits"]
  },
  {
    "id": "cat-wine",
    "name": "Wine",
    "parentId": "cat-alcohol",
    "excluded": true,
    "excludedInheritedFrom": "cat-alcohol",
    "excludeReason": "REGULATORY_INHERITED",
    "children": ["cat-wine-red", "cat-wine-white", "cat-wine-rose"]
  },
  {
    "id": "cat-wine-red",
    "name": "Red Wine",
    "parentId": "cat-wine",
    "excluded": true,
    "excludedInheritedFrom": "cat-alcohol",
    "excludeReason": "REGULATORY_INHERITED",
    "children": []
  },
  {
    "id": "cat-wine-white",
    "name": "White Wine",
    "parentId": "cat-wine",
    "excluded": true,
    "excludedInheritedFrom": "cat-alcohol",
    "excludeReason": "REGULATORY_INHERITED",
    "children": []
  },
  {
    "id": "cat-wine-rose",
    "name": "Rose Wine",
    "parentId": "cat-wine",
    "excluded": true,
    "excludedInheritedFrom": "cat-alcohol",
    "excludeReason": "REGULATORY_INHERITED",
    "children": []
  },
  {
    "id": "cat-beer",
    "name": "Beer",
    "parentId": "cat-alcohol",
    "excluded": true,
    "excludedInheritedFrom": "cat-alcohol",
    "excludeReason": "REGULATORY_INHERITED",
    "children": []
  },
  {
    "id": "cat-spirits",
    "name": "Spirits",
    "parentId": "cat-alcohol",
    "excluded": true,
    "excludedInheritedFrom": "cat-alcohol",
    "excludeReason": "REGULATORY_INHERITED",
    "children": []
  },
  {
    "id": "cat-tobacco",
    "name": "Tobacco",
    "parentId": null,
    "excluded": true,
    "excludeReason": "REGULATORY",
    "children": ["cat-cigarettes", "cat-cigars", "cat-smokeless"]
  },
  {
    "id": "cat-cigarettes",
    "name": "Cigarettes",
    "parentId": "cat-tobacco",
    "excluded": true,
    "excludedInheritedFrom": "cat-tobacco",
    "excludeReason": "REGULATORY_INHERITED",
    "children": []
  },
  {
    "id": "cat-cigars",
    "name": "Cigars",
    "parentId": "cat-tobacco",
    "excluded": true,
    "excludedInheritedFrom": "cat-tobacco",
    "excludeReason": "REGULATORY_INHERITED",
    "children": []
  },
  {
    "id": "cat-smokeless",
    "name": "Smokeless Tobacco",
    "parentId": "cat-tobacco",
    "excluded": true,
    "excludedInheritedFrom": "cat-tobacco",
    "excludeReason": "REGULATORY_INHERITED",
    "children": []
  },
  {
    "id": "cat-snacks",
    "name": "Snacks",
    "parentId": null,
    "excluded": false,
    "children": ["cat-chips", "cat-crackers", "cat-nuts", "cat-popcorn"]
  },
  {
    "id": "cat-chips",     "name": "Chips",     "parentId": "cat-snacks",  "excluded": false, "children": [] },
  {
    "id": "cat-crackers",  "name": "Crackers",  "parentId": "cat-snacks",  "excluded": false, "children": [] },
  {
    "id": "cat-nuts",      "name": "Nuts",      "parentId": "cat-snacks",  "excluded": false, "children": [] },
  {
    "id": "cat-popcorn",   "name": "Popcorn",   "parentId": "cat-snacks",  "excluded": false, "children": [] },
  {
    "id": "cat-breakfast",
    "name": "Breakfast",
    "parentId": null,
    "excluded": false,
    "children": ["cat-cereal", "cat-oatmeal", "cat-granola", "cat-breakfast-bars"]
  },
  {
    "id": "cat-cereal",         "name": "Cereal",          "parentId": "cat-breakfast", "excluded": false, "children": [] },
  {
    "id": "cat-oatmeal",        "name": "Oatmeal",         "parentId": "cat-breakfast", "excluded": false, "children": [] },
  {
    "id": "cat-granola",        "name": "Granola",         "parentId": "cat-breakfast", "excluded": false, "children": [] },
  {
    "id": "cat-breakfast-bars", "name": "Breakfast Bars",  "parentId": "cat-breakfast", "excluded": false, "children": [] },
  {
    "id": "cat-meat",
    "name": "Meat and Deli",
    "parentId": null,
    "excluded": false,
    "children": ["cat-beef", "cat-poultry", "cat-pork", "cat-deli"]
  },
  {
    "id": "cat-beef",    "name": "Beef",    "parentId": "cat-meat", "excluded": false, "children": [] },
  {
    "id": "cat-poultry", "name": "Poultry", "parentId": "cat-meat", "excluded": false, "children": [] },
  {
    "id": "cat-pork",    "name": "Pork",    "parentId": "cat-meat", "excluded": false, "children": [] },
  {
    "id": "cat-deli",    "name": "Deli",    "parentId": "cat-meat", "excluded": false, "children": [] },
  {
    "id": "cat-condiments", "name": "Condiments", "parentId": null, "excluded": false, "children": [] },
  {
    "id": "cat-pharmacy",
    "name": "Pharmacy",
    "parentId": null,
    "excluded": false,
    "children": ["cat-vitamins", "cat-otc-medicine", "cat-personal-care"]
  },
  {
    "id": "cat-vitamins",      "name": "Vitamins",       "parentId": "cat-pharmacy", "excluded": false, "children": [] },
  {
    "id": "cat-otc-medicine",  "name": "OTC Medicine",   "parentId": "cat-pharmacy", "excluded": false, "children": [] },
  {
    "id": "cat-personal-care", "name": "Personal Care",  "parentId": "cat-pharmacy", "excluded": false, "children": [] },
  {
    "id": "cat-bakery",   "name": "Bakery and Bread", "parentId": null, "excluded": false, "children": [] },
  {
    "id": "cat-dairy",    "name": "Dairy",            "parentId": null, "excluded": false, "children": [] },
  {
    "id": "cat-frozen",   "name": "Frozen Foods",     "parentId": null, "excluded": false, "children": [] },
  {
    "id": "cat-produce",  "name": "Produce",          "parentId": null, "excluded": false, "children": [] },
  {
    "id": "cat-household","name": "Household",        "parentId": null, "excluded": false, "children": [] }
]
```

### UPCs

```json
[
  { "code": "upc-cola-12pk",         "name": "Pepsi Cola 12pk",              "price": 6.99,  "categoryId": "cat-soft-drinks",    "tenantId": "tenant-kroger-001" },
  { "code": "upc-diet-cola-12pk",    "name": "Diet Pepsi 12pk",             "price": 6.99,  "categoryId": "cat-soft-drinks",    "tenantId": "tenant-kroger-001" },
  { "code": "upc-cola-2l",           "name": "Pepsi Cola 2L",               "price": 2.49,  "categoryId": "cat-soft-drinks",    "tenantId": "tenant-kroger-001" },
  { "code": "upc-oj-64",             "name": "Tropicana OJ 64oz",           "price": 5.99,  "categoryId": "cat-juice",          "tenantId": "tenant-kroger-001" },
  { "code": "upc-apple-juice-64",    "name": "Mott's Apple Juice 64oz",     "price": 4.49,  "categoryId": "cat-juice",          "tenantId": "tenant-kroger-001" },
  { "code": "upc-water-24pk",        "name": "Aquafina Water 24pk",         "price": 4.99,  "categoryId": "cat-water",          "tenantId": "tenant-kroger-001" },
  { "code": "upc-sparkling-12pk",    "name": "LaCroix Sparkling 12pk",      "price": 5.49,  "categoryId": "cat-water",          "tenantId": "tenant-kroger-001" },
  { "code": "upc-energy-4pk",        "name": "Red Bull 4pk",                "price": 9.99,  "categoryId": "cat-energy-drinks",  "tenantId": "tenant-kroger-001" },
  { "code": "upc-wine-cab",          "name": "Cabernet Sauvignon 750ml",    "price": 14.99, "categoryId": "cat-wine-red",       "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-wine-pinot",        "name": "Pinot Noir 750ml",            "price": 16.99, "categoryId": "cat-wine-red",       "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-wine-chard",        "name": "Chardonnay 750ml",            "price": 12.99, "categoryId": "cat-wine-white",     "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-wine-sauv-blanc",   "name": "Sauvignon Blanc 750ml",       "price": 13.99, "categoryId": "cat-wine-white",     "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-wine-rose",         "name": "Rose 750ml",                  "price": 11.99, "categoryId": "cat-wine-rose",      "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-beer-6pk-ipa",      "name": "Goose Island IPA 6pk",        "price": 10.99, "categoryId": "cat-beer",           "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-beer-12pk-lager",   "name": "Bud Light 12pk",              "price": 14.99, "categoryId": "cat-beer",           "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-spirits-vodka",     "name": "Tito's Vodka 750ml",          "price": 24.99, "categoryId": "cat-spirits",        "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-cigarettes-marlboro","name": "Marlboro Red 20pk",           "price": 9.99,  "categoryId": "cat-cigarettes",     "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-cigarettes-camel",  "name": "Camel Blue 20pk",             "price": 9.49,  "categoryId": "cat-cigarettes",     "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-cigars-cohiba",     "name": "Cohiba Cigars 5pk",           "price": 29.99, "categoryId": "cat-cigars",         "tenantId": "tenant-kroger-001", "excluded": true },
  { "code": "upc-chips-family",      "name": "Lays Family Size",            "price": 5.49,  "categoryId": "cat-chips",          "tenantId": "tenant-kroger-001" },
  { "code": "upc-chips-ruffles",     "name": "Ruffles Family Size",         "price": 5.49,  "categoryId": "cat-chips",          "tenantId": "tenant-kroger-001" },
  { "code": "upc-crackers-ritz",     "name": "Ritz Crackers 16oz",          "price": 4.29,  "categoryId": "cat-crackers",       "tenantId": "tenant-kroger-001" },
  { "code": "upc-nuts-mixed",        "name": "Planters Mixed Nuts 16oz",    "price": 8.99,  "categoryId": "cat-nuts",           "tenantId": "tenant-kroger-001" },
  { "code": "upc-popcorn-orville",   "name": "Orville Redenbacher 6pk",     "price": 4.99,  "categoryId": "cat-popcorn",        "tenantId": "tenant-kroger-001" },
  { "code": "upc-cheerios",          "name": "Cheerios 18oz",               "price": 4.99,  "categoryId": "cat-cereal",         "tenantId": "tenant-kroger-001" },
  { "code": "upc-frosted-flakes",    "name": "Frosted Flakes 19.2oz",       "price": 4.49,  "categoryId": "cat-cereal",         "tenantId": "tenant-kroger-001" },
  { "code": "upc-granola",           "name": "Nature Valley Granola 16oz",  "price": 5.99,  "categoryId": "cat-granola",        "tenantId": "tenant-kroger-001" },
  { "code": "upc-oatmeal-quick",     "name": "Quaker Quick Oats 42oz",      "price": 6.49,  "categoryId": "cat-oatmeal",        "tenantId": "tenant-kroger-001" },
  { "code": "upc-granola-bar-6pk",   "name": "Nature Valley Bars 6pk",      "price": 4.29,  "categoryId": "cat-breakfast-bars", "tenantId": "tenant-kroger-001" },
  { "code": "upc-hotdogs",           "name": "Oscar Meyer Hotdogs 8pk",     "price": 4.99,  "categoryId": "cat-pork",           "tenantId": "tenant-kroger-001" },
  { "code": "upc-ground-beef-1lb",   "name": "Ground Beef 80/20 1lb",       "price": 5.99,  "categoryId": "cat-beef",           "tenantId": "tenant-kroger-001" },
  { "code": "upc-chicken-breast-2lb","name": "Chicken Breast 2lb",          "price": 8.99,  "categoryId": "cat-poultry",        "tenantId": "tenant-kroger-001" },
  { "code": "upc-buns",              "name": "Wonder Hotdog Buns 8pk",      "price": 2.49,  "categoryId": "cat-bakery",         "tenantId": "tenant-kroger-001" },
  { "code": "upc-bread-white",       "name": "Wonder White Bread 20oz",     "price": 2.99,  "categoryId": "cat-bakery",         "tenantId": "tenant-kroger-001" },
  { "code": "upc-ketchup",           "name": "Heinz Ketchup 32oz",          "price": 4.29,  "categoryId": "cat-condiments",     "tenantId": "tenant-kroger-001" },
  { "code": "upc-mustard",           "name": "French's Mustard 20oz",       "price": 2.99,  "categoryId": "cat-condiments",     "tenantId": "tenant-kroger-001" },
  { "code": "upc-mayo",              "name": "Hellmann's Mayo 30oz",        "price": 5.49,  "categoryId": "cat-condiments",     "tenantId": "tenant-kroger-001" },
  { "code": "upc-vitamin-c",         "name": "Vitamin C 1000mg 60ct",       "price": 12.99, "categoryId": "cat-vitamins",       "tenantId": "tenant-kroger-001" },
  { "code": "upc-vitamin-d",         "name": "Vitamin D 2000IU 90ct",       "price": 14.99, "categoryId": "cat-vitamins",       "tenantId": "tenant-kroger-001" },
  { "code": "upc-multivitamin",      "name": "Centrum Adults 100ct",        "price": 18.99, "categoryId": "cat-vitamins",       "tenantId": "tenant-kroger-001" },
  { "code": "upc-milk-gallon",       "name": "Kroger Whole Milk 1gal",      "price": 3.99,  "categoryId": "cat-dairy",          "tenantId": "tenant-kroger-001" },
  { "code": "upc-cheese-cheddar",    "name": "Tillamook Cheddar 32oz",      "price": 8.99,  "categoryId": "cat-dairy",          "tenantId": "tenant-kroger-001" },
  { "code": "upc-frozen-pizza",      "name": "DiGiorno Pepperoni 12in",     "price": 7.99,  "categoryId": "cat-frozen",         "tenantId": "tenant-kroger-001" },
  { "code": "upc-frozen-fries",      "name": "Ore-Ida French Fries 32oz",   "price": 4.99,  "categoryId": "cat-frozen",         "tenantId": "tenant-kroger-001" },
  { "code": "upc-bananas-bunch",     "name": "Bananas per bunch",           "price": 1.49,  "categoryId": "cat-produce",        "tenantId": "tenant-kroger-001" },
  { "code": "upc-apples-bag",        "name": "Gala Apples 3lb bag",         "price": 4.99,  "categoryId": "cat-produce",        "tenantId": "tenant-kroger-001" },
  { "code": "upc-paper-towels-6pk",  "name": "Bounty Paper Towels 6pk",     "price": 11.99, "categoryId": "cat-household",      "tenantId": "tenant-kroger-001" },
  { "code": "upc-dish-soap",         "name": "Dawn Dish Soap 21.6oz",       "price": 4.99,  "categoryId": "cat-household",      "tenantId": "tenant-kroger-001" }
]
```

---

## Customers

### Loyalty Tier Boundaries (for validation)

| Tier | Annual Spend |
|------|-------------|
| BASIC | < $500 |
| SILVER | $500 — $1,999 |
| GOLD | $2,000 — $4,999 |
| PLATINUM | $5,000+ |

### Customer Records

```json
[
  {
    "id": "cust-001",
    "tenantId": "tenant-kroger-001",
    "name": "Alice Johnson",
    "loyaltyTier": "PLATINUM",
    "annualSpend": 8500.00,
    "segments": ["segment-high-value", "segment-wine-buyer", "segment-midwest"],
    "division": "division-midwest",
    "last90DaysPurchases": [
      { "upc": "upc-wine-cab",      "qty": 4, "spend": 59.96,  "date": "2026-05-20" },
      { "upc": "upc-wine-chard",    "qty": 2, "spend": 25.98,  "date": "2026-05-18" },
      { "upc": "upc-cheerios",      "qty": 3, "spend": 14.97,  "date": "2026-05-15" },
      { "upc": "upc-cola-12pk",     "qty": 4, "spend": 27.96,  "date": "2026-05-10" },
      { "upc": "upc-cheese-cheddar","qty": 2, "spend": 17.98,  "date": "2026-05-08" },
      { "upc": "upc-chicken-breast-2lb", "qty": 3, "spend": 26.97, "date": "2026-05-01" }
    ]
  },
  {
    "id": "cust-002",
    "tenantId": "tenant-kroger-001",
    "name": "Bob Martinez",
    "loyaltyTier": "GOLD",
    "annualSpend": 3200.00,
    "segments": ["segment-mid-value", "segment-southeast"],
    "division": "division-southeast",
    "last90DaysPurchases": [
      { "upc": "upc-chips-family",   "qty": 5, "spend": 27.45, "date": "2026-05-22" },
      { "upc": "upc-hotdogs",        "qty": 3, "spend": 14.97, "date": "2026-05-19" },
      { "upc": "upc-frozen-pizza",   "qty": 4, "spend": 31.96, "date": "2026-05-15" },
      { "upc": "upc-cola-12pk",      "qty": 6, "spend": 41.94, "date": "2026-05-10" }
    ]
  },
  {
    "id": "cust-003",
    "tenantId": "tenant-kroger-001",
    "name": "Carol White",
    "loyaltyTier": "SILVER",
    "annualSpend": 1200.00,
    "segments": ["segment-low-value", "segment-midwest"],
    "division": "division-midwest",
    "last90DaysPurchases": [
      { "upc": "upc-bread-white",  "qty": 8, "spend": 23.92, "date": "2026-05-25" },
      { "upc": "upc-milk-gallon",  "qty": 6, "spend": 23.94, "date": "2026-05-20" },
      { "upc": "upc-oatmeal-quick","qty": 2, "spend": 12.98, "date": "2026-05-15" },
      { "upc": "upc-bananas-bunch","qty": 5, "spend": 7.45,  "date": "2026-05-10" }
    ]
  },
  {
    "id": "cust-004",
    "tenantId": "tenant-kroger-001",
    "name": "David Kim",
    "loyaltyTier": "BASIC",
    "annualSpend": 200.00,
    "segments": ["segment-new-customer"],
    "division": "division-southeast",
    "last90DaysPurchases": []
  },
  {
    "id": "cust-005",
    "tenantId": "tenant-kroger-001",
    "name": "Eve Thompson",
    "loyaltyTier": "PLATINUM",
    "annualSpend": 12000.00,
    "segments": ["segment-high-value", "segment-midwest", "segment-vitamin-buyer"],
    "division": "division-midwest",
    "last90DaysPurchases": [
      { "upc": "upc-vitamin-c",    "qty": 2, "spend": 25.98, "date": "2026-05-10" },
      { "upc": "upc-multivitamin", "qty": 1, "spend": 18.99, "date": "2026-05-10" },
      { "upc": "upc-cheerios",     "qty": 4, "spend": 19.96, "date": "2026-05-05" },
      { "upc": "upc-milk-gallon",  "qty": 8, "spend": 31.92, "date": "2026-04-20" }
    ]
  },
  {
    "id": "cust-006",
    "tenantId": "tenant-kroger-001",
    "name": "Frank Nguyen",
    "loyaltyTier": "GOLD",
    "annualSpend": 4999.00,
    "segments": ["segment-mid-value", "segment-west"],
    "division": "division-west",
    "last90DaysPurchases": [
      { "upc": "upc-ground-beef-1lb",    "qty": 8, "spend": 47.92, "date": "2026-05-20" },
      { "upc": "upc-chicken-breast-2lb", "qty": 5, "spend": 44.95, "date": "2026-05-15" },
      { "upc": "upc-frozen-pizza",       "qty": 6, "spend": 47.94, "date": "2026-05-10" }
    ],
    "edgeCase": "annualSpend exactly $1 below PLATINUM threshold"
  },
  {
    "id": "cust-007",
    "tenantId": "tenant-kroger-001",
    "name": "Grace Lee",
    "loyaltyTier": "PLATINUM",
    "annualSpend": 5000.00,
    "segments": ["segment-high-value", "segment-west"],
    "division": "division-west",
    "last90DaysPurchases": [
      { "upc": "upc-cheese-cheddar", "qty": 4, "spend": 35.96, "date": "2026-05-22" },
      { "upc": "upc-milk-gallon",    "qty": 6, "spend": 23.94, "date": "2026-05-18" }
    ],
    "edgeCase": "annualSpend exactly at PLATINUM threshold"
  },
  {
    "id": "cust-008",
    "tenantId": "tenant-kroger-001",
    "name": "Henry Park",
    "loyaltyTier": "SILVER",
    "annualSpend": 499.00,
    "segments": ["segment-low-value", "segment-northeast"],
    "division": "division-northeast",
    "last90DaysPurchases": [
      { "upc": "upc-bread-white", "qty": 4, "spend": 11.96, "date": "2026-05-20" },
      { "upc": "upc-milk-gallon", "qty": 3, "spend": 11.97, "date": "2026-05-15" }
    ],
    "edgeCase": "annualSpend $1 below SILVER threshold — should be BASIC"
  },
  {
    "id": "cust-009",
    "tenantId": "tenant-kroger-001",
    "name": "Iris Chen",
    "loyaltyTier": "GOLD",
    "annualSpend": 2800.00,
    "segments": ["segment-mid-value", "segment-wine-buyer", "segment-midwest"],
    "division": "division-midwest",
    "last90DaysPurchases": [
      { "upc": "upc-wine-cab",      "qty": 3, "spend": 44.97, "date": "2026-05-18" },
      { "upc": "upc-wine-chard",    "qty": 2, "spend": 25.98, "date": "2026-05-10" },
      { "upc": "upc-chips-family",  "qty": 4, "spend": 21.96, "date": "2026-05-05" }
    ],
    "edgeCase": "GOLD wine buyer — qualifies for general wine promo but not PLATINUM-exclusive"
  },
  {
    "id": "cust-010",
    "tenantId": "tenant-kroger-001",
    "name": "Jack Wilson",
    "loyaltyTier": "BASIC",
    "annualSpend": 50.00,
    "segments": ["segment-new-customer", "segment-southwest"],
    "division": "division-southwest",
    "last90DaysPurchases": [],
    "edgeCase": "geo miss — division-southwest has no active geo-restricted campaigns"
  },
  {
    "id": "cust-011",
    "tenantId": "tenant-kroger-001",
    "name": "Karen Adams",
    "loyaltyTier": "PLATINUM",
    "annualSpend": 9200.00,
    "segments": ["segment-high-value", "segment-wine-buyer", "segment-vitamin-buyer", "segment-midwest"],
    "division": "division-midwest",
    "last90DaysPurchases": [
      { "upc": "upc-wine-cab",      "qty": 5, "spend": 74.95, "date": "2026-05-20" },
      { "upc": "upc-vitamin-c",     "qty": 3, "spend": 38.97, "date": "2026-05-15" },
      { "upc": "upc-multivitamin",  "qty": 2, "spend": 37.98, "date": "2026-05-10" },
      { "upc": "upc-cheerios",      "qty": 6, "spend": 29.94, "date": "2026-05-05" }
    ],
    "edgeCase": "multi-segment PLATINUM — eligible for both platinum-exclusive and general campaigns simultaneously"
  },
  {
    "id": "cust-012",
    "tenantId": "tenant-kroger-001",
    "name": "Leo Patel",
    "loyaltyTier": "GOLD",
    "annualSpend": 3800.00,
    "segments": ["segment-mid-value", "segment-midwest"],
    "division": "division-midwest",
    "last90DaysPurchases": [
      { "upc": "upc-cigarettes-marlboro", "qty": 10, "spend": 99.90, "date": "2026-05-20" },
      { "upc": "upc-milk-gallon",          "qty": 4,  "spend": 15.96, "date": "2026-05-15" }
    ],
    "edgeCase": "tobacco buyer — eligible for tobacco-only midwest promo, ineligible for general exclusion campaigns"
  }
]
```

---

## Campaigns

```json
[
  {
    "id": "camp-001",
    "tenantId": "tenant-kroger-001",
    "name": "Weekend Mega Sale",
    "status": "ACTIVE",
    "offer": { "type": "PCT_OFF", "value": 20, "upcScope": ["upc-cola-12pk", "upc-chips-family", "upc-water-24pk", "upc-diet-cola-12pk"] },
    "funding": { "vendorId": "vendor-pepsi-001", "vendorShare": 60, "krogerShare": 40 },
    "budget": { "totalAmount": 50000.00, "burnedAmount": 47600.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": false,
    "segmentRestriction": null,
    "geoScope": ["division-midwest", "division-southeast"],
    "edgeCase": "budget at 95.2% — should trigger auto-pause"
  },
  {
    "id": "camp-002",
    "tenantId": "tenant-kroger-001",
    "name": "Buy 2 Get 1 Free — Cereal",
    "status": "ACTIVE",
    "offer": { "type": "BOGO", "value": 1, "upcScope": ["upc-cheerios", "upc-granola", "upc-oatmeal-quick", "upc-frosted-flakes"] },
    "funding": { "vendorId": "vendor-generalmills-001", "vendorShare": 100, "krogerShare": 0 },
    "budget": { "totalAmount": 30000.00, "burnedAmount": 12000.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-15" },
    "stackPermission": true,
    "segmentRestriction": null,
    "geoScope": ["division-all"]
  },
  {
    "id": "camp-003",
    "tenantId": "tenant-kroger-001",
    "name": "Spend $50 Get $10 Off",
    "status": "ACTIVE",
    "offer": { "type": "THRESHOLD", "thresholdAmount": 50.00, "discountAmount": 10.00, "upcScope": [] },
    "funding": { "vendorId": "vendor-kroger-own", "vendorShare": 0, "krogerShare": 100 },
    "budget": { "totalAmount": 100000.00, "burnedAmount": 23000.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": true,
    "segmentRestriction": null,
    "geoScope": ["division-all"],
    "edgeCase": "100% Kroger funded — vendorShare is 0 — ROI must be null not divide-by-zero"
  },
  {
    "id": "camp-004",
    "tenantId": "tenant-kroger-001",
    "name": "Platinum Member Exclusive — 30% Off Wine",
    "status": "ACTIVE",
    "offer": { "type": "PCT_OFF", "value": 30, "upcScope": ["upc-wine-cab", "upc-wine-chard", "upc-wine-rose", "upc-wine-pinot", "upc-wine-sauv-blanc"] },
    "funding": { "vendorId": "vendor-kroger-own", "vendorShare": 0, "krogerShare": 100 },
    "budget": { "totalAmount": 20000.00, "burnedAmount": 5000.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": false,
    "segmentRestriction": "PLATINUM",
    "geoScope": ["division-all"],
    "edgeCase": "PLATINUM-only AND alcohol UPCs — general exclusions do NOT apply here since campaign explicitly scopes wine"
  },
  {
    "id": "camp-005",
    "tenantId": "tenant-kroger-001",
    "name": "Summer BBQ Bundle",
    "status": "ACTIVE",
    "offer": { "type": "AMT_OFF", "value": 5.00, "upcScope": ["upc-hotdogs", "upc-buns", "upc-ketchup", "upc-mustard", "upc-ground-beef-1lb"] },
    "funding": { "vendorId": "vendor-oscar-meyer-001", "vendorShare": 75, "krogerShare": 25 },
    "budget": { "totalAmount": 15000.00, "burnedAmount": 14300.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": false,
    "segmentRestriction": null,
    "geoScope": ["division-all"],
    "edgeCase": "budget at 95.3% — should already be paused"
  },
  {
    "id": "camp-006",
    "tenantId": "tenant-kroger-001",
    "name": "Pharmacy — Vitamin Bundle",
    "status": "PAUSED",
    "pauseReason": "BUDGET_EXHAUSTED",
    "offer": { "type": "PCT_OFF", "value": 15, "upcScope": ["upc-vitamin-c", "upc-vitamin-d", "upc-multivitamin"] },
    "funding": { "vendorId": "vendor-nature-made-001", "vendorShare": 50, "krogerShare": 50 },
    "budget": { "totalAmount": 10000.00, "burnedAmount": 10000.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": false,
    "segmentRestriction": null,
    "geoScope": ["division-all"],
    "edgeCase": "already PAUSED — eligibility engine must not serve offers for this campaign"
  },
  {
    "id": "camp-007",
    "tenantId": "tenant-kroger-001",
    "name": "Back to School Stationery",
    "status": "SCHEDULED",
    "offer": { "type": "PCT_OFF", "value": 25, "upcScope": [] },
    "funding": { "vendorId": "vendor-mead-001", "vendorShare": 80, "krogerShare": 20 },
    "budget": { "totalAmount": 25000.00, "burnedAmount": 0, "currency": "USD" },
    "dateRange": { "startDate": "2026-08-01", "endDate": "2026-08-31" },
    "stackPermission": false,
    "segmentRestriction": null,
    "geoScope": ["division-all"],
    "edgeCase": "SCHEDULED — must not be served, future date"
  },
  {
    "id": "camp-008",
    "tenantId": "tenant-kroger-001",
    "name": "Tobacco Promo — Midwest Only",
    "status": "ACTIVE",
    "offer": { "type": "AMT_OFF", "value": 2.00, "upcScope": ["upc-cigarettes-marlboro", "upc-cigarettes-camel"] },
    "funding": { "vendorId": "vendor-altria-001", "vendorShare": 100, "krogerShare": 0 },
    "budget": { "totalAmount": 5000.00, "burnedAmount": 1200.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": false,
    "segmentRestriction": null,
    "geoScope": ["division-midwest"],
    "edgeCase": "tobacco + geo-restricted — cust-012 (midwest, tobacco buyer) eligible; cust-002 (southeast) ineligible"
  },
  {
    "id": "camp-009",
    "tenantId": "tenant-kroger-001",
    "name": "Conflicting UPC — Cannot Publish",
    "status": "DRAFT",
    "offer": { "type": "PCT_OFF", "value": 10, "upcScope": ["upc-cola-12pk"] },
    "funding": { "vendorId": "vendor-cocacola-001", "vendorShare": 100, "krogerShare": 0 },
    "budget": { "totalAmount": 10000.00, "burnedAmount": 0, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": false,
    "geoScope": ["division-all"],
    "conflictsWith": "camp-001",
    "edgeCase": "upc-cola-12pk already in camp-001 — publish must fail with UPC_OVERLAP"
  },
  {
    "id": "camp-010",
    "tenantId": "tenant-kroger-001",
    "name": "No Funding Source — Cannot Publish",
    "status": "DRAFT",
    "offer": { "type": "PCT_OFF", "value": 5, "upcScope": ["upc-bread-white"] },
    "funding": null,
    "budget": { "totalAmount": 5000.00, "burnedAmount": 0, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "edgeCase": "no funding — publish must fail with NO_FUNDING_SOURCE"
  },
  {
    "id": "camp-011",
    "tenantId": "tenant-kroger-001",
    "name": "Stackable Breakfast Deal",
    "status": "ACTIVE",
    "offer": { "type": "AMT_OFF", "value": 2.00, "upcScope": ["upc-granola-bar-6pk", "upc-oatmeal-quick", "upc-granola"] },
    "funding": { "vendorId": "vendor-quaker-001", "vendorShare": 70, "krogerShare": 30 },
    "budget": { "totalAmount": 8000.00, "burnedAmount": 2400.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": true,
    "segmentRestriction": null,
    "geoScope": ["division-all"],
    "edgeCase": "stackable with camp-002 — cereal buyer can use both in one transaction"
  },
  {
    "id": "camp-012",
    "tenantId": "tenant-kroger-001",
    "name": "Frozen Foods Flash Sale",
    "status": "ACTIVE",
    "offer": { "type": "PCT_OFF", "value": 15, "upcScope": ["upc-frozen-pizza", "upc-frozen-fries"] },
    "funding": { "vendorId": "vendor-nestle-001", "vendorShare": 85, "krogerShare": 15 },
    "budget": { "totalAmount": 12000.00, "burnedAmount": 11400.00, "currency": "USD" },
    "dateRange": { "startDate": "2026-06-01", "endDate": "2026-06-30" },
    "stackPermission": false,
    "segmentRestriction": null,
    "geoScope": ["division-all"],
    "edgeCase": "budget at 95.0% exactly — tests the boundary condition precisely"
  }
]
```

---

## Eligibility Rules

```json
[
  {
    "campaignId": "camp-001",
    "tenantId": "tenant-kroger-001",
    "stackLimit": 1,
    "segmentRestriction": null,
    "exclusions": ["cat-alcohol", "cat-tobacco"],
    "geoScope": ["division-midwest", "division-southeast"],
    "threshold": null
  },
  {
    "campaignId": "camp-002",
    "tenantId": "tenant-kroger-001",
    "stackLimit": 2,
    "segmentRestriction": null,
    "exclusions": [],
    "geoScope": ["division-all"],
    "threshold": null
  },
  {
    "campaignId": "camp-003",
    "tenantId": "tenant-kroger-001",
    "stackLimit": 2,
    "segmentRestriction": null,
    "exclusions": ["cat-alcohol", "cat-tobacco"],
    "geoScope": ["division-all"],
    "threshold": { "type": "SPEND", "value": 50.00 }
  },
  {
    "campaignId": "camp-004",
    "tenantId": "tenant-kroger-001",
    "stackLimit": 1,
    "segmentRestriction": "PLATINUM",
    "exclusions": [],
    "geoScope": ["division-all"],
    "threshold": null,
    "note": "No exclusions — wine is explicitly in scope for this campaign"
  },
  {
    "campaignId": "camp-005",
    "tenantId": "tenant-kroger-001",
    "stackLimit": 1,
    "segmentRestriction": null,
    "exclusions": ["cat-alcohol", "cat-tobacco"],
    "geoScope": ["division-all"],
    "threshold": null
  },
  {
    "campaignId": "camp-008",
    "tenantId": "tenant-kroger-001",
    "stackLimit": 1,
    "segmentRestriction": null,
    "exclusions": [],
    "geoScope": ["division-midwest"],
    "threshold": null,
    "note": "Tobacco explicitly in scope — no alcohol/tobacco exclusion applied"
  },
  {
    "campaignId": "camp-011",
    "tenantId": "tenant-kroger-001",
    "stackLimit": 2,
    "segmentRestriction": null,
    "exclusions": [],
    "geoScope": ["division-all"],
    "threshold": null
  },
  {
    "campaignId": "camp-012",
    "tenantId": "tenant-kroger-001",
    "stackLimit": 1,
    "segmentRestriction": null,
    "exclusions": ["cat-alcohol", "cat-tobacco"],
    "geoScope": ["division-all"],
    "threshold": null
  }
]
```

---

## Redemptions

```json
[
  {
    "id": "redeem-001",
    "tenantId": "tenant-kroger-001",
    "idempotencyKey": "pos-store-chi-001-txn-88821",
    "customerId": "cust-001",
    "campaignId": "camp-001",
    "discountApplied": 6.99,
    "cartTotal": 34.95,
    "redeemedAt": "2026-06-01T10:22:00Z",
    "storeId": "store-chicago-001",
    "division": "division-midwest",
    "status": "CONFIRMED"
  },
  {
    "id": "redeem-002",
    "tenantId": "tenant-kroger-001",
    "idempotencyKey": "pos-store-atl-001-txn-88822",
    "customerId": "cust-002",
    "campaignId": "camp-002",
    "discountApplied": 4.99,
    "cartTotal": 22.50,
    "redeemedAt": "2026-06-01T11:05:00Z",
    "storeId": "store-atlanta-001",
    "division": "division-southeast",
    "status": "CONFIRMED"
  },
  {
    "id": "redeem-003",
    "tenantId": "tenant-kroger-001",
    "idempotencyKey": "pos-store-chi-001-txn-88821",
    "customerId": "cust-001",
    "campaignId": "camp-001",
    "discountApplied": 6.99,
    "cartTotal": 34.95,
    "redeemedAt": "2026-06-01T10:22:05Z",
    "storeId": "store-chicago-001",
    "division": "division-midwest",
    "edgeCase": "DUPLICATE — same idempotencyKey as redeem-001 — must return 409"
  },
  {
    "id": "redeem-004",
    "tenantId": "tenant-kroger-001",
    "idempotencyKey": "pos-store-col-001-txn-11201",
    "customerId": "cust-005",
    "campaignId": "camp-003",
    "discountApplied": 10.00,
    "cartTotal": 67.50,
    "redeemedAt": "2026-06-02T09:15:00Z",
    "storeId": "store-columbus-001",
    "division": "division-midwest",
    "status": "CONFIRMED"
  },
  {
    "id": "redeem-005",
    "tenantId": "tenant-kroger-001",
    "idempotencyKey": "pos-store-chi-001-txn-88821",
    "customerId": "cust-001",
    "campaignId": "camp-001",
    "discountApplied": 6.99,
    "cartTotal": 34.95,
    "redeemedAt": "2026-06-01T10:22:05Z",
    "storeId": "store-chicago-001",
    "division": "division-midwest",
    "edgeCase": "SAME key DIFFERENT tenant — tenantId: tenant-other-001 — must SUCCEED, idempotency is per-tenant"
  },
  {
    "id": "redeem-006",
    "tenantId": "tenant-kroger-001",
    "idempotencyKey": "pos-store-ind-001-txn-55001",
    "customerId": "cust-011",
    "campaignId": "camp-002",
    "discountApplied": 4.99,
    "cartTotal": 45.20,
    "redeemedAt": "2026-06-03T14:00:00Z",
    "storeId": "store-indy-001",
    "division": "division-midwest",
    "status": "CONFIRMED",
    "edgeCase": "multi-segment PLATINUM customer stacking camp-002 (stackPermission:true) and camp-011"
  },
  {
    "id": "redeem-007",
    "tenantId": "tenant-kroger-001",
    "idempotencyKey": "pos-store-det-001-txn-22201",
    "customerId": "cust-012",
    "campaignId": "camp-008",
    "discountApplied": 2.00,
    "cartTotal": 19.99,
    "redeemedAt": "2026-06-03T15:30:00Z",
    "storeId": "store-detroit-001",
    "division": "division-midwest",
    "status": "CONFIRMED",
    "edgeCase": "tobacco promo redemption — valid because camp-008 explicitly includes tobacco"
  },
  {
    "id": "redeem-008",
    "tenantId": "tenant-kroger-001",
    "idempotencyKey": "pos-store-chi-001-txn-SESSION",
    "customerId": "cust-001",
    "campaignId": "camp-002",
    "discountApplied": 4.99,
    "cartTotal": 28.50,
    "redeemedAt": "SESSION_START_TIME",
    "storeId": "store-chicago-001",
    "division": "division-midwest",
    "status": "CONFIRMED",
    "edgeCase": "Scenario 28 — T+24 claim timing. Use SESSION_START_TIME as redeemedAt when seeding during Sprint 2. Claim must be generated at SESSION_START_TIME + 24hrs. Do not use a historical timestamp — the claim scheduling job tests real timing."
  }
]
```

---

## Analytics Benchmarks

```json
[
  {
    "campaignId": "camp-001",
    "tenantId": "tenant-kroger-001",
    "baselineSalesPerDay": 1200.00,
    "actualSalesPerDay": 1980.00,
    "lift": 780.00,
    "liftPercentage": 65.0,
    "totalFundingCost": 47600.00,
    "incrementalMargin": 78000.00,
    "roi": 1.64,
    "budgetBurnPercent": 95.2,
    "redemptionCount": 6800,
    "edgeCase": "burn at 95.2% — BudgetExhausted must have fired"
  },
  {
    "campaignId": "camp-002",
    "tenantId": "tenant-kroger-001",
    "baselineSalesPerDay": 800.00,
    "actualSalesPerDay": 1100.00,
    "lift": 300.00,
    "liftPercentage": 37.5,
    "totalFundingCost": 12000.00,
    "incrementalMargin": 30000.00,
    "roi": 2.50,
    "budgetBurnPercent": 40.0,
    "redemptionCount": 2400
  },
  {
    "campaignId": "camp-003",
    "tenantId": "tenant-kroger-001",
    "baselineSalesPerDay": 5000.00,
    "actualSalesPerDay": 6200.00,
    "lift": 1200.00,
    "liftPercentage": 24.0,
    "totalFundingCost": 0.00,
    "incrementalMargin": 120000.00,
    "roi": null,
    "budgetBurnPercent": 23.0,
    "redemptionCount": 2300,
    "edgeCase": "roi is null — 100% Kroger funded, vendorShare=0, divide-by-zero guard required"
  },
  {
    "campaignId": "camp-004",
    "tenantId": "tenant-kroger-001",
    "baselineSalesPerDay": 400.00,
    "actualSalesPerDay": 380.00,
    "lift": -20.00,
    "liftPercentage": -5.0,
    "totalFundingCost": 5000.00,
    "incrementalMargin": -2000.00,
    "roi": -0.40,
    "budgetBurnPercent": 25.0,
    "redemptionCount": 333,
    "edgeCase": "negative lift — promo underperformed baseline, ROI is negative"
  },
  {
    "campaignId": "camp-005",
    "tenantId": "tenant-kroger-001",
    "baselineSalesPerDay": 500.00,
    "actualSalesPerDay": 820.00,
    "lift": 320.00,
    "liftPercentage": 64.0,
    "totalFundingCost": 14300.00,
    "incrementalMargin": 32000.00,
    "roi": 2.24,
    "budgetBurnPercent": 95.3,
    "redemptionCount": 2860,
    "edgeCase": "burn at 95.3% — BudgetExhausted must have fired"
  },
  {
    "campaignId": "camp-006",
    "tenantId": "tenant-kroger-001",
    "baselineSalesPerDay": 300.00,
    "actualSalesPerDay": 480.00,
    "lift": 180.00,
    "liftPercentage": 60.0,
    "totalFundingCost": 10000.00,
    "incrementalMargin": 18000.00,
    "roi": 1.80,
    "budgetBurnPercent": 100.0,
    "redemptionCount": 2000,
    "edgeCase": "burn at 100% — campaign paused, BudgetExhausted fired, no further redemptions possible"
  },
  {
    "campaignId": "camp-012",
    "tenantId": "tenant-kroger-001",
    "baselineSalesPerDay": 600.00,
    "actualSalesPerDay": 600.00,
    "lift": 0.00,
    "liftPercentage": 0.0,
    "totalFundingCost": 11400.00,
    "incrementalMargin": 0.00,
    "roi": 0.00,
    "budgetBurnPercent": 95.0,
    "redemptionCount": 1900,
    "edgeCase": "zero lift — promo had no incremental effect; burn at exactly 95.0% boundary"
  }
]
```

---

## Regions

```json
[
  { "id": "region-central", "name": "Central", "divisionIds": ["division-midwest", "division-southwest"] },
  { "id": "region-east",    "name": "East",    "divisionIds": ["division-southeast", "division-northeast"] },
  { "id": "region-west",    "name": "West",    "divisionIds": ["division-west"] }
]
```

---

## Stores

```json
[
  {
    "id": "store-chicago-001",
    "name": "Kroger Chicago Lincoln Park",
    "divisionId": "division-midwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-chicago-001",
    "address": { "street": "2200 N Halsted St", "city": "Chicago", "state": "IL", "zip": "60614" }
  },
  {
    "id": "store-chicago-002",
    "name": "Kroger Chicago Wicker Park",
    "divisionId": "division-midwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-chicago-002",
    "address": { "street": "1600 N Milwaukee Ave", "city": "Chicago", "state": "IL", "zip": "60647" }
  },
  {
    "id": "store-columbus-001",
    "name": "Kroger Columbus Short North",
    "divisionId": "division-midwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-columbus-001",
    "address": { "street": "875 N High St", "city": "Columbus", "state": "OH", "zip": "43215" }
  },
  {
    "id": "store-columbus-002",
    "name": "Kroger Columbus Easton",
    "divisionId": "division-midwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-columbus-002",
    "address": { "street": "3891 Easton Square Pl", "city": "Columbus", "state": "OH", "zip": "43219" }
  },
  {
    "id": "store-indy-001",
    "name": "Kroger Indianapolis Meridian",
    "divisionId": "division-midwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-indy-001",
    "address": { "street": "4015 N Meridian St", "city": "Indianapolis", "state": "IN", "zip": "46208" }
  },
  {
    "id": "store-detroit-001",
    "name": "Kroger Detroit Midtown",
    "divisionId": "division-midwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-detroit-001",
    "address": { "street": "4960 Woodward Ave", "city": "Detroit", "state": "MI", "zip": "48202" }
  },
  {
    "id": "store-atlanta-001",
    "name": "Kroger Atlanta Buckhead",
    "divisionId": "division-southeast",
    "regionId": "region-east",
    "storeManagerId": "mgr-atlanta-001",
    "address": { "street": "3330 Piedmont Rd NE", "city": "Atlanta", "state": "GA", "zip": "30305" }
  },
  {
    "id": "store-atlanta-002",
    "name": "Kroger Atlanta Midtown",
    "divisionId": "division-southeast",
    "regionId": "region-east",
    "storeManagerId": "mgr-atlanta-002",
    "address": { "street": "725 Ponce De Leon Ave NE", "city": "Atlanta", "state": "GA", "zip": "30306" }
  },
  {
    "id": "store-charlotte-001",
    "name": "Kroger Charlotte South End",
    "divisionId": "division-southeast",
    "regionId": "region-east",
    "storeManagerId": "mgr-charlotte-001",
    "address": { "street": "1601 Camden Rd", "city": "Charlotte", "state": "NC", "zip": "28203" }
  },
  {
    "id": "store-nyc-001",
    "name": "Kroger New York Upper East Side",
    "divisionId": "division-northeast",
    "regionId": "region-east",
    "storeManagerId": "mgr-nyc-001",
    "address": { "street": "1498 Third Ave", "city": "New York", "state": "NY", "zip": "10075" }
  },
  {
    "id": "store-boston-001",
    "name": "Kroger Boston Back Bay",
    "divisionId": "division-northeast",
    "regionId": "region-east",
    "storeManagerId": "mgr-boston-001",
    "address": { "street": "15 Westland Ave", "city": "Boston", "state": "MA", "zip": "02115" }
  },
  {
    "id": "store-boston-002",
    "name": "Kroger Boston Fenway",
    "divisionId": "division-northeast",
    "regionId": "region-east",
    "storeManagerId": "mgr-boston-002",
    "address": { "street": "1380 Boylston St", "city": "Boston", "state": "MA", "zip": "02215" }
  },
  {
    "id": "store-la-001",
    "name": "Kroger Los Angeles Silver Lake",
    "divisionId": "division-west",
    "regionId": "region-west",
    "storeManagerId": "mgr-la-001",
    "address": { "street": "2716 Hyperion Ave", "city": "Los Angeles", "state": "CA", "zip": "90027" }
  },
  {
    "id": "store-la-002",
    "name": "Kroger Los Angeles Westwood",
    "divisionId": "division-west",
    "regionId": "region-west",
    "storeManagerId": "mgr-la-002",
    "address": { "street": "10861 Weyburn Ave", "city": "Los Angeles", "state": "CA", "zip": "90024" }
  },
  {
    "id": "store-seattle-001",
    "name": "Kroger Seattle Capitol Hill",
    "divisionId": "division-west",
    "regionId": "region-west",
    "storeManagerId": "mgr-seattle-001",
    "address": { "street": "1401 Broadway", "city": "Seattle", "state": "WA", "zip": "98122" }
  },
  {
    "id": "store-phoenix-001",
    "name": "Kroger Phoenix Scottsdale",
    "divisionId": "division-southwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-phoenix-001",
    "address": { "street": "7119 E Shea Blvd", "city": "Scottsdale", "state": "AZ", "zip": "85254" }
  },
  {
    "id": "store-dallas-001",
    "name": "Kroger Dallas Uptown",
    "divisionId": "division-southwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-dallas-001",
    "address": { "street": "4142 Cedar Springs Rd", "city": "Dallas", "state": "TX", "zip": "75219" }
  },
  {
    "id": "store-dallas-002",
    "name": "Kroger Dallas Oak Lawn",
    "divisionId": "division-southwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-dallas-002",
    "address": { "street": "3751 Lemmon Ave", "city": "Dallas", "state": "TX", "zip": "75219" }
  },
  {
    "id": "store-denver-001",
    "name": "Kroger Denver Capitol Hill",
    "divisionId": "division-southwest",
    "regionId": "region-central",
    "storeManagerId": "mgr-denver-001",
    "address": { "street": "900 E 11th Ave", "city": "Denver", "state": "CO", "zip": "80218" }
  }
]
```

---

## Store Managers

```json
[
  {
    "id": "mgr-chicago-001",
    "storeId": "store-chicago-001",
    "name": "Patricia Okonkwo",
    "email": "p.okonkwo@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-chicago-002",
    "storeId": "store-chicago-002",
    "name": "Marcus Webb",
    "email": "m.webb@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-columbus-001",
    "storeId": "store-columbus-001",
    "name": "Sandra Reyes",
    "email": "s.reyes@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-columbus-002",
    "storeId": "store-columbus-002",
    "name": "Derek Huang",
    "email": "d.huang@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-indy-001",
    "storeId": "store-indy-001",
    "name": "Tonya Bridges",
    "email": "t.bridges@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-detroit-001",
    "storeId": "store-detroit-001",
    "name": "James Kowalski",
    "email": "j.kowalski@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-atlanta-001",
    "storeId": "store-atlanta-001",
    "name": "Denise Ware",
    "email": "d.ware@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-atlanta-002",
    "storeId": "store-atlanta-002",
    "name": "Calvin Frost",
    "email": "c.frost@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-charlotte-001",
    "storeId": "store-charlotte-001",
    "name": "Rhonda Simmons",
    "email": "r.simmons@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-nyc-001",
    "storeId": "store-nyc-001",
    "name": "Anthony Rossi",
    "email": "a.rossi@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-boston-001",
    "storeId": "store-boston-001",
    "name": "Claire Dunlap",
    "email": "c.dunlap@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-boston-002",
    "storeId": "store-boston-002",
    "name": "Terrence Flynn",
    "email": "t.flynn@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-la-001",
    "storeId": "store-la-001",
    "name": "Sofia Vega",
    "email": "s.vega@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-la-002",
    "storeId": "store-la-002",
    "name": "Brian Nakamura",
    "email": "b.nakamura@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-seattle-001",
    "storeId": "store-seattle-001",
    "name": "Megan Hollis",
    "email": "m.hollis@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-phoenix-001",
    "storeId": "store-phoenix-001",
    "name": "Gilbert Moreno",
    "email": "g.moreno@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-dallas-001",
    "storeId": "store-dallas-001",
    "name": "Latasha Greer",
    "email": "l.greer@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-dallas-002",
    "storeId": "store-dallas-002",
    "name": "Paul Whitfield",
    "email": "p.whitfield@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  },
  {
    "id": "mgr-denver-001",
    "storeId": "store-denver-001",
    "name": "Angela Tran",
    "email": "a.tran@kroger.com",
    "notificationPreferences": { "channels": ["EMAIL", "WEBSOCKET"], "events": ["CampaignPaused"] }
  }
]
```

---

## Vendor Profiles

```json
[
  {
    "vendorId": "vendor-pepsi-001",
    "name": "PepsiCo",
    "contactEmail": "promos@pepsico.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-pepsi-001"
  },
  {
    "vendorId": "vendor-generalmills-001",
    "name": "General Mills",
    "contactEmail": "trade@generalmills.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-genmills-001"
  },
  {
    "vendorId": "vendor-kroger-own",
    "name": "Kroger Own Brands",
    "contactEmail": "ownbrands@kroger.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-kroger-internal"
  },
  {
    "vendorId": "vendor-oscar-meyer-001",
    "name": "Oscar Mayer (Kraft Heinz)",
    "contactEmail": "promotions@kraftheinz.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-kraftheinz-001"
  },
  {
    "vendorId": "vendor-nature-made-001",
    "name": "Nature Made (Pharmavite)",
    "contactEmail": "retail@pharmavite.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-pharmavite-001"
  },
  {
    "vendorId": "vendor-mead-001",
    "name": "Mead (ACCO Brands)",
    "contactEmail": "kroger@accobrands.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-acco-001"
  },
  {
    "vendorId": "vendor-altria-001",
    "name": "Altria Group",
    "contactEmail": "trade@altria.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-altria-001"
  },
  {
    "vendorId": "vendor-cocacola-001",
    "name": "The Coca-Cola Company",
    "contactEmail": "kroger-promos@coca-cola.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-cocacola-001"
  },
  {
    "vendorId": "vendor-quaker-001",
    "name": "Quaker Oats (PepsiCo)",
    "contactEmail": "quaker-trade@pepsico.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-quaker-001"
  },
  {
    "vendorId": "vendor-nestle-001",
    "name": "Nestle USA",
    "contactEmail": "retail-promos@nestle.com",
    "status": "ACTIVE",
    "bankAccountId": "bank-nestle-001"
  }
]
```

---

## Campaign — camp-013 (Approval Workflow Trigger)

```json
{
  "id": "camp-013",
  "tenantId": "tenant-kroger-001",
  "name": "Premium Beverage Mega Push",
  "status": "DRAFT",
  "offer": {
    "type": "PCT_OFF",
    "value": 25,
    "upcScope": ["upc-cola-12pk", "upc-diet-cola-12pk", "upc-sparkling-12pk"]
  },
  "funding": {
    "vendorId": "vendor-pepsi-001",
    "vendorShare": 80,
    "krogerShare": 20
  },
  "budget": {
    "totalAmount": 75000.00,
    "burnedAmount": 0,
    "currency": "USD"
  },
  "dateRange": { "startDate": "2026-07-01", "endDate": "2026-07-31" },
  "stackPermission": false,
  "segmentRestriction": null,
  "geoScope": ["division-all"],
  "edgeCase": "totalAmount $75,000 > $50,000 approval threshold — requires FundingApproval before publish. PENDING_APPROVAL status expected on PUT /publish attempt."
}
```

---

## Notification Preferences (by Role)

```json
[
  {
    "role": "MX_TEAM",
    "channels": ["EMAIL", "WEBSOCKET"],
    "subscribedEvents": [
      "CampaignPaused",
      "BudgetExhausted",
      "FundingApproved",
      "FundingRejected"
    ],
    "note": "MX team members receive both email and real-time WebSocket alerts for all operational campaign events."
  },
  {
    "role": "STORE_MANAGER",
    "channels": ["EMAIL", "WEBSOCKET"],
    "subscribedEvents": [
      "CampaignPaused"
    ],
    "geoFilter": "divisionId — only campaigns whose geoScope includes the manager's divisionId",
    "note": "Store managers are notified only for CampaignPaused events affecting campaigns active in their division."
  },
  {
    "role": "VENDOR_CONTACT",
    "channels": ["EMAIL"],
    "subscribedEvents": [
      "ClaimSubmitted",
      "FundingProposalSubmitted",
      "FundingApproved"
    ],
    "note": "Vendor contacts receive email-only notifications for claim activity and funding lifecycle events tied to their vendorId."
  }
]
```

---

## Forecasting Benchmarks

Lookup key: `category × promoType × season`. `liftFactor` is a multiplier applied to baseline daily sales to derive predicted daily sales during the promotion.

```json
[
  { "category": "beverages",   "promoType": "PCT_OFF",   "season": "summer",  "liftFactor": 1.65 },
  { "category": "beverages",   "promoType": "PCT_OFF",   "season": "winter",  "liftFactor": 1.30 },
  { "category": "beverages",   "promoType": "PCT_OFF",   "season": "holiday", "liftFactor": 1.45 },
  { "category": "beverages",   "promoType": "PCT_OFF",   "season": "spring",  "liftFactor": 1.40 },
  { "category": "beverages",   "promoType": "BOGO",      "season": "summer",  "liftFactor": 1.80 },
  { "category": "beverages",   "promoType": "BOGO",      "season": "winter",  "liftFactor": 1.35 },
  { "category": "beverages",   "promoType": "AMT_OFF",   "season": "summer",  "liftFactor": 1.50 },
  { "category": "beverages",   "promoType": "THRESHOLD", "season": "summer",  "liftFactor": 1.25 },
  { "category": "breakfast",   "promoType": "PCT_OFF",   "season": "summer",  "liftFactor": 1.20 },
  { "category": "breakfast",   "promoType": "PCT_OFF",   "season": "winter",  "liftFactor": 1.55 },
  { "category": "breakfast",   "promoType": "PCT_OFF",   "season": "holiday", "liftFactor": 1.60 },
  { "category": "breakfast",   "promoType": "PCT_OFF",   "season": "spring",  "liftFactor": 1.25 },
  { "category": "breakfast",   "promoType": "BOGO",      "season": "summer",  "liftFactor": 1.30 },
  { "category": "breakfast",   "promoType": "BOGO",      "season": "winter",  "liftFactor": 1.70 },
  { "category": "breakfast",   "promoType": "AMT_OFF",   "season": "winter",  "liftFactor": 1.45 },
  { "category": "breakfast",   "promoType": "THRESHOLD", "season": "holiday", "liftFactor": 1.35 },
  { "category": "snacks",      "promoType": "PCT_OFF",   "season": "summer",  "liftFactor": 1.55 },
  { "category": "snacks",      "promoType": "PCT_OFF",   "season": "winter",  "liftFactor": 1.40 },
  { "category": "snacks",      "promoType": "PCT_OFF",   "season": "holiday", "liftFactor": 1.75 },
  { "category": "snacks",      "promoType": "PCT_OFF",   "season": "spring",  "liftFactor": 1.45 },
  { "category": "snacks",      "promoType": "BOGO",      "season": "summer",  "liftFactor": 1.70 },
  { "category": "snacks",      "promoType": "BOGO",      "season": "holiday", "liftFactor": 1.90 },
  { "category": "snacks",      "promoType": "AMT_OFF",   "season": "summer",  "liftFactor": 1.50 },
  { "category": "snacks",      "promoType": "THRESHOLD", "season": "holiday", "liftFactor": 1.40 },
  { "category": "meat",        "promoType": "PCT_OFF",   "season": "summer",  "liftFactor": 1.75 },
  { "category": "meat",        "promoType": "PCT_OFF",   "season": "winter",  "liftFactor": 1.30 },
  { "category": "meat",        "promoType": "PCT_OFF",   "season": "holiday", "liftFactor": 1.85 },
  { "category": "meat",        "promoType": "PCT_OFF",   "season": "spring",  "liftFactor": 1.50 },
  { "category": "meat",        "promoType": "BOGO",      "season": "summer",  "liftFactor": 2.00 },
  { "category": "meat",        "promoType": "AMT_OFF",   "season": "summer",  "liftFactor": 1.65 },
  { "category": "meat",        "promoType": "AMT_OFF",   "season": "holiday", "liftFactor": 1.80 },
  { "category": "meat",        "promoType": "THRESHOLD", "season": "summer",  "liftFactor": 1.35 },
  { "category": "condiments",  "promoType": "PCT_OFF",   "season": "summer",  "liftFactor": 1.60 },
  { "category": "condiments",  "promoType": "PCT_OFF",   "season": "winter",  "liftFactor": 1.20 },
  { "category": "condiments",  "promoType": "PCT_OFF",   "season": "holiday", "liftFactor": 1.35 },
  { "category": "condiments",  "promoType": "PCT_OFF",   "season": "spring",  "liftFactor": 1.45 },
  { "category": "condiments",  "promoType": "BOGO",      "season": "summer",  "liftFactor": 1.70 },
  { "category": "condiments",  "promoType": "AMT_OFF",   "season": "summer",  "liftFactor": 1.55 },
  { "category": "condiments",  "promoType": "THRESHOLD", "season": "summer",  "liftFactor": 1.25 }
]
```

---

## Approval Rules

```json
{
  "approvalThreshold": 50000.00,
  "approverRole": "MX_DIRECTOR",
  "autoApproveBelow": 25000.00,
  "currency": "USD",
  "note": "Campaigns with totalAmount >= $50,000 require explicit FundingApproved event from MX_DIRECTOR before status can transition from PENDING_APPROVAL to ACTIVE. Campaigns below $25,000 are auto-approved on publish. Campaigns between $25,000 and $49,999.99 are approved on publish without director sign-off."
}
```

---

## Validation Scenarios

All teams validate their implementation against these scenarios. Pass/fail is explicit.

**Sprint scope key:** S1 = Sprint 1, S2 = Sprint 2, S3 = Sprint 3 regression, S4 = Sprint 4 demo

**Frontend note:** Frontend TL wires to live APIs from Sprint 1. Campaign Service and Analytics Service are pre-built and live from day one. Eligibility Service is live from end of Sprint 1. All Frontend scenarios validate against live services — no mock data.

| # | Sprint | Scenario | Customer | Campaign | Input | Expected Output | Validates |
|---|--------|----------|----------|----------|-------|----------------|-----------|
| 1 | S2 | Happy path — basic eligibility | cust-002 | camp-002 | cartTotal: 22.50 | eligible: true, discount: 4.99 | Eligibility basic flow |
| 2 | S1 | Segment mismatch — GOLD rejected from PLATINUM | cust-002 | camp-004 | — | eligible: false, SEGMENT_MISMATCH | Segment restriction |
| 3 | S1 | PLATINUM qualifies for PLATINUM campaign | cust-001 | camp-004 | — | eligible: true, discount: 30% | Segment + loyalty tier |
| 4 | S2 | Alcohol excluded from general campaign | cust-001 | camp-001 | cart includes upc-wine-cab | eligible: false, UPC_EXCLUDED | Exclusion check |
| 5 | S2 | Threshold not met | cust-003 | camp-003 | cartTotal: 35.00 | eligible: false, THRESHOLD_NOT_MET | Threshold evaluation |
| 6 | S2 | Threshold met | cust-003 | camp-003 | cartTotal: 67.50 | eligible: true, discount: 10.00 | Threshold evaluation |
| 7 | S1 | Duplicate redemption blocked | cust-001 | camp-001 | idempotencyKey: pos-store-chi-001-txn-88821 | 409 DUPLICATE_REDEMPTION | Idempotency guard |
| 8 | S2 | Budget exhaustion triggers pause | — | camp-001 | full event chain live | CampaignPaused event emitted | Budget exhaustion |
| 9 | S1 | Campaign publish fails — no funding | — | camp-010 | PUT /publish | 400 NO_FUNDING_SOURCE | Funding validation |
| 10 | S2 | Campaign publish fails — UPC overlap | — | camp-009 | PUT /publish | 409 UPC_OVERLAP | Conflict resolution |
| 11 | S1 | PAUSED campaign not served | cust-001 | camp-006 | GET /offers | campaign not in response | Status filter |
| 12 | S1 | SCHEDULED campaign not served | cust-001 | camp-007 | GET /offers | campaign not in response | Status filter |
| 13 | S2 | Geo — customer in scope | cust-002 | camp-001 | southeast in scope | eligible: true | Geo validation |
| 14 | S2 | Geo miss — customer outside all scope | cust-010 | camp-001 | southwest not in scope | eligible: false, GEO_MISMATCH | Geo validation |
| 15 | S2 | Stack — two stackable campaigns | cust-011 | camp-002 + camp-011 | both stackLimit:2 | both eligible | Stack validation |
| 16 | S2 | Stack — limit exceeded | cust-011 | camp-002+camp-011+camp-001 | stackLimit:1 on camp-001 | camp-001 rejected, STACK_EXCEEDED | Stack validation |
| 17 | S1 | Same idempotency key different tenant | cust-001 | camp-001 | tenantId: tenant-other-001 | 200 success | Tenant isolation |
| 18 | S1 | Zero lift campaign analytics | — | camp-012 | GET /analytics/camp-012/report | lift: 0.0, roi: 0.0 | Zero lift handling |
| 19 | S1 | Negative lift campaign analytics | — | camp-004 | GET /analytics/camp-004/report | lift: -20.00, roi: -0.40 | Negative lift |
| 20 | S1 | Zero-cost campaign — no divide-by-zero | — | camp-003 | GET /analytics/camp-003/report | roi: null, 200 response | Zero-guard on ROI |
| 21 | S1 | Tier boundary — exactly PLATINUM | cust-007 | camp-004 | annualSpend: 5000.00 | PLATINUM tier, eligible | Tier boundary |
| 22 | S1 | Tier boundary — $1 below PLATINUM | cust-006 | camp-004 | annualSpend: 4999.00 | GOLD tier, ineligible | Tier boundary |
| 23 | S1 | Exclusion inheritance | — | — | GET /catalog/upc/upc-wine-chard | excluded: true, inheritedFrom: cat-alcohol | Hierarchy resolver |
| 24 | S2 | Tobacco promo — geo + product in scope | cust-012 | camp-008 | midwest tobacco buyer | eligible: true | Geo + product scope |
| 25 | S2 | Tobacco promo — geo miss | cust-002 | camp-008 | southeast division | eligible: false, GEO_MISMATCH | Geo validation |
| 26 | S1 | Budget at exactly 95.0% boundary | — | camp-012 | burnedAmount: 11400.00 (95.0%) | CampaignPaused event emitted | Budget boundary (isolated) |
| 27 | S2 | BudgetExhausted published exactly once | — | camp-001 | subsequent redemptions after pause | BudgetExhausted NOT re-emitted | Idempotent event |
| 28 | S2 | Claim generated at T+24hrs | — | camp-002 | redeem-008 at SESSION_START_TIME | claim at SESSION_START_TIME+24hrs | Claim lifecycle |
| 29 | S2 | Claim deduction = vendorShare % | — | camp-001 | discount: 6.99, vendorShare: 60% | deduction: 4.19 | Claim calculation |
| 30 | S1 | Frontend: campaign list shows live data | — | camp-001,002,006,007 | GET /campaigns live | PAUSED/ACTIVE/SCHEDULED badges correct | Frontend live wire |
| 31 | S2 | Multi-segment sees all eligible offers | cust-011 | camp-002, camp-003, camp-004 | GET /offers | all three returned, ranked | Multi-segment |
| 32 | S1 | camp-013 publish blocked — needs approval | — | camp-013 | PUT /publish | 423 PENDING_APPROVAL | Approval threshold |
| 33 | S2 | FundingApproved unblocks camp-013 publish | — | camp-013 | FundingApproved event received | campaign transitions to ACTIVE | Vendor approval flow |
| 34 | S2 | FundingRejected keeps camp-013 in DRAFT | — | camp-013 | FundingRejected event received | campaign stays DRAFT, reason recorded | Vendor rejection flow |
| 35 | S1 | Notification sent on CampaignPaused | — | camp-001 | CampaignPaused event | MX_TEAM email + WebSocket delivered | Notification delivery |
| 36 | S2 | WebSocket delivers alert in real-time | Frontend client | camp-001 | BudgetExhausted event | alert fires < 1 second, no polling | WebSocket real-time |
| 37 | S2 | Store manager notified for their division only | store-chicago-001 manager | camp-001 (midwest) | CampaignPaused | chicago/columbus/indy/detroit managers notified, NOT atlanta | Store geo routing |
| 38 | S2 | Vendor notified on ClaimSubmitted | vendor-pepsi-001 | redeem-001 claim | ClaimSubmitted event | vendor-pepsi-001 contact receives email | Vendor notification |
| 39 | S3 | Predicted lift returned for scheduled campaign | — | camp-013 | GET /vendor/forecasts?campaignId=camp-013 | predictedLift based on beverage × PCT_OFF × summer benchmark | Forecasting |
| 40 | S3 | Burn velocity calculated for active campaign | — | camp-001 | GET /analytics/camp-001/burn-velocity | burnVelocity = burnedAmount / daysSinceStart | Burn velocity |
