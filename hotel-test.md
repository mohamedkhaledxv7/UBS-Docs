# Hotels Module Testing Plan

## Prerequisites
- Authenticated user with hotel management permissions
- Access to backend at `https://api.stgtenant.hub-travel.getpayin.com`
- At least one location, one connector, and one amenity configured in Settings

---

## Suggested Test Order

```
Room Inventories (Create → List → Edit → Delete)
    ↓
Hotels (Create → List → Search/Filter → Details → Edit → Delete)
    ↓
Rooms (Create with hotel + inventory → List → Search/Filter → Details → Edit → Delete)
    ↓
Rates & Availability (via Room Details → Single Edit → Bulk Edit → Standalone Screen)
```

This order respects dependencies (rooms need hotels; calendar needs rooms) and mirrors real user flows.

---

## 1. Room Inventories (test first — rooms depend on them)

### 1.1 Create Room Inventory
- [ ] Navigate to Properties > Room Inventories > Create
- [ ] Submit empty form → verify validation error on **Name** and **Quantity**
- [ ] Fill name + quantity → submit → verify success and appears in list
- [ ] Create with size (m²) → verify it saves correctly
- [ ] Create with amenities → open amenities picker, select 2–3, verify count shown, submit
- [ ] "Create & create another" → verify form resets and stays on create screen

### 1.2 List & Search
- [ ] Verify room inventories appear in list with correct name/quantity/amenities
- [ ] Search by name → verify results filter correctly (with 600ms debounce)
- [ ] Clear search → verify all results restore
- [ ] Scroll to trigger infinite pagination (if > 15 records)

### 1.3 Edit Room Inventory
- [ ] Tap a card → verify form pre-populated with existing data
- [ ] Change name and save → verify list updates
- [ ] Add/remove amenities and save → verify changes persist

### 1.4 Delete Room Inventory
- [ ] Delete a room inventory → verify confirmation prompt
- [ ] Confirm delete → verify removed from list
- [ ] Cancel → verify room inventory NOT deleted
- [ ] Try to delete one that's linked to a room (if supported) → verify error handling

---

## 2. Hotels

### 2.1 Create Hotel

**Required fields validation:**
- [ ] Submit empty form → verify errors on **Name** and **Location (City)**

**Basic creation:**
- [ ] Fill name + city → submit → verify hotel appears in list
- [ ] Fill all fields: name, content, permalink, order, rating (tap 3 stars), min advance days, min stay nights, status=Active, source

**Policies & FAQs:**
- [ ] Add a policy (title + content) → verify it appears
- [ ] Add a second policy → verify both display
- [ ] Remove a policy → verify it disappears
- [ ] Add a FAQ (question + answer) → verify it saves

**Location:**
- [ ] Tap location picker → search for a city → select → verify city populates
- [ ] Add address, latitude, longitude

**Media:**
- [ ] Upload a cover image → verify preview shows
- [ ] Upload gallery images → verify multiple images appear
- [ ] Delete a gallery image before submitting → verify it's removed

**SEO:**
- [ ] Fill SEO title, description, toggle indexable → verify they save

**Submit variants:**
- [ ] "Create" → verify redirect to hotel list
- [ ] "Create & create another" → verify form resets and stays on create screen

### 2.2 List, Search & Filter
- [ ] Verify hotels appear in list with name, status badge, source
- [ ] Search by name → verify debounced filter works
- [ ] Open filter modal → filter by status=Draft → verify only drafts show
- [ ] Filter by location → verify results scoped to that city
- [ ] Sort by `created_at` desc → verify order changes
- [ ] Apply multiple filters → verify filter badge count updates
- [ ] Clear filters → verify all hotels return
- [ ] Scroll to paginate (if > 15 hotels)

### 2.3 Hotel Details (Read-only View)
- [ ] Tap a hotel card → verify all fields display correctly
- [ ] Cover image shows (or placeholder if none)
- [ ] Policies and FAQs sections render
- [ ] Gallery renders all uploaded images
- [ ] SEO section shows correct data

### 2.4 Edit Hotel
- [ ] Tap "Edit Hotel" → verify form pre-populated with all existing data
- [ ] Change name, save → verify list and detail view update
- [ ] Change status Draft → Active → verify badge updates
- [ ] Add new policy → save → verify added
- [ ] Remove existing policy → save → verify removed
- [ ] Upload new cover image → save → verify new image shows
- [ ] Delete gallery image → save → verify image removed from detail view

### 2.5 Delete Hotel
- [ ] Tap "Delete Hotel" → verify confirmation dialog
- [ ] Confirm → verify hotel removed from list
- [ ] Cancel → verify hotel NOT deleted

---

## 3. Rooms (requires at least one hotel)

### 3.1 Create Room

**Required fields validation:**
- [ ] Submit empty form → verify errors on **Name**, **Hotel**, **Display Price**

**Basic creation:**
- [ ] Open hotel picker → search → select hotel → verify hotel name shown
- [ ] Fill name, display price → submit → verify room appears in list

**Room Setup:**
- [ ] Select room inventory via picker (optional) → verify it links
- [ ] Set beds count, max adults, max children
- [ ] Select meal plan (e.g. Half Board) → verify selection saved

**Connectors:**
- [ ] Open connector picker → select one or more → verify count badge shown
- [ ] Deselect a connector → verify count decreases

**Media:**
- [ ] Upload cover image, gallery images → verify previews

**SEO:**
- [ ] Fill SEO fields → verify they save

**Submit variants:**
- [ ] "Create" → verify redirect to rooms list
- [ ] "Create & create another" → verify form resets

### 3.2 List, Search & Filter
- [ ] Verify rooms list shows name, hotel, status, price
- [ ] Search by room name → verify filter
- [ ] Filter by connector → verify only linked rooms show
- [ ] Filter by room inventory → verify results
- [ ] Filter by bed count, max adults, status
- [ ] Clear filters → all rooms return

### 3.3 Room Details
- [ ] Tap room → verify all fields display correctly
- [ ] **Sales Channels section** appears if connectors linked, showing name and visibility
- [ ] Cover image, gallery, description, SEO all render
- [ ] "Rates & Availability" button is visible

### 3.4 Edit Room
- [ ] Open edit → verify form pre-populated
- [ ] Change hotel → verify hotel picker updates correctly
- [ ] Change meal plan → save → verify change persists
- [ ] Add/remove connectors → save → verify Sales Channels section updates in detail view

### 3.5 Delete Room
- [ ] Delete a room → confirmation prompt → confirm → verify removed from list
- [ ] Cancel → verify room NOT deleted

---

## 4. Rates & Availability

### 4.1 Access via Room Details (Calendar View)
- [ ] From a room's detail screen → tap "Rates & Availability"
- [ ] Verify calendar loads showing current month
- [ ] Verify each day cell shows: date, website price, available rooms, capacity, availability status

### 4.2 Calendar Navigation
- [ ] Tap ">" to go to next month → verify month/year updates
- [ ] Tap "<" to go to previous month → verify
- [ ] Tap "Today" → verify returns to current month

### 4.3 Single Day Edit
- [ ] Tap a day cell → verify `AvailabilityEditModal` opens with current values
- [ ] Change website price → save → verify day cell updates with new price
- [ ] Change available rooms count → save → verify updates
- [ ] Toggle `is_available` off → save → verify visual change on day cell
- [ ] Edit per-connector visibility/price/availability → save → verify changes

### 4.4 Bulk Update
- [ ] Tap "Bulk Update" in header → verify `BulkEditModal` opens
- [ ] Select a start date and end date spanning 5+ days
- [ ] Set a price and availability → apply
- [ ] Verify all days in the range show the updated values on the calendar

### 4.5 Standalone Rates Screen (via Properties Tab)
- [ ] Navigate to Properties tab → Rates & Availability (standalone)
- [ ] Verify "No Room Selected" placeholder shown initially
- [ ] Select a hotel → verify room picker activates and filters to that hotel's rooms
- [ ] Select a room → verify calendar loads
- [ ] Switch hotel → verify room picker resets and calendar clears
- [ ] Perform single-day edit → verify works same as room-detail flow

---

## 5. Cross-Cutting Concerns

### 5.1 Error Handling
- [ ] Kill network → attempt any list load → verify error state shown (not crash)
- [ ] Submit create form with network off → verify error message displayed
- [ ] Attempt to load a deleted resource by navigating back → verify graceful error

### 5.2 Loading States
- [ ] Verify skeleton/spinner shown while fetching hotel list
- [ ] Verify spinner shown while loading edit form data
- [ ] Verify calendar shows loading state while month data fetches

### 5.3 Navigation Flow
- [ ] Create hotel → cancel → verify back navigation works correctly
- [ ] From rooms list → create room → "Create & create another" multiple times → back → verify correct screen
- [ ] Deep flow: Hotels List → Hotel Details → Edit → Save → verify returns to Detail with updated data

### 5.4 Data Consistency
- [ ] Create a room linked to Hotel A → list rooms → filter by Hotel A → verify room appears
- [ ] Edit room to link a room inventory → open room detail → verify room inventory name shows
- [ ] Update rates for a room → navigate away → come back → verify rates persist
