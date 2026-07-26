# Shop System

## Initial scope

- Dialog action `OpenShop:<shopKey>` opens one configured shop.
- `ShopLogic` caches `shopConfig` and `shopProductConfig` on the server.
- The client receives a presentation snapshot enriched from `inventoryConfig`.
- `ShopUICom` clones the disabled dummy slot under the vendor layout.
- Product selection opens the existing confirmation panel.
- Purchase, selling, stock depletion, restocking, limits, and conditions are reserved for a later iteration.

## Data ownership

- `shopConfig.csv`: shop-level settings.
- `shopProductConfig.csv`: product rows and display order.
- `inventoryConfig.csv`: item name, type, icon, stack and base sell price.
- Prices sent by the client must never become authoritative.

## Runtime flow

`DialogLogic.ExecuteAction` → `ShopLogic.OpenShopForUser` → `ShopLogic.OpenShopClient`
→ `ShopUICom.OpenShop` → runtime dummy-slot clones.
