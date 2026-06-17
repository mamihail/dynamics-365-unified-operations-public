---
title: Prevent multiple owners in warehouse locations
description: Learn about the Prevent multiple owners feature, which prevents inventory from different owners from being stored in the same warehouse location. This article includes configuration guidance and operational considerations.
author: mamihail
ms.author: mamihail
ms.topic: article
ms.date: 06/16/2026
ms.custom:
ms.reviewer: kamaybac
ms.search.form: WHSLocationProfile, WHSWorkProcessTemplate, WHSPutAwayTemplate
---

# Prevent multiple owners in warehouse locations

[!include [banner](../includes/banner.md)]

## Overview

The **Prevent multiple owners** feature helps you enforce inventory owner segregation in your warehouse by preventing inventory owned by different legal entities or consigned from different parties from being stored in the same location. This is essential for warehouses that manage inventory on behalf of multiple owners, such as third-party logistics (3PL) operations, consignment warehouses, or warehouses supporting multiple legal entities with distinct ownership tracking requirements.

This feature is available starting in **Dynamics 365 Supply Chain Management version 10.0.49**.

### What is inventory owner tracking?

In Dynamics 365 Supply Chain Management, inventory ownership is tracked through the **Inventory Owner** field (`InventOwnerId_RU`) in the inventory dimensions. This field identifies which legal entity or external party owns a specific inventory lot, enabling proper tracking and settlement in multi-owner scenarios. The **Prevent multiple owners** feature uses this field to enforce location-level segregation.

### When to use this feature

Consider enabling the **Prevent multiple owners** feature in these scenarios:

- **Multi-owner warehouses**: You manage physical inventory owned by multiple legal entities and need to segregate their stock.
- **Third-party logistics (3PL)**: You manage consigned inventory from multiple suppliers or customers and need to prevent cross-mingling.
- **Consignment operations**: You track and manage consigned inventory separately from owned inventory and need location-level guarantees.
- **Regulatory compliance**: Your industry or operational model requires documented segregation of ownership.

### Benefits

- **Prevents ownership disputes**: Eliminates ambiguity about which owner inventory belongs to when stock is physically co-located.
- **Operational clarity**: Warehouse operators receive clear, immediate feedback when they attempt to violate the constraint.
- **Compliance support**: Simplifies audit trails and regulatory compliance by enforcing segregation rules consistently.
- **Automated constraints**: Reduces the need for manual workarounds or bypass procedures.

---

## How the feature works

### Core behavior

When you enable the **Prevent multiple owners** option on a location profile, that profile enforces a single-owner constraint:

- Only inventory with the same inventory owner can be stored in locations using that profile.
- Inventory with an empty (undefined) owner value is treated as its own distinct owner and mixes only with other inventory of the same empty state.
- The constraint is evaluated during put-away, manual location changes, and replenishment operations.

### Constraint evaluation points

The system evaluates the owner-mixing constraint at three key stages:

1. **Put-away operations**: When the system selects a location for received or internal-order inventory, it confirms that the location's current inventory (if any) belongs to the same owner.
2. **Manual location changes**: When a warehouse operator manually reassigns inventory to a new location via the mobile device, the system verifies the owner compatibility before allowing the move.
3. **Replenishment operations**: When the system triggers replenishment to refill a picking location, it respects the owner constraint by selecting inventory of the same owner.

### Error handling and operator feedback

When an operator attempts to put away inventory to a location that would violate the owner constraint:

- **Mobile device**: A dedicated error message informs the operator that the location already contains inventory owned by a different party, and suggests alternative locations.
- **Manual put-away**: If a manual location change would violate the constraint, the system rejects the change with a label-driven error before any posting occurs.
- **Automatic put-away**: If automatic location selection fails due to owner mismatches, the system triggers immediate replenishment to build new capacity for the incoming owner's inventory.

### Relationship to other mixing constraints

The **Prevent multiple owners** feature works alongside the existing mixing constraints on location profiles:

- **Allow mixed items**: Controls whether different products can coexist in a location.
- **Allow mixed batches**: Controls whether different batch lots can coexist in a location.
- **Allow mixed status**: Controls whether inventory with different statuses (e.g., available, on-order, damaged) can coexist.

All constraints are evaluated together. If any constraint is violated, the location is not considered as a put-away candidate.

---

## Configuration

### Prerequisites

Before you enable the **Prevent multiple owners** feature, ensure the following:

- Your warehouse uses Warehouse Management System (WMS) with location-directive-driven operations.
- Inventory owner tracking (`InventOwnerId_RU`) is configured in your tracking dimension groups for relevant items.
- The **Prevent multiple owners** feature flight has been enabled for your environment. (Contact your system administrator if you're unsure.)

### Enable the feature for a location profile

1. Go to **Warehouse management** > **Setup** > **Warehouse** > **Location profiles**.
2. In the list of location profiles, select the profile you want to configure. If it doesn't exist, select **New** to create one.
3. On the **Action Pane**, select **Edit**.
4. On the **General** FastTab, locate the **Prevent multiple owners** option.
5. Set **Prevent multiple owners** to **Yes** to enforce single-owner segregation for locations using this profile.

   > [!NOTE]
   > The **Prevent multiple owners** option is independent of the other mixing constraints (**Allow mixed items**, **Allow mixed batches**, **Allow mixed status**). You can combine it with any other constraint combination.

6. Select **Save**.

### Validate existing inventory before enabling

> [!IMPORTANT]
> If you're changing **Prevent multiple owners** from *No* to *Yes* for a location profile that already contains inventory, the system validates that no existing inventory violates the new constraint. If violations are detected, the system displays them and prevents the change until the conflicting inventory is removed or relocated.

To check for existing violations before enabling the constraint:

1. On the **Location profiles** page, select the profile you're configuring.
2. If you see an error dialog when you try to enable **Prevent multiple owners**, it lists locations and their current inventory owners.
3. Use the suggested corrective actions:
   - **Relocate inventory**: Move inventory with different owners to separate locations manually or via a transfer order.
   - **Reserve capacity**: Create new locations dedicated to specific owners.
   - **Schedule the change**: Enable the constraint during a maintenance window when relevant locations are empty.

### Example configuration

**Scenario**: Your 3PL warehouse stores consigned inventory from two customer partners (Partner A and Partner B). You want to prevent their stock from commingling.

**Configuration steps**:

1. Create two location profiles:
   - **3PL-PARTNER-A**: Set **Prevent multiple owners** to *Yes*
   - **3PL-PARTNER-B**: Set **Prevent multiple owners** to *Yes*

2. Assign locations in zone A to **3PL-PARTNER-A** profile and locations in zone B to **3PL-PARTNER-B** profile.

3. Configure location directives and putaway templates to route Partner A inventory to zone A and Partner B inventory to zone B.

4. When receiving Partner A inventory:
   - The system automatically assigns it to a zone A location using the **3PL-PARTNER-A** profile.
   - If a location is full, the system finds alternate locations within zone A with the same owner already present.
   - If no available capacity exists for Partner A's owner, the system either suggests a new empty location or initiates replenishment to free capacity.

---

## Operational guidance

### For warehouse managers

- **Monitor owner segregation**: Use the warehouse location master data to confirm that locations assigned to constrained profiles contain expected inventory owners.
- **Plan location allocation**: Allocate sufficient location capacity to each owner based on forecasted volumes.
- **Coordinate with 3PL partners**: If you manage multiple partners' inventory, agree on owner-assignment conventions to avoid empty-owner ambiguity.

### For warehouse operators (RF/mobile device)

**When receiving inventory:**

- Scan the item and owner code as directed by your receiving process.
- The system automatically selects a location respecting the owner constraint.
- If you see an error message that the location already has inventory from a different owner, **do not manually override it**. The constraint exists to prevent commingling. Instead:
  - Confirm with your supervisor that the inventory owner is correct.
  - Wait for the system to suggest an alternate location (usually automatic).
  - If prompted, manually select an alternate location suggested by the system.

**When manually relocating inventory (if applicable):**

- Always verify the inventory owner before reassigning to a new location.
- If the system rejects your location change due to owner mismatch, the current location is incompatible. Choose a location that already contains inventory of the same owner, or an empty location if available.

### When put-away fails

If the system cannot find an available location due to owner constraints:

1. **Automatic recovery**: The system may trigger immediate replenishment to consolidate and free capacity for the new owner.
2. **Operator guidance**: The mobile device or work queue displays a message explaining the constraint and suggesting next steps.
3. **Manual intervention**: In rare cases, a supervisor may need to manually relocate existing low-priority inventory to free capacity.

---

## Interaction with location directives

Location directives control where inventory is placed during put-away and picking. When the **Prevent multiple owners** constraint is active:

- **Strategy refinement**: The system augments location-directive logic to include owner as a grouping dimension alongside item, batch, and status.
- **Multi-step filtering**: Location selection first applies traditional directive criteria (zone, aisle, etc.), then filters by compatible owners.
- **Replenishment triggers**: If the directive is set to enable immediate replenishment, a failed put-away due to owner mismatch triggers replenishment, similar to item or batch mismatches.

### Example directive interaction

**Scenario**: A directive routes all PO receipts to a bulk location.

- **Without owner constraint**: Any items are placed in any BULK location with available capacity.
- **With owner constraint enabled**: Items are only placed in BULK locations containing the same owner. If no suitable BULK location exists, the directive may select a different location class or trigger replenishment.

---

## Troubleshooting

### Inventory owner information is not available

**Problem**: The system does not show inventory owner information on location or work displays.

**Resolution**: Confirm that the tracking dimension group for your items includes the **Inventory Owner** field. If it's missing, add it to the tracking dimension group for the items, then perform a new receipt to generate inventory with owner tracking.

### Changing **Prevent multiple owners** from No to Yes fails

**Problem**: When you try to enable **Prevent multiple owners** on a location profile, the system displays an error dialog listing conflicting inventory.

**Resolution**:
1. Relocate the conflicting inventory to other locations not using this profile.
2. Or, create separate location profiles for each owner and adjust your location assignments.
3. Once the location is empty or contains only single-owner inventory, retry enabling the constraint.

### Operators receive "owner mismatch" errors frequently

**Problem**: Warehouse operators report frequent constraint-violation errors during put-away.

**Possible causes**:
- **Insufficient segregated capacity**: You have allocated too few locations to each owner. Expand the location pool or reduce expected volumes.
- **Incorrect owner assignment**: Incoming inventory is being tagged with the wrong owner code. Review the receiving process and incoming document data.
- **Directive misconfiguration**: Location directives may not be routing inventory to the correct owner-dedicated locations. Review directive sequencing and query criteria.

**Resolution**:
- Work with your warehouse planner to rebalance capacity allocation per owner.
- Audit incoming data quality and owner-assignment rules.
- Review location directives to ensure they route inventory by owner correctly.

---

## Related features and topics

- **Location profiles**: For general information on setting up location profiles and constraints, see [Create location profiles](create-location-profile.md).
- **Location product dimension mixing**: To learn about mixing constraints for product dimensions, see [Location product dimension mixing](location-product-dimension-mixing.md).
- **Location directives**: For guidance on setting up location directives that respect owner constraints, see [Create location directives](create-location-directive.md).
- **Warehouse management overview**: For a broader introduction to warehouse management concepts, see [Warehouse management overview](warehouse-management-overview.md).
- **Inventory owner tracking**: For details on how inventory ownership is tracked in the system, see [Inventory tracking dimensions](inventory-tracking-dimensions.md).

---

## Frequently asked questions

**Q: Does the "Prevent multiple owners" constraint apply during cycle counting?**  
A: No. Cycle counting is a physical inventory verification process and does not enforce put-away constraints. However, if cycle counting reveals that a location contains inventory with multiple owners (from inventory that was placed before the constraint was enabled), you should relocate one owner's stock to another location to comply with the constraint going forward.

**Q: What happens if an owner code is empty or null?**  
A: Empty owner values are treated as a distinct owner and mix only with other inventory that also has an empty owner. This prevents accidentally commingling unowned inventory with owner-specific inventory.

**Q: Can I have both "Prevent multiple owners" and "Allow mixed items" enabled on the same location profile?**  
A: Yes. Both constraints work independently. The location will allow multiple items but only from the same owner.

**Q: Does this feature support multi-owner inventory transfers?**  
A: The constraint prevents storing multi-owner inventory in the same location. To transfer inventory between owners, you must use a separate transfer order and ensure inventory passes through a quarantine or staging location configured to allow multiple owners (i.e., with the constraint disabled).

**Q: If I disable "Prevent multiple owners" after it has been enabled, what happens to existing inventory?**  
A: Disabling the constraint does not automatically move inventory. Existing multi-owner inventory (if any) will remain in place. Disabling simply stops enforcing the constraint for future put-away operations.

---

## See also

- [Warehouse management overview](warehouse-management-overview.md)
- [Location profiles overview](location-profiles-overview.md)
- [Create location profiles](create-location-profile.md)
- [Location directives overview](location-directives-overview.md)
