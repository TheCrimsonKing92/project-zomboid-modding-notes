# Transferring Items
Notes on transferring items between inventories from a game client.

## Use the Platform
ISInventoryTransferAction exists. For the most part you point it at the character, item, item's container, and new (character's, usually) inventory.

## Multiplayer Hangups
In Multiplayer (I tend to test mod changes in the Project Zomboid Dedicated Server, running in no-Steam mode), you may find that item transfers "just don't work".

*Perhaps* because I created ISInventoryTransferAction objects in my action's update function (perhaps not), I found transfers on servers to become inconsistent and fail. There may be a more elegant solution in the timing of when I create the action or add it to the queue, but I found that adding state to the ISInventoryTransferAction which prevents calling its doActionAnim function multiple times also prevents the failure.

doActionAnim is important not because of the animations per se, but because it calls the Lua global createItemTransaction(InventoryItem item, InventoryContainer srcContainer, InventoryContainer destContainer). This in turn calls the static Java method ItemTransactionManager.createItemTransaction to register the item transaction and transmit packets.

### Example Fix
```lua
-- State for ItemSearcher action
self.foundItem = true;
-- Create the new action, but don't immediately add it to the queue
local transferItemAction = ISInventoryTransferAction:new(self.character, item, item:getContainer(), self.character:getInventory());
-- Decorate action with new state
transferItemAction.hasDoneActionAnim = false;
-- Store the original function
local oldDoActionAnim = transferItemAction.doActionAnim;
-- Replace it, writing the new function in non-self-invocation style
transferItemAction.doActionAnim = function(self, cont)
    -- Check the new state before calling
    if not self.hasDoneActionAnim then
        oldDoActionAnim(self, cont);
        -- Set it, prevent it from happening again
        self.hasDoneActionAnim = true;
    else
        -- Be clear to those looking
        print("This transfer action from ItemSearcher has had doActionAnim already called previously: avoiding invoking it again (item transactions will become inconsistent on the server)");
    end
end
-- Store the action to be added to the queue when appropriate
self.transferItemAction = transferItemAction;

--[[
    doActionAnim should have called createItemTransaction with self.item, self.srcContainer, self.destContainer
    this calls ItemTransactionManager.createItemTransaction(item:getID(), srcContainerContainingItemId, destContainerContainingItemId);
    In many cases, the ids for the container containing items are nil
    So long as the initial isConsistent check is passed
    -> the ITM should create a new ItemTransaction(itemId, srcContainerContainingItemId, destContainerContainingItemId)
    -> the ITM should call sendItemTransaction(GameClient.connection, (byte)0, itemId, srcContainerContainingItemId, destContainerContainingItemId);

    sendItemTransaction starts writing a ByteBuffer into a packet to the client connection
    -> PacketType.ItemTransaction.doPacket(GameClient.connection.startPacket()) (ByteBufferWriter received from startPacket and used in this call)
    -> put the state byte
    -> put the itemId
    -> put the srcContainerContainingItemId
    -> put the destContainerContainingItemId
    -> send the packet

    Theoretically at the end of sendItemTransaction, receiveOnClient should be called
    It seems to unpack the byte buffer from the packet in order of puts
    -> get the state byte
    -> get the itemId
    -> get the srcContainerContainingItemId
    -> get the destContainerContainingItemId

    Debug logs in the format of:
        <state> [ itemId : srcContainerContainingItemId => destContainerContainingItemId]

    ItemRequest state bytes:
        StateUnknown 0
        StateRejected 1
        StateAccepted 2

    Example (bad case) logs for ItemTransactionManager.receiveOnClient:
        2[1817133102: -1 => -1]
        1[1817133102: -1 => -1]
]]--
-- ItemTransactionManager.isConsistent(self.item:getID(), srcContainerId, destContainerId);
-- return itm.requests.stream()
--                    .filter(request -> itemId == request.itemID || srcContainerId == request.itemID || destContainerId == request.itemID || itemId == request.srcID || itemID == request.dstID)
--                    .noneMatch(request -> request.state == 1)
```