# Cruises Module Testing Plan

## Prerequisites
- Authenticated user with cruise management permissions
- Access to backend at `https://api.stgtenant.hub-travel.getpayin.com`
- At least one connector and one amenity configured in Settings

---

## Suggested Test Order

```
Cabin Inventories (Create → List → Edit → Delete)
    ↓
Cruises (Create with Programs → List → Search/Filter → Edit → Delete)
    ↓
Cabins (Create with cruise + inventory → List → Search/Filter → Edit → Delete)
    ↓
Rates & Availability (via Cabin → Single Edit → Bulk Edit → Standalone Screen)
```

This order respects dependencies (cabins need cruises + inventories; calendar needs cabins with programs) and mirrors real user flows.

---

## 1. Cabin Inventories (test first — cabins depend on them)

### 1.1 Create Cabin Inventory
- [ ] Navigate to Properties > Cabin Inventories > Create
- [ ] Submit empty form → verify validation error on **Name** and **Quantity**
- [ ] Fill name + quantity → submit → verify success and appears in list
- [ ] Create with size (m²) → verify it saves correctly
- [ ] Create with amenities → open amenities picker, select 2–3, verify count shown, submit
- [ ] "Create & create another" → verify form resets and stays on create screen

### 1.2 List & Search
- [ ] Verify cabin inventories appear in list with correct name/quantity/amenities
- [ ] Search by name → verify results filter correctly (with 600ms debounce)
- [ ] Clear search → verify all results restore
- [ ] Scroll to trigger infinite pagination (if > 15 records)

### 1.3 Edit Cabin Inventory
- [ ] Tap a card → verify form pre-populated with existing data
- [ ] Change name and save → verify list updates
- [ ] Add/remove amenities and save → verify changes persist

### 1.4 Delete Cabin Inventory
- [ ] Delete a cabin inventory → verify confirmation prompt
- [ ] Confirm delete → verify removed from list
- [ ] Cancel → verify cabin inventory NOT deleted
- [ ] Try to delete one that's linked to a cabin (if supported) → verify error handling

---

## 2. Cruises

### 2.1 Create Cruise

**Required fields validation:**
- [ ] Submit empty form → verify error on **Name**

**Basic creation:**
- [ ] Fill name → submit → verify cruise appears in list
- [ ] Fill all fields: name, content (rich text), permalink, order, status=Active

**Programs:**
- [ ] Add a program → fill name, content, departure location, arrival location, route, departure day, number of nights, duration, order
- [ ] Add a second program → verify both display in collapsible sections
- [ ] Remove a program → verify it disappears
- [ ] Reorder programs → verify order persists after save

**Media:**
- [ ] Upload a cover image → verify preview shows
- [ ] Upload gallery images → verify multiple images appear
- [ ] Delete a gallery image before submitting → verify it's removed

**SEO:**
- [ ] Fill SEO title, description, toggle indexable → verify they save

**Submit variants:**
- [ ] "Create" → verify redirect to cruise list
- [ ] "Create & create another" → verify form resets and stays on create screen

### 2.2 List, Search & Filter
- [ ] Verify cruises appear in list with name, status badge
- [ ] Search by name → verify debounced filter works (600ms)
- [ ] Open filter modal → filter by status=Draft → verify only drafts show
- [ ] Filter by created date range (from/until) → verify results scoped
- [ ] Sort by `name` → verify alphabetical order
- [ ] Sort by `order` → verify custom ordering
- [ ] Sort by `created_at` desc → verify chronological order
- [ ] Apply multiple filters → verify filter badge count updates
- [ ] Clear filters → verify all cruises return
- [ ] Scroll to paginate (if > 15 cruises)

### 2.3 Edit Cruise
- [ ] Tap a cruise card → verify edit form pre-populated with all existing data
- [ ] Change name, save → verify list updates
- [ ] Change status Draft → Active → verify badge updates
- [ ] Add new program → save → verify added
- [ ] Edit existing program fields → save → verify changes persist
- [ ] Remove existing program → save → verify removed
- [ ] Upload new cover image → save → verify new image shows
- [ ] Delete gallery image → save → verify image removed

### 2.4 Delete Cruise
- [ ] Tap "Delete" on a cruise → verify confirmation dialog
- [ ] Confirm → verify cruise removed from list
- [ ] Cancel → verify cruise NOT deleted
- [ ] Try to delete a cruise that has cabins linked → verify error handling

---

## 3. Cabins (requires at least one cruise)

### 3.1 Create Cabin (Multi-Step)

**Step 1 — Details & Setup:**

**Required fields validation:**
- [ ] Submit empty form → verify errors on **Name**, **Cruise**, **Display Price**

**Basic creation:**
- [ ] Open cruise picker → search → select cruise → verify cruise name shown
- [ ] Fill name, display price → proceed to step 2

**Cabin Setup:**
- [ ] Select cabin inventory via picker (optional) → verify it links
- [ ] Set number of beds
- [ ] Set max adults, max children
- [ ] Select meal plan (Room Only / Bed & Breakfast / Half Board / Full Board / All Inclusive) → verify selection shown

**Programs:**
- [ ] Select which cruise programs this cabin belongs to → verify selections
- [ ] Deselect a program → verify it's removed

**Connectors (Sales Channels):**
- [ ] Open connector picker → select one or more → verify count badge shown
- [ ] Set visibility and price override per connector
- [ ] Deselect a connector → verify count decreases

**Media:**
- [ ] Upload cover image → verify preview
- [ ] Upload gallery images → verify multiple images appear

**SEO:**
- [ ] Fill SEO fields → verify they save

**Step 2 — Rates Info:**
- [ ] Verify informational rates step displays correctly
- [ ] Verify step indicator shows step 2 of 2

**Submit variants:**
- [ ] "Create" → verify redirect to cabins list
- [ ] "Create & create another" → verify form resets to step 1

### 3.2 List, Search & Filter
- [ ] Verify cabins list shows name, cruise, status, price
- [ ] Search by cabin name → verify debounced filter (600ms)
- [ ] Filter by cruise → verify only cabins for that cruise show
- [ ] Filter by connector → verify only linked cabins show
- [ ] Filter by cabin inventory → verify results
- [ ] Filter by status (Published / Draft / Pending)
- [ ] Sort by order, status, created_at, display_price → verify each
- [ ] Apply multiple filters → verify filter badge count updates
- [ ] Clear filters → all cabins return
- [ ] Scroll to paginate (if > 15 cabins)

### 3.3 Edit Cabin
- [ ] Tap a cabin card → verify form pre-populated with all existing data
- [ ] Change cruise → verify cruise picker updates correctly
- [ ] Change meal plan → save → verify change persists
- [ ] Change display price, sale price → save → verify prices update
- [ ] Add/remove connectors → save → verify sales channels update
- [ ] Add/remove programs → save → verify program assignments update
- [ ] Upload new cover image → save → verify new image shows
- [ ] Delete gallery image → save → verify image removed

### 3.4 Delete Cabin
- [ ] Delete a cabin → confirmation prompt → confirm → verify removed from list
- [ ] Cancel → verify cabin NOT deleted

---

## 4. Rates & Availability

### 4.1 Access via Cabin (Calendar View)
- [ ] From cabins list → tap cabin → navigate to "Rates & Availability"
- [ ] Verify calendar loads showing current month
- [ ] Verify each day cell shows: date, website price, available cabins, capacity, availability status
- [ ] Verify departure days are visually highlighted

### 4.2 Program Switching
- [ ] If cabin has multiple programs → verify program selector/switcher is available
- [ ] Switch between programs → verify calendar data updates for each program
- [ ] Verify departure day changes per program

### 4.3 Calendar Navigation
- [ ] Tap ">" to go to next month → verify month/year updates
- [ ] Tap "<" to go to previous month → verify
- [ ] Tap "Today" → verify returns to current month
- [ ] Verify past days are visually distinct

### 4.4 Single Day Edit
- [ ] Tap a day cell → verify edit modal opens with current values
- [ ] Verify prices are rounded up in the modal
- [ ] Change website price → save → verify day cell updates with new price
- [ ] Change available cabins count → save → verify updates
- [ ] Toggle `is_available` off → save → verify visual change on day cell
- [ ] Edit per-connector visibility/price/availability → save → verify changes
- [ ] Verify auto-calculated connector pricing (margin) displays correctly

### 4.5 Bulk Update
- [ ] Tap "Bulk Update" header button → verify bulk edit modal opens
- [ ] Select a start date and end date spanning 5+ days
- [ ] Set a price and availability → apply
- [ ] Verify all days in the range show the updated values on the calendar
- [ ] Verify prices are rounded up in the bulk edit modal

### 4.6 Standalone Rates Screen (via Properties Tab)
- [ ] Navigate to Properties tab → Cruise Rates & Availability (standalone)
- [ ] Verify placeholder shown when no cabin is selected
- [ ] Select a cruise → verify cabin picker activates and filters to that cruise's cabins
- [ ] Select a cabin → verify calendar loads
- [ ] If cabin has programs → verify program selector appears
- [ ] Switch cruise → verify cabin picker resets and calendar clears
- [ ] Perform single-day edit → verify works same as cabin-detail flow
- [ ] Perform bulk edit → verify works correctly
- [ ] Verify scroll bottom padding is present for comfortable scrolling

---

## 5. Cross-Cutting Concerns

### 5.1 Error Handling
- [ ] Kill network → attempt any list load → verify error state shown (not crash)
- [ ] Submit create form with network off → verify error message displayed
- [ ] Attempt to load a deleted resource by navigating back → verify graceful error

### 5.2 Loading States
- [ ] Verify skeleton/spinner shown while fetching cruise list
- [ ] Verify spinner shown while loading edit form data
- [ ] Verify calendar shows loading state while month data fetches
- [ ] Verify loading indicator during media uploads

### 5.3 Navigation Flow
- [ ] Create cruise → cancel → verify back navigation works correctly
- [ ] From cabins list → create cabin → "Create & create another" multiple times → back → verify correct screen
- [ ] Deep flow: Cabins List → Edit Cabin → Save → verify returns with updated data
- [ ] Cabin → Rates & Availability → back → verify correct screen
- [ ] Standalone Rates → switch cruise/cabin multiple times → verify no stale data

### 5.4 Data Consistency
- [ ] Create a cabin linked to Cruise A → list cabins → filter by Cruise A → verify cabin appears
- [ ] Edit cabin to link a cabin inventory → verify cabin inventory name shows
- [ ] Update rates for a cabin → navigate away → come back → verify rates persist
- [ ] Delete a program from a cruise → verify cabins linked to that program handle it gracefully
- [ ] Create cabin with connectors → verify connector visibility/price override saved correctly

### 5.5 Rich Text / Content Fields
- [ ] Enter rich text content in cruise description → save → verify formatting preserved
- [ ] Enter rich text in program content → save → verify formatting preserved
- [ ] Enter rich text in cabin description → save → verify formatting preserved
