# Tierfy: Customer Tag-Based Pricing Setup Guide

Display discounted wholesale/VIP prices directly on collection pages and automatically hide the tier table on product pages for single-tier flat discounts.

---

## 📋 Overview of Changes
1. **Create 1 Snippet:** `snippets/tierfy-collection-pricing.liquid` (handles the logic)
2. **Add 1 Line to Theme:** In `layout/theme.liquid` (`{% render 'tierfy-collection-pricing' %}`)
3. **Wrap Price in Collection:** In `snippets/price.liquid` (adds the product data wrapper)

---

## 📍 Step 1: Create the Snippet File

1. In your Shopify Admin, go to **Online Store → Themes**.
2. Click the **three dots (`...`)** next to your active theme and select **Edit code**.
3. Under the **Snippets** folder on the left, click **Add a new snippet**.
4. Name the snippet: `tierfy-collection-pricing` (it will create `tierfy-collection-pricing.liquid`).
5. Paste the following code into the file and click **Save**:

```liquid
<!-- Tierfy Collection Pricing & PDP Control (Preserving 'From' & Variant Support) -->
<script>
(function() {
    const currentCustomerTags = "{% if customer %}{{ customer.tags | join: ',' | downcase | escape }}{% endif %}";
    const currencySymbol = "{{ cart.currency.symbol }}";
    let cachedRules = null;
    let cachedProducts = {};

    // 1. Product Detail Page (PDP): Hide table if it only has 1 tier for Min Qty = 1
    function checkAndHidePdpTable() {
        const table = document.getElementById('tiered-pricing-table-wrapper');
        if (!table) return;

        const rows = table.querySelectorAll('.tp-row, .tpm-row, .tpn-row, .tpp-row, .tpw-row, .tps-card, tbody tr');
        if (rows.length === 1) {
            const text = rows[0].textContent || '';
            const isMinQty1 = /\b1\s*(\+|unit|piece|item|ea|qty)/i.test(text) || text.includes('1+');
            if (isMinQty1) {
                table.style.setProperty('display', 'none', 'important');
                const skeleton = document.getElementById('tiered-pricing-skeleton');
                if (skeleton) skeleton.style.setProperty('display', 'none', 'important');
                return;
            }
        }
        if (rows.length > 1 && table.style.display === 'none') {
            table.style.removeProperty('display');
        }
    }

    function calculateSavings(price, tier) {
        const discountVal = parseFloat(tier.discountValue || 0);
        if (tier.discountType === 'PERCENTAGE') {
            const priceInCents = Math.round(price * 100);
            const discountInCents = Math.floor((priceInCents * discountVal) / 100);
            return discountInCents / 100;
        }
        return discountVal;
    }

    function getMatchingRules(rules, pData) {
        return rules.filter(rule => {
            let rVals = [];
            try {
                const parsed = JSON.parse(rule.targetValues);
                rVals = Array.isArray(parsed) ? parsed : [String(parsed)];
            } catch(e) {
                rVals = String(rule.targetValues).split(',');
            }
            rVals = rVals.map(v => String(v).toLowerCase().trim());

            if (rule.targetType === 'PRODUCT') {
                return rVals.some(v => {
                    const shortV = v.split('/').pop();
                    return v === pData.id || shortV === pData.id || pData.id.includes(shortV);
                });
            }
            if (rule.targetType === 'COLLECTION') {
                return rVals.some(v => {
                    const shortV = v.split('/').pop();
                    return pData.collectionIds.includes(v) || pData.collectionIds.includes(shortV);
                });
            }
            if (rule.targetType === 'TAG') {
                return rVals.some(v => pData.tags.includes(v));
            }
            if (rule.targetType === 'SKU') {
                return rVals.some(v => pData.skus.includes(v));
            }
            return false;
        });
    }

    function getBestTiers(matchingRules, basePrice) {
        const tierMap = new Map();
        matchingRules.forEach(r => {
            if (!r.tiers || !Array.isArray(r.tiers)) return;
            r.tiers.forEach(t => {
                const minQty = parseInt(t.minQuantity, 10);
                if (isNaN(minQty)) return;

                const savings = calculateSavings(basePrice, t);
                const tierObj = { ...t, minQuantity: minQty, _savings: savings };
                const existing = tierMap.get(minQty);
                if (!existing || tierObj._savings > existing._savings) {
                    tierMap.set(minQty, tierObj);
                }
            });
        });
        return Array.from(tierMap.values()).sort((a, b) => a.minQuantity - b.minQuantity);
    }

    // 2. Render Single Price Element
    function renderPriceElement(el, rules) {
        const pData = {
            id: String(el.dataset.productId || ''),
            handle: el.dataset.productHandle || '',
            price: parseFloat(el.dataset.productPrice || 0),
            priceMin: parseFloat(el.dataset.priceMin || el.dataset.productPrice || 0),
            priceMax: parseFloat(el.dataset.priceMax || el.dataset.productPrice || 0),
            priceVaries: el.dataset.priceVaries === 'true',
            tags: (el.dataset.tags || '').split(',').map(s => s.trim().toLowerCase()).filter(Boolean),
            collectionIds: (el.dataset.collectionIds || '').split(',').map(s => s.trim()).filter(Boolean),
            skus: (el.dataset.skus || '').split(',').map(s => s.trim().toLowerCase()).filter(Boolean)
        };

        const isPdpElement = Boolean(el.closest('.product, .product__info-wrapper, .product__info-container, .product-single, .product-form, main [class*="product"]'));
        const matchingRules = getMatchingRules(rules, pData);
        if (matchingRules.length === 0) return;

        const basePriceToUse = (!isPdpElement && pData.priceVaries) ? pData.priceMin : pData.price;
        const uniqueTiers = getBestTiers(matchingRules, basePriceToUse);
        if (uniqueTiers.length === 0) return;

        const tierQty1 = uniqueTiers.find(t => t.minQuantity === 1);
        const maxSavingsTier = uniqueTiers.reduce((prev, curr) => (prev._savings > curr._savings) ? prev : curr);

        // Detect and preserve theme's "From" element / text if present
        const ogPriceEl = el.querySelector('.original-theme-price');
        let fromPrefixHtml = '';

        if (!isPdpElement) {
            if (ogPriceEl) {
                const fromEl = ogPriceEl.querySelector('.price__from, [class*="from"], [class*="price-from"]');
                if (fromEl && fromEl.textContent.trim()) {
                    fromPrefixHtml = fromEl.outerHTML + ' ';
                } else if (pData.priceVaries) {
                    const ogText = ogPriceEl.textContent || '';
                    const matchFrom = ogText.match(/\b(from|ab|à partir de|desde)\b/i);
                    if (matchFrom) {
                        fromPrefixHtml = `<span class="price__from">${matchFrom[0]}</span> `;
                    } else {
                        fromPrefixHtml = `<span class="price__from">From</span> `;
                    }
                }
            } else if (pData.priceVaries) {
                fromPrefixHtml = `<span class="price__from">From</span> `;
            }
        }

        let newHtml = '';
        const originalPriceStr = basePriceToUse.toFixed(2);
        const originalPriceHtml = `<span style="text-decoration: line-through; opacity: 0.6; margin-right: 6px;">${currencySymbol}${originalPriceStr}</span>`;

        if (isPdpElement) {
            // PDP: Always show exact active variant price (no "From")
            if (tierQty1) {
                const newPrice = (pData.price - tierQty1._savings).toFixed(2);
                newHtml = `${originalPriceHtml} <span style="color: #e32c2b; font-weight: bold;">${currencySymbol}${newPrice}</span>`;
            }
        } else if (uniqueTiers.length === 1 && tierQty1) {
            // Case 1: Single flat discount (e.g. 50% off) -> Keeps "From" if present
            const newPrice = (basePriceToUse - tierQty1._savings).toFixed(2);
            newHtml = `${fromPrefixHtml}${originalPriceHtml}<span style="color: #e32c2b; font-weight: bold;">${currencySymbol}${newPrice}</span>`;
        } else if (uniqueTiers.length > 1) {
            // Case 2: Volume tier card
            const minPrice = (basePriceToUse - maxSavingsTier._savings).toFixed(2);
            newHtml = `${fromPrefixHtml}<span style="color: #e32c2b; font-weight: bold;">As low as ${currencySymbol}${minPrice}</span>`;
        } else if (!tierQty1) {
            // Case 3: Volume tier starting at qty > 1
            const newPrice = (basePriceToUse - maxSavingsTier._savings).toFixed(2);
            newHtml = `${fromPrefixHtml}<span style="font-weight: bold; color: #e32c2b;">Buy ${maxSavingsTier.minQuantity}+ for ${currencySymbol}${newPrice}/ea</span>`;
        }

        if (newHtml) {
            if (ogPriceEl) ogPriceEl.style.display = 'none';

            let smartPriceEl = el.querySelector('.tierfy-smart-price');
            if (smartPriceEl) {
                smartPriceEl.innerHTML = newHtml;
            } else {
                el.insertAdjacentHTML('beforeend', `<div class="tierfy-smart-price">${newHtml}</div>`);
            }
        }
    }

    // 3. Collection & PDP Discount Display
    function loadTieredPrices() {
        const priceElements = Array.from(document.querySelectorAll('.tierfy-collection-price:not(.tierfy-processed)'));
        if (priceElements.length === 0) return;
        priceElements.forEach(el => el.classList.add('tierfy-processed'));

        if (cachedRules) {
            priceElements.forEach(el => renderPriceElement(el, cachedRules));
            return;
        }

        let allProductIds = [], allTags = [], allCollectionIds = [], allSkus = [];
        document.querySelectorAll('.tierfy-collection-price').forEach(el => {
            if (el.dataset.productId) allProductIds.push(el.dataset.productId);
            if (el.dataset.tags) allTags = allTags.concat(el.dataset.tags.split(',').map(s => s.trim().toLowerCase()));
            if (el.dataset.collectionIds) allCollectionIds = allCollectionIds.concat(el.dataset.collectionIds.split(',').map(s => s.trim()));
            if (el.dataset.skus) allSkus = allSkus.concat(el.dataset.skus.split(',').map(s => s.trim().toLowerCase()));
        });

        const uniqueProductIds = [...new Set(allProductIds)].filter(Boolean).join(',');
        const uniqueTags = [...new Set(allTags)].filter(Boolean).join(',');
        const uniqueCollectionIds = [...new Set(allCollectionIds)].filter(Boolean).join(',');
        const uniqueSkus = [...new Set(allSkus)].filter(Boolean).join(',');

        const url = `/apps/tiered-pricing?product_id=${encodeURIComponent(uniqueProductIds)}` +
                    `&tags=${encodeURIComponent(uniqueTags)}` +
                    `&collection_ids=${encodeURIComponent(uniqueCollectionIds)}` +
                    `&skus=${encodeURIComponent(uniqueSkus)}` +
                    `&customer_tags=${encodeURIComponent(currentCustomerTags)}`;

        fetch(url)
          .then(res => res.json())
          .then(data => {
              if (!data.rules || data.rules.length === 0) return;
              cachedRules = data.rules;
              document.querySelectorAll('.tierfy-collection-price').forEach(el => {
                  renderPriceElement(el, cachedRules);
              });
          })
          .catch(err => console.error("Tierfy Error:", err));
    }

    // 4. Dynamic Variant Change Handler on PDP
    function handleVariantChange() {
        const pdpPriceElements = document.querySelectorAll('.product .tierfy-collection-price, .product__info-wrapper .tierfy-collection-price, .product-form .tierfy-collection-price, main [class*="product"] .tierfy-collection-price');
        if (pdpPriceElements.length === 0) return;

        const urlParams = new URLSearchParams(window.location.search);
        let variantId = urlParams.get('variant');

        if (!variantId) {
            const variantInput = document.querySelector('input[name="id"], select[name="id"]');
            if (variantInput) variantId = variantInput.value;
        }
        if (!variantId) return;

        pdpPriceElements.forEach(el => {
            const handle = el.dataset.productHandle;
            if (!handle) return;

            function applyVariantPrice(productData) {
                const variant = (productData.variants || []).find(v => String(v.id) === String(variantId));
                if (variant) {
                    el.dataset.productPrice = (variant.price / 100.0).toString();
                    if (variant.sku) el.dataset.skus = variant.sku.toLowerCase();
                    if (cachedRules) {
                        renderPriceElement(el, cachedRules);
                    }
                }
            }

            if (cachedProducts[handle]) {
                applyVariantPrice(cachedProducts[handle]);
            } else {
                const root = (window.Shopify && window.Shopify.routes && window.Shopify.routes.root) || '/';
                fetch(`${root}products/${handle}.js`)
                    .then(r => r.json())
                    .then(data => {
                        cachedProducts[handle] = data;
                        applyVariantPrice(data);
                    })
                    .catch(() => {});
            }
        });
    }

    // Setup Variant Listeners (URL History, Radio/Select Changes, Custom Events)
    function setupVariantListeners() {
        const origPushState = history.pushState;
        const origReplaceState = history.replaceState;
        history.pushState = function() {
            origPushState.apply(this, arguments);
            setTimeout(handleVariantChange, 20);
        };
        history.replaceState = function() {
            origReplaceState.apply(this, arguments);
            setTimeout(handleVariantChange, 20);
        };
        window.addEventListener('popstate', handleVariantChange);
        document.addEventListener('change', function(e) {
            if (e.target.matches('input[name="id"], select[name="id"], [name*="options["], variant-radios input, variant-selects select')) {
                setTimeout(handleVariantChange, 50);
            }
        });
        document.addEventListener('variant:change', handleVariantChange);
        document.addEventListener('theme:variant:change', handleVariantChange);
    }

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', function() {
            checkAndHidePdpTable();
            loadTieredPrices();
            setupVariantListeners();
        });
    } else {
        checkAndHidePdpTable();
        loadTieredPrices();
        setupVariantListeners();
    }

    const observer = new MutationObserver((mutations) => {
        checkAndHidePdpTable();

        let shouldLoad = false;
        mutations.forEach((mutation) => {
            if (mutation.addedNodes.length) {
                mutation.addedNodes.forEach(node => {
                    if (node.nodeType === 1) {
                        if (node.classList && node.classList.contains('tierfy-collection-price')) {
                            shouldLoad = true;
                        } else if (node.querySelector && node.querySelector('.tierfy-collection-price')) {
                            shouldLoad = true;
                        }
                    }
                });
            }
        });
        if (shouldLoad) {
            clearTimeout(window.__tpCollectionTimeout);
            window.__tpCollectionTimeout = setTimeout(loadTieredPrices, 50);
        }
    });

    observer.observe(document.body, { childList: true, subtree: true });
})();
</script>
```

---

## 📍 Step 2: Render in `theme.liquid`

1. In the search bar on the left, search for `theme.liquid` under the **Layout** folder.
2. Scroll to the very bottom and locate the closing `</body>` tag.
3. Paste this **single line** right before `</body>` and click **Save**:

```liquid
{% render 'tierfy-collection-pricing' %}
```

---

## 📍 Step 3: Wrap the Price in `price.liquid`

1. In the search bar on the left, search for `price.liquid` (or `card-product.liquid`) under **Snippets**.
2. Find the main price container and **wrap it** with `<div class="tierfy-collection-price">`:

```liquid
{%- liquid
  assign tierfy_product = product_resource | default: product | default: card_product
  assign tierfy_target = target | default: tierfy_product.selected_or_first_available_variant | default: tierfy_product
-%}
<div class="tierfy-collection-price"
     data-product-id="{{ tierfy_product.id }}"
     data-product-handle="{{ tierfy_product.handle }}"
     data-product-price="{{ tierfy_target.price | default: tierfy_product.price | divided_by: 100.0 }}"
     data-price-min="{{ tierfy_product.price_min | divided_by: 100.0 }}"
     data-price-max="{{ tierfy_product.price_max | divided_by: 100.0 }}"
     data-price-varies="{{ tierfy_product.price_varies }}"
     data-selected-variant-id="{{ tierfy_target.id }}"
     data-tags="{{ tierfy_product.tags | join: ',' | downcase | escape }}"
     data-collection-ids="{{ tierfy_product.collections | map: 'id' | join: ',' }}"
     data-skus="{{ tierfy_product.variants | map: 'sku' | join: ',' | downcase | escape }}">
  
  <div class="original-theme-price">
    <!-- YOUR EXISTING THEME PRICE LIQUID CODE HERE -->
  </div>
</div>
```
3. Click **Save**.

---

## 📍 Step 4: Test & Verify

1. Log into your Shopify storefront with a customer account tagged with your customer rule (e.g., `VIP` or `Wholesale`).
2. Open any **Collection page** to verify that discounted prices appear directly in the grid.
3. Open a product with a 1-tier flat discount on the **Product Page** to verify that the redundant table is hidden.

🎉 **All done!**
