# POS Product Search — 29/06/2026

## What Was Done

Implemented the product search feature for the POS billing page. The top search bar in the POS page was a dead UI element (no state, no API call). It now fetches products from the database and shows a dropdown with matching results. Also added an "Add New Product" modal that creates products directly from the POS page without navigating away.

## File Modified

- `frontend/app/Pos/page.tsx` — single file changed

## No Backend Changes

The `GET /api/products/search?query=...` endpoint already existed in the backend (`productController.js:455-520`). It searches `name`, `barcode`, `modelNo`, `hsnSac` with case-insensitive `contains`. The `POST /api/products` endpoint also already existed (`productController.js:42-131`). No backend work was needed.

## Changes Made

### 1. `Product` interface (line 68-93)
Added TypeScript interface matching the backend `searchProducts` response format.

### 2. State variables (line 1179-1183)
- `productSearchQuery` — current search input value
- `productResults` — array of Product[] from API
- `showProductDropdown` — controls dropdown visibility
- `isSearchingProduct` — loading state for spinner
- `productSelected` — flag to prevent re-search after selection
- `productDropdownRef` — ref for click-outside detection

### 3. Debounced search useEffect (after customer search useEffect)
- 300ms debounce to avoid hammering API
- Calls `GET ${NEXT_PUBLIC_API_URL}/api/products/search?query=${productSearchQuery}`
- Auth via `localStorage.getItem("sessionToken")`
- Returns `data.data || []`

### 4. Click-outside useEffect
- Closes dropdown when clicking outside `productDropdownRef`

### 5. `addProductToBill(product: Product)` function (line 1494)
Maps Product to POS Item. **First checks for duplicates** — if an item with the same `code` already exists in the bill, shows error toast `"{name} already exists in the bill"` and returns without adding.

Mapping:
- `code` ← `barcode || _id`
- `name` ← `productName`
- `qty` ← `1`
- `unit` ← `primaryUnit || "PCS"`
- `price` ← `sellingPrice`
- `discount` ← `0`
- `tax` ← calculated from `sellingPrice * gstPercentage / 100` if `gstType === "Include"`, else `0`

Then calls `updateItems`, `setSelRow`, clears search, closes dropdown, shows toast.

### 6. Wired search input (line ~1845)
Added `value`, `onChange`, `onFocus` handlers to the previously dead `<input>`.

### 7. Dropdown JSX (after search bar)
- Loading spinner with `RefreshCw` icon
- Product result buttons showing name, barcode/model/HSN, price
- "No products found" empty state
- "+ Add New Product" button — opens `AddProductModal` (previously navigated to `/products`)

### 8. `AddProductModal` component (line 1163-1290)
Modal for creating products directly from POS. Follows `AddCustomerModal` pattern. Fields:
- Product Name (required)
- Selling Price (required, number)
- Barcode
- HSN/SAC Code
- Model Number
- Unit (default "PCS") + GST% (side by side)
- GST Type (dropdown: Include/Exclude, default "Include")
- Description (textarea)

POSTs to `POST ${NEXT_PUBLIC_API_URL}/api/products` with backend's expected field names (`productName`, `sellingPrice`, `barcode`, `hsnSac`, `modelNo`, `primaryUnit`, `gstPercentage`, `gstType`, `description`).

On success: maps response to `Product` interface, calls `addProductToBill()` to immediately add the new product to the current bill, closes modal.

### 9. `ModalType` union (line 93)
Added `"add_product"` to the union type.

### 10. `AddProductModalProps` type (line 118)
Props interface: `onClose`, `onSuccess(product: Product)`, `notify`.

### 11. Modal rendering (line 1918)
```tsx
{modal === "add_product" && (
  <AddProductModal
    notify={notify}
    onClose={() => setModal(null)}
    onSuccess={(newProduct) => addProductToBill(newProduct)}
  />
)}
```

## Data Flow

### Search Flow
```
User types → 300ms debounce → GET /api/products/search?query=...
→ dropdown appears → user clicks product → addProductToBill()
→ Product mapped to Item → added to bill → search clears
```

### Add Product Flow
```
User clicks "+ Add New Product" → dropdown closes → modal opens
→ fills form (name, price, barcode, etc.) → clicks "Add Product"
→ POST /api/products → product created in DB
→ response mapped to Product → addProductToBill() → item added to bill
→ modal closes → toast notification
```

## Testing Checklist

### Search
- [ ] Type product name → dropdown shows matches after ~300ms
- [ ] Type barcode → matching product appears
- [ ] Type model number → matching product appears
- [ ] Type HSN/SAC code → matching product appears
- [ ] Click product → item added to bill with correct fields
- [ ] After selection → search clears, dropdown closes
- [ ] Click outside dropdown → closes
- [ ] No results → "No products found" shown
- [ ] Loading spinner shows while fetching
- [ ] Tax calculated correctly for Include vs Exclude GST
- [ ] Add same product twice → error toast "already exists in the bill", not added

### Add Product Modal
- [ ] Click "+ Add New Product" → modal opens, dropdown closes
- [ ] Leave product name empty → validation error
- [ ] Leave selling price empty or 0 → validation error
- [ ] Fill required fields + submit → product created in DB
- [ ] Modal closes after success
- [ ] New product immediately added to bill
- [ ] Toast notification shows success/error
- [ ] Cancel button closes modal without creating
- [ ] Optional fields (barcode, HSN, model, GST%, description) save correctly
