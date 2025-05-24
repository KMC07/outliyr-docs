# Customization & Examples

Now that you understand the core components and advanced concepts of the Equipment System, let's explore how you can tailor it to your specific needs and look at some practical examples.

### Adding Custom Visuals & Sounds

One of the most common customizations is adding specific visual or auditory feedback when equipment changes state.

**Method:** Subclass `ULyraEquipmentInstance` in Blueprint and implement the `K2_` lifecycle events.

**Example: Weapon Equip Animation & Sound**

1. **Create Blueprint:** Create a new Blueprint class, `BP_WeaponInstance`, inheriting from `ULyraWeaponInstance` (or `ULyraEquipmentInstance` if it's not specifically a weapon).
2. **Implement Events:**
   * Open `BP_WeaponInstance`.
   * In the "My Blueprint" panel, under "Functions > Override", find and implement `K2_OnEquipped`.
   * Inside `K2_OnEquipped`:
     * Get the Pawn (`Get Pawn`).
     * Get the Pawn's Mesh component (e.g., `Get Mesh`).
     * Play an Animation Montage on the Mesh (e.g., `Play Anim Montage` node with your weapon equip montage).
     * Spawn a Sound Attached (`Spawn Sound Attached` node) to the Pawn's Mesh for the equip sound effect.
   * Implement `K2_OnUnequipped` similarly to play an unequip animation/sound or potentially stop looping sounds started in `K2_OnEquipped`.
   * Implement `K2_OnHolster` / `K2_OnUnHolster` if you need specific animations/sounds for holstering/unholstering distinct from equipping/unequipping.
3. **Configure Definition:** Open the corresponding `ULyraEquipmentDefinition` (e.g., `ED_Rifle`). Set its `Instance Type` property to your new `BP_WeaponInstance`.

**Other Visual Ideas:**

* Enable/disable particle effects (muzzle flash components, cosmetic effects) on spawned actors during `K2_OnEquipped`/`K2_OnUnequipped`.
* Drive Material Parameters on the character or equipment meshes based on state changes.
* Attach/detach specific cosmetic actors (like a scope cover) during `K2_OnHolster`/`K2_OnUnHolster`.

### Modifying Behavior with Custom Instances

Sometimes you need more than just visuals; you need custom logic within the instance itself.

**Method:** Subclass `ULyraEquipmentInstance` in C++ or Blueprint and add custom variables and functions.

**Example: Tracking Weapon Heat in C++**

(Similar to the `ULyraRangedWeaponInstance` example provided earlier)

1. **Create C++ Class:** Create `MyRangedWeaponInstance` inheriting from `ULyraWeaponInstance`.
2. **Add Variables:** Add properties like `CurrentHeat`, `HeatDissipationRate`, etc. Make them `UPROPERTY()` and potentially `Replicated` if clients need the exact value.
3. **Override Tick:** Implement the `Tick` function to handle heat dissipation over time.
4. **Add Functions:** Create functions like `AddHeat(float Amount)` to be called by abilities (like the firing ability).
5. **Update Definition:** Set the `Instance Type` in the `ULyraEquipmentDefinition` to `MyRangedWeaponInstance`.
6. **Access from Ability:** In your firing ability (derived from `ULyraGameplayAbility_FromEquipment`), get the instance via `GetAssociatedEquipment()`, cast it to `MyRangedWeaponInstance*`, and call `AddHeat()`.

### Reacting to Equipment Changes (UI & Other Systems)

External systems often need to know when the player's equipment changes.

**Method:** Listen for the Gameplay Messages broadcast by the system using the `UGameplayMessageSubsystem`.

<details>

<summary>Updating HUD Ammo Counter</summary>

* **Identify Messages:** The HUD might need to know:
  * When the held weapon changes (`TAG_Lyra_Equipment_Message_EquipmentChanged`, potentially filtered by checking if `bIsHeld` is true and the instance is a weapon).
  * When the ammo count _for the currently held weapon_ changes (This often comes from the _Inventory System_, perhaps via a message broadcast when an item's `StatTags` change, or by directly querying the `ULyraInventoryItemInstance` linked via the held `ULyraEquipmentInstance`).

- **Create Listener:** In your HUD Widget Blueprint or C++ code:
  * Get the `UGameplayMessageSubsystem`.
  * Register a listener for the relevant message tags (e.g., `TAG_Lyra_Equipment_Message_EquipmentChanged`). You'll likely need to filter messages to only react to those relevant to the locally controlled player's Pawn/Controller.

* **Implement Handler:** Create a function to handle the received message payload.
  * When `FLyraEquipmentChangeMessage` is received:
    * Check if it's for the local player's Pawn.
    * Check if `bIsHeld` is true and `EquipmentInstance` is valid and perhaps cast it to `ULyraWeaponInstance`.
    * If a new weapon is held, get its associated `ULyraInventoryItemInstance` (`EquipmentInstance->GetInstigator()`).
    * Query the `ItemInstance` for the current ammo (`GetStatTagStackCount(TAG_Inventory_Ammo_Current)`) and update the HUD text.
    * Store a reference to the current `ItemInstance` to listen for its specific ammo changes later if needed.
  * If listening for inventory-specific messages regarding ammo counts, update the HUD accordingly, but only if the changing item matches the currently stored reference for the held weapon.

</details>

<details>

<summary><strong>Updating Visual Equipment Slots (Paper Doll UI)</strong></summary>

* **Equipment Layout Widget:** This is your main UI container (e.g., a "Paper Doll" or "Character Screen" widget) where individual equipment slot widgets are placed and arranged according to your desired design.
* **Individual Equipment Slot Widgets:**
  * Each visual equipment slot (e.g., for Head, Chest, Primary Weapon) should ideally be its own dedicated widget (`W_EquipmentSlot` or similar).
  * **Association:** Within each slot widget, you associate it with a specific `GameplayTag` that it represents (e.g., `Lyra.Equipment.Slot.Armor.Head`). This tag is typically configurable in the widget's properties.
* **Listening for Changes:**
  * Each individual equipment slot widget registers to listen for the `FLyraEquipmentChangeMessage` (via the `UGameplayMessageSubsystem`).
* **Updating Individual Slot UI:**
  * When an equipment slot widget receives an `FLyraEquipmentChangeMessage`:
    1. It first checks if the message is relevant to the local player and if the `SlotTag` in the message matches the `GameplayTag` this widget is configured to represent.
    2. If it matches, the widget uses the information from the `FLyraEquipmentChangeMessage` (like `EquipmentInstance` and `bRemoval`) to update its appearance. This could involve:
       * Displaying the item's icon if an `EquipmentInstance` is now present.
       * Showing an empty state if `bRemoval` is true or `EquipmentInstance` is null for its slot.
       * Storing the `EquipmentInstance` for potential interaction (e.g., context menus).
* **Dynamic & Modular Design:**
  * This approach ensures the UI dynamically reflects the `ULyraEquipmentManagerComponent`'s state, which itself is driven by the flexible, item-defined slots.
  * If you introduce an equippable item that uses a brand-new slot tag, you simply:
    1. Create or duplicate an equipment slot widget.
    2. Place it in your main `Equipment Layout Widget`.
    3. Configure its representative `GameplayTag` to the new slot tag.\
       The existing messaging system will ensure it updates correctly.

{% hint style="success" %}
**Example Implementation (TetrisInventory Plugin):**

* The **TetrisInventory** **Plugin** provides an example of a simple paper doll UI. Equipment slot widgets can be added to an "Equipment Layout" parent widget.
* Each slot widget's target `GameplayTag` can be set in its default properties within the layout.
* The layout widget automaticallys find its child equipment slot widgets and initialize them or pass down necessary references. This example also showcases UI changes based on equipment state.
{% endhint %}

{% hint style="danger" %}
**Important Note on UI Completeness:** If an item is equipped to a `GameplayTag` for which no corresponding equipment slot widget exists in your UI, the `ULyraEquipmentManagerComponent` will still successfully equip the item. However, players would have no visual representation or direct UI-based way to interact with that specific equipped slot.
{% endhint %}

</details>

### Creating Custom Instance Attributes (Tag Attributes)

Use Tag Attributes to manage ability-specific or modifiable parameters on the equipment instance.

**Method:** Add/Modify attributes using the `ULyraEquipmentInstance` functions, typically from within Gameplay Abilities or attachment logic.

**Example: Attachment Reducing Spread**

1. **Define Tag:** Ensure a tag like `Weapon.Stat.SpreadExponent` exists. The base weapon ability (`GA_ShootBullet`) should ideally `AddTagAttribute` for this on grant and `GetTagAttributeValue` when firing.
2. **Attachment Logic:** Assume an attachment system where applying an attachment grants a temporary Gameplay Ability or Effect to the Pawn.
3. **Modify Attribute:** Inside the `OnGranted` logic (or `ExecuteGameplayEffect`) for the attachment's effect:
   * Get the Pawn's `ULyraEquipmentManagerComponent`.
   * Get the currently held `ULyraEquipmentInstance` (assuming attachments only affect held items).
   * If it's the correct weapon type:
     * Call `EquipmentInstance->ModifyTagAttribute(TAG_Weapon_Stat_SpreadExponent, -0.2f, EFloatModOp::Add)`.
     * **Important:** Store the returned `FFloatStatModification` struct somewhere associated with this attachment effect instance.
4. **Revert Modification:** Inside the `OnRemoved` logic (or when the effect expires) for the attachment:
   * Retrieve the stored `FFloatStatModification` struct.
   * Get the `ULyraEquipmentInstance` again.
   * Call `EquipmentInstance->ReverseTagAttributeModification(StoredModInfo)`.

### Debugging Tips

* **Verify Definitions:** Double-check that your `ULyraInventoryItemDefinition` has the `InventoryFragment_EquippableItem` and that it correctly points to the intended `ULyraEquipmentDefinition`. Ensure the `ULyraEquipmentDefinition` has details configured for the slots you are trying to use and for the Held state if applicable.
* **Check Authority:** Remember that core functions like `EquipItemToSlot`, `HoldItem`, etc., are generally authority-only. Ensure they are being called on the server. Use `HasAuthority()` checks.
* **Replication Issues:**
  * Use the Gameplay Debugger (`'`) or network debugging tools to inspect the `ULyraEquipmentManagerComponent` on both server and client. Is the `EquipmentList` replicated correctly?
  * Check the logs for errors related to subobject replication. Ensure `bReplicateUsingRegisteredSubObjectList = true` on the Manager component if using UE5's newer subobject system (which is recommended).
  * Verify the `ULyraEquipmentInstance` and its `Instigator` (`ULyraInventoryItemInstance`) are valid on the client after replication.
* **GAS Issues:**
  * Use the GAS console command `showabilitysystem` to inspect the Pawn's ASC. Are the expected ability sets granted when equipment is equipped/held? Are they removed correctly?
  * Ensure abilities intended to be granted are `Instanced`.
  * Check that abilities derived from `ULyraGameplayAbility_FromEquipment` are correctly getting the Equipment Instance and Item Instance. Add log messages inside `GetAssociatedEquipment` or `GetAssociatedItem` if needed.
* **Visual/Actor Issues:**
  * Ensure the Actor Blueprints specified in the `ULyraEquipmentDefinition` exist and are valid.
  * Verify the `Attach Socket` names exist on the Pawn's skeletal mesh.
  * Use the debugger or `Print String` nodes within the `K2_` events (`K2_OnEquipped`, etc.) in your `BP_WeaponInstance` (or similar) to confirm they are being called when expected.

***

This concludes the core documentation for the Equipment System. By leveraging its data-driven nature, component-based structure, and integration with GAS and Inventory, you can build sophisticated and flexible equipment mechanics for your shooter project. Remember to refer back to these pages as you implement and customize your game's specific equipment needs.
