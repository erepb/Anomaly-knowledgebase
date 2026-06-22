---
domain: flows/dialog-trade
tags: [xrgame, dialog, trade, npc, lua, inventory, money, flow]
sources: [src/xrGame/PhraseDialog.cpp, src/xrGame/PhraseScript.cpp, src/xrGame/PhraseDialogManager.cpp, src/xrGame/actor_communication.cpp, src/xrGame/trade.cpp, src/xrGame/trade2.cpp]
related: [flows/inventory.md, flows/lua-callback-lifecycle.md, flows/hud-ui-load.md, flows/relations-info.md, flows/dialog-system-lua.md, modlayer/gameplay/dialogs-info.md, flows/trade-economy.md, modlayer/gameplay/trade-economy.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: dialog & trade

The two player↔NPC interaction systems. **Dialog** = a phrase graph whose nodes carry Lua
precondition/action functors and info-portion gates; **trade** = item+money transfer between two
inventory owners with a condition/relation/script-discount price. Both are engine-side (C++); the
per-phrase Lua functions and dialog/trade-parameter data live in the base-game corpus (`unpacked/`)
— that Lua/data tier (attachment, XML graphs, script-built trees, `dialog_manager` chatter, the
`dialogs.script` functor library) is documented in `flows/dialog-system-lua.md` (F99). **Trade
caveat (F101):** the shipped player shop is the *Lua* trade UI — `ui_inventory` intercepts
`CUIGameSP::StartTrade` via the `actor_menu_inventory` functor and prices/transfers entirely in
Lua — so the `CTrade::TransferItem`/`GetItemPrice` path below is the engine tier underneath, not
the live player-shop transaction; profiles/restock/the NPC economy are in
`flows/trade-economy.md`.

**Provenance:** spine personally code-verified — `CPhraseDialog::SayPhrase`/`Load`/`AddPhrase`,
`CDialogScriptHelper::Precondition`/`Action` (2-speaker), `CActor::RunTalkDialog`/`StartTalk`/
`UpdateAvailableDialogs`, `CTrade::TransferItem`/`GetItemPrice`/`StartTradeEx`/`StartTrade`/
`StopTrade`/`CanTrade`. The UI windows (`CUITalkWnd`, `CUIActorMenu` trade mode) are confirmed homes
but their bodies are *(agent-traced — not personally re-read)*.

## Dialog sequence (verified)

1. **`CActor::RunTalkDialog`/`StartTalk`** (`actor_communication.cpp`) — entry when the player talks
   to the looked-at NPC; `CActor::UpdateAvailableDialogs` → `CPhraseDialogManager::UpdateAvailableDialogs`
   filters dialogs by precondition. The talk window (`CUITalkWnd`) shows the topic list. *(UI agent-traced.)*
2. **`CPhraseDialog::Load`** — builds the phrase graph for a dialog id from its XML (`<phrase_list>`):
   `AddPhrase` per node, wiring graph edges; `CDialogScriptHelper::Load` extracts each phrase's text,
   `precondition`/`action` functor names and `has_info`/`dont_has_info`/`give_info`/`disable_info` tags.
3. **`CPhraseDialog::SayPhrase`** (the engine of the conversation) — for a chosen phrase id:
   - toggle the speaking side (`m_bFirstIsSpeaking`), resolve the two `CGameObject` speakers.
   - run the said phrase's **`Action`** (info transfer + Lua action functors).
   - if the phrase vertex has no outgoing edges → `m_bFinished = true`; else for **each outgoing
     edge** run the next phrase's **`Precondition`** (speakers swapped: listener first) and collect the
     ones that pass into `m_PhraseVector`, sorted by goodwill (`PhraseGoodwillPred`). `R_ASSERT` if no
     phrase is available.
   - call **`ReceivePhrase`** on the listener (NPC auto-answer or player option list).
4. **`CDialogScriptHelper::Precondition`** (2-speaker) — `CheckInfo` (has/dont_has info on speaker1 as
   `CInventoryOwner`) gate first, then run each precondition Lua functor with
   `(speaker1, speaker2, dialog_id, phrase_id, next_phrase_id)`; first false short-circuits. (A
   `DIALOG_UPGRADE` build variant parses `func(a:b:c)` parameter-table args.)
5. **`CDialogScriptHelper::Action`** (2-speaker) — `TransferInfo` (apply give/disable info on speaker1)
   then run each action Lua functor. Info transfer is the quest-state side effect of a phrase.

## Trade sequence (verified)

1. **`CTrade::StartTradeEx(partner)`** → `StartTrade` — set partner, `TradeState = true`, record
   `m_dwLastTradeTime`; trader types get a `TradeCB(true)`. (Opened from the dialog "trade" button →
   `CUIGameSP::StartTalk`/trade UI → `CUIActorMenu` trade mode; UI agent-traced.)
2. **`CTrade::CanTrade`** — proximity/facing/alive validation gate.
3. **`CTrade::TransferItem(pItem, bBuying, bFree)`** — the money+item move:
   - `dwTransferMoney = GetItemPrice(...)`.
   - `on_before_sell`/`on_before_buy` hooks on the two owners (order depends on direction).
   - fire **`GE_TRADE_SELL`** (seller) then move money, fire **`GE_TRADE_BUY`** (buyer) then move
     money — item ownership transfers via the GE_TRADE events (see `flows/inventory.md` GE_OWNERSHIP
     family). `set_money(..., false)` adjusts both sides.
   - trader artefact special-case (`CAI_Trader::BuyArtefact`, sets `m_bNeedToUpdateArtefactTasks`).
   - if either side is the actor, fire the actor callback **`eTradeSellBuyItem(item, dir, money)`**.
4. **`CTrade::GetItemPrice(pItem, b_buying, b_free)`** — `b_free`→0; `base_cost` = artefact trader
   price or `pItem->Cost()`; `condition_factor = pow(condition*0.9+0.1, buy/sell_exponent)`;
   `relation_factor` from `RELATION_REGISTRY().GetAttitude`; `action_factor` interpolates the trade
   parameters' friend/enemy factors by relation (returns 0 / not-tradeable if the section isn't
   `enabled`); `result = floor(base_cost * condition_factor * action_factor * deficit_factor)`, then a
   **Lua script discount** (`trade_manager.get_buy_discount` / `get_sell_discount`, `R_ASSERT`ed to
   exist) is applied.
5. **`CTrade::StopTrade`** — `TradeState = false`, clear partner.

## Isolation ledger (the seams)

- **Dialog Lua functors:** per-phrase `precondition`/`action` functions resolved by name via
  `ai().script_engine().functor(...)` (`flows/lua-callback-lifecycle.md`); the functions themselves
  are base-game scripts. Info portions (`HasInfo`/`TransferInfo`) are the quest-state store dialogs
  read/write.
- **Trade Lua hooks:** `trade_manager.get_buy_discount`/`get_sell_discount` (price), `on_before_buy`/
  `on_before_sell` (`CInventoryOwner` virtuals), and the actor `eTradeSellBuyItem` callback.
- **Inventory/money:** item transfer rides the `GE_TRADE_SELL`/`GE_TRADE_BUY` net events; money via
  `set_money` (`flows/inventory.md` for the item-ownership mechanics).
- **Config/XML:** dialog phrase graphs from `<dialogs><files>` XML; trade pricing from
  `trade_parameters()` (`CTradeParameters` over the trader's trade section); relations from
  `RELATION_REGISTRY`.
- **UI:** `CUITalkWnd` (dialog), `CUIActorMenu` trade mode (`flows/hud-ui-load.md` for how those
  windows are built). The dialog↔trade switch is a UI transition.
- **No live gamedata overrides** for dialog/trade in this repo — the per-phrase functors and
  dialog/trade-parameter data are base-game content.

## Threading

Single-threaded on the main game thread; dialog/trade state advances in the frame/UI update. Trade
item transfer uses net events (`u_EventGen`/`u_EventSend`) for inventory sync.

## Failure & edge cases

- **`R_ASSERT` no available phrase:** `SayPhrase` hard-asserts if a non-terminal phrase has zero
  passing successors — a dialog graph whose preconditions all fail mid-conversation crashes rather
  than dead-ending.
- **Missing Lua functor:** `THROW3`/`R_ASSERT` if a named precondition/action or the trade discount
  functor can't be resolved — a typo'd functor name is fatal.
- **Not-tradeable item:** `GetItemPrice` returns 0 when the trade-parameters section isn't `enabled`
  for that item section (also `b_free`).
- **CanTrade gate:** proximity/facing/alive must pass or trade won't open.

## Deeper

- Dialog graph: `src/xrGame/PhraseDialog.cpp` (`SayPhrase`, `Load`, `AddPhrase`),
  `src/xrGame/PhraseScript.cpp` (`CDialogScriptHelper::Load`/`Precondition`/`Action`),
  `src/xrGame/PhraseDialogManager.cpp` (`UpdateAvailableDialogs`, `ReceivePhrase`)
- Dialog entry: `src/xrGame/actor_communication.cpp` (`RunTalkDialog`, `StartTalk`,
  `UpdateAvailableDialogs`), UI `src/xrGame/ui/UITalkWnd.cpp`
- Trade: `src/xrGame/trade.cpp` (`StartTrade*`, `StopTrade`), `src/xrGame/trade2.cpp`
  (`TransferItem`, `GetItemPrice`, `CanTrade`); params `CTradeParameters`
- Relations: `RELATION_REGISTRY` (relation_registry); UI `src/xrGame/ui/UIActorMenu*`

## Maintenance

- Re-grep `CPhraseDialog::SayPhrase` if dialog branching changes; `CTrade::GetItemPrice` if the price
  formula changes.
- Keep the price formula `floor(base*condition*action*deficit) → script discount` accurate, and the
  `R_ASSERT`ed `trade_manager.get_*_discount` requirement noted (it's a hard dependency).
- Promote the *(agent-traced)* UI windows (`CUITalkWnd`, `CUIActorMenu` trade) to verified if a task
  needs the exact UI wiring.
