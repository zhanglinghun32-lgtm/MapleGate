# Shop System

The current implementation displays vendor products and player inventory. Purchase,
sale, item transfer, currency transfer, vendor stock mutation, and persistence are
intentionally deferred.

## TODO: Server-Authoritative Transactions

- [ ] After the confirmation button is clicked, `ShopUICom` sends a buy or sell
  request containing identifiers and amount only.
- [ ] `ShopLogic` resolves the submitted shop, product, actor, and inventory slot
  again on the server. Client-provided price and item data are never trusted.
- [ ] Validate that the shop allows the operation, the amount is positive, stock or
  ownership is sufficient, and the player has enough currency.
- [ ] Transfer currency, player items, and vendor stock as one authoritative
  operation. A partial transfer must not be exposed as success.
- [ ] Mark PlayerData dirty only after a successful state change.
- [ ] Return a fresh vendor-product and player-inventory snapshot to the requesting
  client; the UI does not predict the result.
- [ ] Add duplicate-request protection so repeated confirmation events cannot apply
  the same transaction twice.

## TODO: Per-Player Shop Persistence

`shopConfig` and `shopProductConfig` are design templates, not the long-term runtime
authority for an initialized player shop.

1. When a SaveSlot or a particular shop has no saved shop state, seed it from Config.
2. Store the initialized state in `slotData.ShopState`, keyed by `shopKey`.
3. After initialization, opening a shop reads the user's in-memory PlayerData state.
4. Saving and loading a slot serializes and restores `ShopState`.
5. Buying and selling mutate the in-memory shop state and call
   `PlayerDataLogic:MarkDirty(userId)`.

The exact `ShopState` product fields, currency representation, restock policy, and
schema-version migration remain to be designed before implementation.
